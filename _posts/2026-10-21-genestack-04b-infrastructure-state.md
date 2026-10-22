---
title: "OpenStack trên Kubernetes với Genestack – Phần 4b: Longhorn, MariaDB Galera, RabbitMQ và memcached"
date: 2026-10-21 08:05:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, longhorn, mariadb, galera, rabbitmq, memcached]
render_with_liquid: false
---

Ở [Phần 4a](/posts/genestack-04a-infrastructure-network/), chúng ta đã có đường vào cho API: MetalLB, cert-manager và Envoy Gateway. Bài này hoàn thiện lớp hạ tầng nền với những thành phần **giữ state** cho OpenStack:

| Thành phần | Vai trò | Version trong lab |
|---|---|---|
| Longhorn | Persistent volume cho mọi pod có dữ liệu | 1.11.1 |
| MariaDB Galera (mariadb-operator) | Database của mọi service OpenStack | operator 26.3.0, MariaDB 11.8 |
| RabbitMQ (cluster operator) | Message bus cho RPC và notification | RabbitMQ 4.1 |
| memcached | Cache token và dữ liệu cho các API | chart 2025.2 |

Thứ tự cài có ràng buộc: MariaDB và RabbitMQ lưu dữ liệu trên Longhorn, nên **Longhorn phải chạy trước**. Ở giữa có thêm một bước tạo secret.

## Longhorn

### Cách Longhorn lưu dữ liệu

![Longhorn: pod gắn volume qua engine, engine ghi đồng bộ tới replica trên 2 node; volume RWX dùng share-manager làm NFS server](/assets/img/genestack/04b-longhorn.svg)

Longhorn là block storage phân tán chạy ngay trên các node Kubernetes:

- Mỗi **volume** (tương ứng một PVC) có một **engine** chạy trên node của pod đang dùng nó. Engine cung cấp block device cho pod.
- Engine ghi đồng bộ dữ liệu xuống nhiều **replica**, mỗi replica nằm trên một node khác nhau, trong thư mục data của Longhorn.
- Khi một node chết, volume vẫn hoạt động nhờ replica còn lại, và Longhorn tự dựng lại replica bị mất ở một node khác.
- Với volume **RWX** (nhiều pod cùng gắn), Longhorn chạy thêm một pod **share-manager** làm NFS server, và các pod mount volume qua NFS.

Longhorn chỉ chạy trên các node có label `longhorn.io/storage-node=enabled`, đã gắn ở Phần 3. Trong lab đó là 3 controller.

**Yêu cầu trên các node đó:** Longhorn dùng iSCSI để gắn volume vào node, nên cần gói `open-iscsi` (dịch vụ `iscsid` chạy). Node dùng volume RWX cần thêm NFS client (`nfs-common`).

### Thêm ổ đĩa riêng cho Longhorn

Mặc định Longhorn lưu dữ liệu ở `/var/lib/longhorn/` trên ổ root. Nên dành cho Longhorn một ổ riêng, để dữ liệu của volume không tranh chỗ với hệ điều hành và container image. Lab gắn thêm ổ `vdd` (50 GB) cho mỗi controller (Phần 2). Trên **mỗi** storage node:

```bash
mkfs.ext4 -L longhorn-disk2 /dev/vdd
mkdir -p /var/lib/longhorn-disk2
echo "UUID=$(blkid -s UUID -o value /dev/vdd) /var/lib/longhorn-disk2 ext4 defaults,nofail 0 2" >> /etc/fstab
mount /var/lib/longhorn-disk2
```

Tuỳ chọn `nofail` giúp node vẫn khởi động được nếu ổ có vấn đề. Sau khi cài Longhorn ở bước dưới, khai báo ổ này cho Longhorn, ví dụ với controller-1:

```bash
kubectl -n longhorn-system patch nodes.longhorn.io openstack-controller-1 --type merge -p '
{"spec":{"disks":{"disk2":{
  "path":"/var/lib/longhorn-disk2",
  "allowScheduling":true,
  "storageReserved":5368709120
}}}}'
```

`storageReserved` là dung lượng Longhorn không được dùng trên ổ đó (ở đây 5 GiB). Ổ mặc định trên root được Longhorn tự giữ lại khoảng 30%, để hệ điều hành không bị đầy đĩa. Có thể thao tác tương tự trên giao diện web của Longhorn.

### Cài Longhorn

```bash
kubectl apply -f /etc/genestack/manifests/longhorn/longhorn-namespace.yaml
/opt/genestack/bin/install-longhorn.sh
```

Base config của Genestack đã giới hạn Longhorn vào các storage node:

```yaml
global:
  nodeSelector:
    longhorn.io/storage-node: "enabled"
  defaultSettings:
    systemManagedComponentsNodeSelector: "longhorn.io/storage-node:enabled"
```

Hai giá trị này có vai trò khác nhau:

- **`global.nodeSelector`** áp cho các thành phần do Helm tạo, như `longhorn-manager` và driver.
- **`systemManagedComponentsNodeSelector`** áp cho các pod mà Longhorn **tự tạo** trong lúc chạy: instance-manager, share-manager, engine image…

Nếu thiếu giá trị thứ hai, share-manager có thể bị schedule lên một node không chạy Longhorn, và volume RWX sẽ không gắn được.

`defaultSettings` chỉ được Longhorn đọc khi setting chưa có giá trị. Sau khi cài, hãy kiểm tra lại:

```bash
kubectl -n longhorn-system get settings.longhorn.io system-managed-components-node-selector -o jsonpath='{.value}{"\n"}'
# longhorn.io/storage-node:enabled
```

Nếu kết quả rỗng, đặt trực tiếp:

```bash
kubectl -n longhorn-system patch settings.longhorn.io system-managed-components-node-selector \
  --type merge -p '{"value":"longhorn.io/storage-node:enabled"}'
```

### StorageClass

```bash
kubectl apply -f /etc/genestack/manifests/longhorn/longhorn-general-storageclass.yaml
kubectl apply -f /etc/genestack/manifests/longhorn/longhorn-general-multi-attach-storageclass.yaml
```

| StorageClass | Kiểu truy cập | Dùng cho |
|---|---|---|
| `general` (mặc định) | RWO | MariaDB, RabbitMQ, và phần lớn PVC khác |
| `general-multi-attach` | RWX | Kho image của Glance, nơi nhiều pod cùng đọc/ghi |

Các tham số chính của `general`:

```yaml
parameters:
  numberOfReplicas: "2"
  dataLocality: "best-effort"
  staleReplicaTimeout: "2880"
  fsType: "ext4"
```

- **`numberOfReplicas`**: số bản sao. 2 là đủ cho lab 3 node. Production thường dùng 3. Script `setup-infrastructure.sh` cho phép đổi giá trị này qua biến `LONGHORN_STORAGE_REPLICAS`. Dung lượng thực tế bằng kích thước PVC nhân với số replica.
- **`dataLocality: best-effort`**: Longhorn cố gắng đặt một replica cùng node với pod, để thao tác đọc không phải đi qua mạng.

### Kiểm tra

```bash
kubectl -n longhorn-system get pods -o wide        # chỉ chạy trên các storage node
kubectl -n longhorn-system get nodes.longhorn.io   # READY=True, SCHEDULABLE=True
kubectl get sc                                     # general (default), general-multi-attach
```

## Secret cho OpenStack

Trước khi dựng database, cần tạo mật khẩu cho mọi thành phần: tài khoản root và tài khoản ứng dụng của MariaDB, tài khoản RabbitMQ, database và admin của từng service OpenStack… Genestack sinh tất cả bằng một script:

```bash
/opt/genestack/bin/create-secrets.sh
kubectl apply -f /etc/genestack/kubesecrets.yaml
```

Script sinh mật khẩu ngẫu nhiên (32–64 ký tự) và ghi toàn bộ vào `/etc/genestack/kubesecrets.yaml`. Trong lab, file này chứa 84 Secret. Một vài lưu ý:

- **Chỉ chạy `create-secrets.sh` một lần.** Chạy lại sẽ sinh mật khẩu mới, không khớp với dữ liệu đã có trong database. Script cài tự động của Genestack cũng tự bỏ qua bước này nếu file đã tồn tại.
- **File chứa mọi mật khẩu của cloud.** Hãy giữ quyền truy cập hạn chế (lab đặt `640`, chủ sở hữu root), sao lưu vào nơi an toàn, và không commit vào git.

## MariaDB Galera

### Mô hình

![MariaDB Galera 3 node, Service primary trỏ vào một node duy nhất; RabbitMQ 3 node; memcached 3 node](/assets/img/genestack/04b-state-services.svg)

Genestack dùng **mariadb-operator** để dựng MariaDB **Galera** 3 node. Galera là cluster đồng bộ multi-master: về lý thuyết ghi được ở mọi node. Tuy vậy, operator tạo ra nhiều Service với vai trò khác nhau:

| Service | Endpoint | Dùng cho |
|---|---|---|
| `mariadb-cluster-primary` | **Một node**, node primary hiện tại | Mọi service OpenStack kết nối vào đây |
| `mariadb-cluster-secondary` | Các node còn lại | Đọc, nếu ứng dụng hỗ trợ tách đọc/ghi |
| `mariadb-cluster` | Mọi node | Truy cập chung |
| `mariadb-cluster-internal` | Headless | Đồng bộ Galera giữa các node (port 4567, 4568, 4444) |

Hướng mọi thao tác ghi vào một node giúp tránh **xung đột certification**: khi hai node cùng sửa một dòng dữ liệu, Galera phải huỷ một giao dịch. Đổi lại, mỗi khi primary chuyển sang node khác (rolling update, node chết), các service OpenStack sẽ lỗi database trong vài giây trước khi kết nối lại.

### Cài operator và cluster

```bash
/opt/genestack/bin/install-mariadb-operator.sh
kubectl -n mariadb-system wait --timeout=5m deploy/mariadb-operator-webhook --for=condition=available
kubectl -n openstack apply -k /etc/genestack/kustomize/mariadb-cluster/overlay
```

Script cài hai chart: CRD (`mariadb-operator-crds`) và operator. Nếu webhook của operator không sẵn sàng sau một phút, script tự động của Genestack sẽ xoá các pod trong `mariadb-system` để chúng khởi động lại. Bạn có thể làm tương tự nếu gặp tình huống này.

Overlay `mariadb-cluster/overlay` tham chiếu tới cấu hình `galera` của Genestack (đã giới thiệu ở Phần 1). Cluster trong lab có:

- 3 replica, **anti-affinity bắt buộc**: mỗi node một pod, chỉ chạy trên node control-plane/worker;
- mỗi pod một PVC 10 GiB trên StorageClass `general`;
- `innodb_buffer_pool_size=4G`. Đây là lý do chính khiến controller cần nhiều RAM;
- Service `primary` kiểu `LoadBalancer` với annotation pool `primary` của MetalLB, để truy cập từ ngoài cluster khi cần quản trị.

Cluster cần vài phút để bootstrap Galera:

```bash
kubectl -n openstack get mariadb mariadb-cluster -w
# NAME              READY   STATUS    PRIMARY             UPDATES
# mariadb-cluster   True    Running   mariadb-cluster-0   RollingUpdate
```

### Backup

Genestack kèm một `Backup` CR chạy `mariadb-dump` hằng ngày vào một PVC riêng. Kiểm tra affinity của job backup: nếu nó yêu cầu anti-affinity **bắt buộc** với các pod MariaDB, trong khi chỉ được chạy trên 3 controller (đều đã có pod MariaDB), job sẽ không bao giờ được schedule. Chỉ nên dùng anti-affinity dạng `preferred` cho job backup.

```bash
kubectl -n openstack get backup
kubectl -n openstack get backup mariadb-backup -o jsonpath='{.spec.affinity}'
```

### Kiểm tra

```bash
kubectl -n openstack exec mariadb-cluster-0 -c mariadb -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" -e "SHOW GLOBAL STATUS WHERE Variable_name IN (\"wsrep_cluster_size\",\"wsrep_cluster_status\",\"wsrep_local_state_comment\")"'
```

```
wsrep_cluster_size          3
wsrep_cluster_status        Primary
wsrep_local_state_comment   Synced
```

Kiểm tra trên cả 3 pod: đều phải là `3 / Primary / Synced`.

## RabbitMQ

RabbitMQ được quản lý bởi hai operator:

- **cluster-operator** dựng cluster RabbitMQ từ CRD `RabbitmqCluster`;
- **messaging-topology-operator** tạo vhost, user và quyền cho từng service OpenStack, cũng qua CRD.

```bash
kubectl apply -k /etc/genestack/kustomize/rabbitmq-operator/base
kubectl apply -k /etc/genestack/kustomize/rabbitmq-topology-operator/base
kubectl -n rabbitmq-system wait --timeout=5m deploy/rabbitmq-cluster-operator --for=condition=available
kubectl -n rabbitmq-system wait --timeout=5m deploy/messaging-topology-operator --for=condition=available

kubectl apply -k /etc/genestack/kustomize/rabbitmq-cluster/overlay
```

Topology operator dùng webhook. Nếu lệnh apply cuối báo lỗi webhook, hãy chờ Service `messaging-topology-webhook-service` có endpoint rồi chạy lại.

Cluster trong lab có 3 replica (anti-affinity bắt buộc, chỉ chạy trên node `worker`), mỗi replica một PVC 20 GiB, image `rabbitmq:4.1.4-management`. Các service OpenStack kết nối qua `rabbitmq.openstack.svc.cluster.local:5672`. Mỗi service có vhost và user riêng.

Kiểm tra:

```bash
kubectl -n openstack get rabbitmqcluster
# NAME       ALLREPLICASREADY   RECONCILESUCCESS
# rabbitmq   True               True
kubectl -n openstack exec rabbitmq-server-0 -- rabbitmqctl cluster_status | grep -A4 "Running Nodes"
```

## memcached

```bash
/opt/genestack/bin/install-memcached.sh
```

memcached chạy 3 replica (anti-affinity bắt buộc), không cần PVC. Các API của OpenStack, đặc biệt là Keystone, dùng nó để cache token và dữ liệu. Mất cache chỉ làm hệ thống chậm đi một lúc chứ không mất dữ liệu, nên memcached là thành phần dễ vận hành nhất trong lớp này.

```bash
kubectl -n openstack get sts memcached-memcached     # READY 3/3
```

## Checklist

| Thành phần | Lệnh | Kết quả mong đợi |
|---|---|---|
| Longhorn | `kubectl -n longhorn-system get nodes.longhorn.io` | Mọi storage node `READY`, `SCHEDULABLE` |
| Longhorn | `kubectl get sc` | `general` (default) và `general-multi-attach` |
| Secret | `kubectl -n openstack get secret mariadb keystone-admin` | Tồn tại |
| MariaDB | `kubectl -n openstack get mariadb` | `READY=True`, `STATUS=Running` |
| Galera | `SHOW GLOBAL STATUS` trên từng pod | `3 / Primary / Synced` |
| RabbitMQ | `kubectl -n openstack get rabbitmqcluster` | `ALLREPLICASREADY=True` |
| memcached | `kubectl -n openstack get sts memcached-memcached` | `3/3` |
| PVC | `kubectl -n openstack get pvc` | Mọi PVC `Bound` |

## Tóm tắt

- Longhorn chạy trước, chỉ trên storage node. Dành cho nó ổ riêng, và kiểm tra `system-managed-components-node-selector` sau khi cài.
- StorageClass `general` (RWO, 2 replica) cho database và message bus. `general-multi-attach` (RWX) cho Glance.
- `create-secrets.sh` chỉ chạy một lần. File `kubesecrets.yaml` là tài sản cần bảo vệ.
- MariaDB Galera 3 node do operator quản lý. Mọi service ghi vào `mariadb-cluster-primary`.
- RabbitMQ do hai operator quản lý. memcached cài bằng Helm.
- Cả ba dịch vụ dùng anti-affinity bắt buộc: mỗi controller một replica.

## Tiếp theo

Lớp hạ tầng nền đã hoàn chỉnh. Ở **Phần 5**, chúng ta bắt đầu cài các service OpenStack: Keystone, Glance, Placement, Nova, Neutron với OVN, và tạo máy ảo đầu tiên.
