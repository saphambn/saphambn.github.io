---
title: "OpenStack trên Kubernetes với Genestack – Phần 5: Triển khai các service OpenStack core"
date: 2026-10-21 08:06:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, keystone, nova, neutron, ovn, glance]
render_with_liquid: false
---

Sau [Phần 4a](/posts/genestack-04a-infrastructure-network/) và [Phần 4b](/posts/genestack-04b-infrastructure-state/), cluster đã có đủ hạ tầng nền: đường vào cho API, chứng chỉ, storage, database, message bus và cache. Đến lúc cài OpenStack.

Bài này cài năm service tạo nên một cloud tối thiểu: **Keystone, Glance, Placement, Nova** (kèm libvirt) và **Neutron** với OVN. Cuối bài, chúng ta sẽ tạo network, upload image và boot máy ảo đầu tiên. Cinder (block storage) và Skyline (dashboard) sẽ có bài riêng ở Phần 6 và Phần 7.

| Service | Chart trong lab | Vai trò |
|---|---|---|
| Keystone | 2025.1.5 | Xác thực, cấp token, danh mục endpoint |
| Glance | 2025.1.4 | Kho image |
| Placement | 2025.1.2 | Theo dõi tài nguyên của các compute node |
| Nova | 2025.1.19 | API máy ảo, lập lịch, điều phối |
| libvirt | 2025.1.4 | Hypervisor trên compute node |
| Neutron | 2025.1.17 | Mạng ảo, dùng ML2 với OVN |

## Bức tranh tổng thể

![Các service OpenStack core: API phía sau Envoy Gateway, chạy trên controller; nova-compute, libvirt, metadata agent và ovn-controller trên node; dùng chung MariaDB, RabbitMQ, memcached, Longhorn, OVN central](/assets/img/genestack/05-core-services.svg)

- Các **API** chạy dưới dạng Deployment trên controller, được đưa ra ngoài qua Envoy Gateway (Phần 4a).
- Trên **compute node**, `nova-compute` và `libvirt` chạy dạng DaemonSet. VM chính là tiến trình QEMU bên trong pod libvirt.
- **Neutron** không có agent L2/L3 truyền thống. Neutron server ghi cấu hình mạng vào **OVN Northbound DB**. `ovn-northd` dịch sang Southbound DB, và `ovn-controller` trên từng node lập trình OVS cho cổng của VM. OVN ở đây chính là OVN mà kube-ovn đã dựng ở Phần 3.
- Mọi service dùng chung MariaDB, RabbitMQ và memcached (Phần 4b). Glance lưu image trên volume RWX của Longhorn.

## Cách một service được cài

Mọi service OpenStack trong Genestack được cài theo cùng một khuôn mẫu:

```bash
/opt/genestack/bin/install-<service>.sh
```

Script này:

1. đọc version chart từ `/etc/genestack/helm-chart-versions.yaml`;
2. nạp lần lượt các file values: `base-helm-configs/<service>/`, rồi `helm-configs/global_overrides/`, rồi `helm-configs/<service>/`. File nạp sau thắng;
3. lấy mật khẩu từ các Secret đã tạo ở Phần 4b và truyền vào chart;
4. chạy `helm upgrade --install` với post-renderer Kustomize.

Mỗi chart OpenStack-Helm còn tạo thêm một loạt **Job** khởi tạo: tạo database (`*-db-init`), migrate schema (`*-db-sync`), tạo user/service/endpoint trong Keystone (`*-ks-user`, `*-ks-service`, `*-ks-endpoints`)… Pod của service chỉ khởi động khi các job này xong, nhờ init container `kubernetes-entrypoint`. Vì vậy trong vài phút đầu, việc thấy pod ở trạng thái `Init:0/1` là bình thường.

### Endpoint public trong catalog

Keystone giữ **danh mục endpoint** (catalog) để client biết gọi service nào ở đâu. Mặc định, endpoint public mà chart OpenStack-Helm đăng ký là tên DNS nội bộ của cluster, kiểu `keystone-api.openstack.svc.cluster.local`. Client ở ngoài cluster không dùng được địa chỉ này.

Để catalog trỏ tới hostname trên Envoy Gateway, khai báo `host_fqdn_override` cho từng service. Lab gom tất cả vào một file dùng chung, được mọi script `install-*.sh` nạp:

```yaml
# /etc/genestack/helm-configs/global_overrides/endpoints.yaml (trích)
endpoints:
  identity:
    host_fqdn_override:
      public:
        tls: {}
        host: keystone.api.openstack.example.com
    port:
      api:
        public: 443
    scheme:
      public: https
  compute:
    host_fqdn_override:
      public:
        tls: {}
        host: nova.api.openstack.example.com
    port:
      api:
        public: 443
    scheme:
      public: https
  # … tương tự cho image, placement, network, compute_metadata, compute_novnc_proxy…
```

Có hai điều cần nhớ:

- **Hostname ở đây phải trùng với hostname của HTTPRoute** mà `setup-envoy-gateway.sh` đã tạo ở Phần 4a.
- **Tạo file này trước khi cài service.** Nếu thêm sau, phải chạy lại `install-<service>.sh` để job `*-ks-endpoints` cập nhật catalog.

Tài liệu Genestack mô tả chi tiết cơ chế này trong `docs/openstack-override-public-endpoint-fqdn.md`.

## Bước 1 – Keystone

Keystone phải có trước, vì mọi service khác cần đăng ký user và endpoint với nó:

```bash
/opt/genestack/bin/install-keystone.sh
kubectl -n openstack get pods -l application=keystone
```

Khi `keystone-api` đã `Running`, Secret `keystone-keystone-admin` chứa thông tin đăng nhập của admin: URL, user, mật khẩu, project, domain, region. Mọi công cụ quản trị đều đọc từ Secret này.

### Công cụ dòng lệnh

Cách nhanh nhất để có `openstack` CLI là chạy pod admin client, đã cấu hình sẵn từ Secret trên:

```bash
kubectl -n openstack apply -f /etc/genestack/manifests/utils/utils-openstack-client-admin.yaml
kubectl -n openstack exec -it openstack-admin-client -- openstack token issue
```

Manifest không khai báo `namespace`, nên bắt buộc phải có `-n openstack`. Thiếu nó, pod sẽ rơi vào namespace `default` và không đọc được Secret `keystone-keystone-admin` trong `openstack`.

Pod này là một `Pod` đơn lẻ, không do Deployment quản lý. Nếu node chứa nó bị tắt, pod sẽ không được tạo lại ở node khác, và bạn cần apply lại manifest trên.

Tiện hơn nữa, tạo một hàm shell trên deployer:

```bash
os() { kubectl -n openstack exec -i openstack-admin-client -- openstack "$@"; }
os endpoint list --interface public -c "Service Name" -c URL
```

Nếu deployer phân giải được hostname của gateway (qua DNS hoặc `/etc/hosts`, Phần 4a), có thể dùng CLI cài trực tiếp trên deployer. Script `setup-openstack-rc.sh` sinh file `~/.config/openstack/genestack-clouds.yaml` từ Secret admin:

```bash
/opt/genestack/bin/setup-openstack-rc.sh
openstack --os-cloud default token issue
```

## Bước 2 – Glance và Placement

Hai service này độc lập với nhau và có thể cài song song:

```bash
/opt/genestack/bin/install-glance.sh
/opt/genestack/bin/install-placement.sh
```

Glance trong Genestack mặc định dùng backend `storage: pvc`: image được lưu dạng file trên PVC `glance-images` (Longhorn RWX, 10 GiB trong lab). Mọi pod `glance-api` gắn chung PVC này. Với lab, như vậy là đủ. Ở production, cân nhắc Ceph RBD, Swift hoặc object storage, và tăng kích thước PVC theo số lượng image dự kiến.

## Bước 3 – Tích hợp OVN cho Neutron

Trước khi cài Neutron, các node cần biết bridge, interface và mạng vật lý nào dành cho Neutron. Genestack khai báo những thông tin này bằng **annotation trên node**, rồi một DaemonSet đọc annotation và cấu hình OVS.

![Từ annotation trên node, qua DaemonSet ovn-setup, tới cấu hình OVS/OVN và mạng Neutron](/assets/img/genestack/05-ovn-network.svg)

### Annotation

| Annotation | Giá trị trong lab | Ý nghĩa |
|---|---|---|
| `ovn.openstack.org/int_bridge` | `br-int` | Bridge tích hợp của OVN |
| `ovn.openstack.org/bridges` | `br-ex` | Các bridge OVS cần tạo |
| `ovn.openstack.org/ports` | `br-ex:enp11s0` | Gắn interface vật lý nào vào bridge nào |
| `ovn.openstack.org/mappings` | `physnet1:br-ex` | Tên mạng vật lý mà Neutron dùng, ánh xạ tới bridge |
| `ovn.openstack.org/availability_zones` | `az1` | Availability zone của mạng |
| `ovn.openstack.org/gateway` | `enabled` | Node có làm gateway chassis hay không |

Ngoài ra còn các annotation tuỳ chọn cho bond (`bonds`, `bond-options`) và VLAN (`vlans`), dành cho môi trường production có nhiều card mạng.

Lab dùng cùng một cấu hình cho cả 5 node, vì node nào cũng có `enp11s0` nối vào mạng provider:

```bash
SEL="-l openstack-network-node=enabled"
kubectl annotate nodes $SEL ovn.openstack.org/int_bridge='br-int'
kubectl annotate nodes $SEL ovn.openstack.org/bridges='br-ex'
kubectl annotate nodes $SEL ovn.openstack.org/ports='br-ex:enp11s0'
kubectl annotate nodes $SEL ovn.openstack.org/mappings='physnet1:br-ex'
kubectl annotate nodes $SEL ovn.openstack.org/availability_zones='az1'
kubectl annotate nodes $SEL ovn.openstack.org/gateway='enabled'
```

`physnet1` chỉ là một cái tên. Neutron dùng nó khi tạo mạng provider, còn OVN tra mapping để biết trên mỗi node, `physnet1` nằm ở bridge nào. Nhờ vậy, các node có thể dùng interface khác nhau mà Neutron không cần biết.

### Chọn gateway

Annotation `gateway` quyết định node nào được giữ **cổng ra mạng ngoài của router Neutron** (nơi xử lý SNAT). Annotation này được chuyển thành `ovn-cms-options=enable-chassis-as-gw` trên chassis OVN.

- **Lab bật gateway trên mọi node**, kể cả compute. Cổng ra của các router được phân tán trên nhiều node, và mất một node thì OVN tự chuyển cổng sang node khác.
- Một mô hình phổ biến khác là **dồn gateway về vài node riêng** (network node). Traffic ra ngoài dễ kiểm soát hơn, và compute không cần nối mạng provider.

Cần ít nhất một node gateway trong cluster.

### Áp dụng

```bash
kubectl apply -k /etc/genestack/kustomize/ovn/overlay
```

Lệnh này tạo DaemonSet `ovn-setup` trong namespace `openstack`. Nó chỉ chạy trên các network node **chưa** có label `ovn.openstack.org/configured`. Trên mỗi node, nó tạo bridge, gắn port, ghi `ovn-bridge-mappings` và `ovn-cms-options` vào OVS, rồi gắn label `configured` kèm thời điểm. Sau khi mọi node đã được cấu hình, DaemonSet có 0 pod: đó là trạng thái bình thường.

Muốn đổi cấu hình một node, sửa annotation rồi gỡ label để DaemonSet chạy lại trên node đó:

```bash
kubectl label node <node> ovn.openstack.org/configured-
```

Script lưu trạng thái cũ trong ConfigMap `ovn-state-<node>`, nên có thể gỡ port cũ trước khi gắn port mới, tránh tạo vòng lặp mạng.

## Bước 4 – Neutron

Override của lab cho Neutron:

```yaml
# /etc/genestack/helm-configs/neutron/neutron-helm-overrides.yaml
conf:
  neutron:
    DEFAULT:
      global_physnet_mtu: 1500
  plugins:
    ml2_conf:
      ml2:
        path_mtu: 1500
        physical_network_mtus: physnet1:1500
manifests:
  deployment_rpc_server: true
pod:
  replicas:
    rpc_server: 2
```

- **Các tham số MTU** cho Neutron biết MTU thật của hạ tầng. Mạng provider `physnet1` có MTU 1500. Mạng tenant kiểu Geneve được Neutron tự trừ đi phần header tunnel: 1500 − 58 = **1442**. Nếu hạ tầng hỗ trợ jumbo frame, tăng các giá trị này để mạng tenant có MTU lớn hơn.
- **`deployment_rpc_server`** tách phần xử lý RPC ra một Deployment riêng (`neutron-rpc-server`), để `neutron-server` chỉ phục vụ API.

```bash
/opt/genestack/bin/install-neutron.sh
```

Sau khi cài, mỗi network node có hai agent: **OVN Controller Gateway agent** (hoặc OVN Controller agent nếu node không làm gateway) và **OVN Metadata agent**. Metadata agent cung cấp dịch vụ metadata `169.254.169.254` cho VM, nơi cloud-init lấy SSH key và user-data.

## Bước 5 – libvirt và Nova

### libvirt

```bash
/opt/genestack/bin/install-libvirt.sh
```

libvirt chạy dạng DaemonSet trên các node có label `openstack-compute-node=enabled`. Mỗi VM là một tiến trình QEMU bên trong pod này.

### Nova

Lab chạy compute node trên máy ảo, không có nested virtualization (Phần 2), nên phải ép Nova dùng giả lập phần mềm:

```yaml
# /etc/genestack/helm-configs/nova/nova-helm-overrides.yaml
conf:
  nova:
    libvirt:
      virt_type: qemu
      cpu_mode: none
```

Trên máy vật lý có KVM, **bỏ hai dòng này** để Nova dùng `virt_type: kvm` mặc định. Hiệu năng của VM sẽ cao hơn rất nhiều.

```bash
/opt/genestack/bin/install-nova.sh
```

Nova gồm nhiều thành phần (hình ở đầu bài):

- `nova-api-osapi`: API chính;
- `nova-api-metadata`: dịch vụ metadata, nhận request chuyển tiếp từ OVN Metadata agent;
- `nova-scheduler`: chọn compute node cho VM, dựa trên thông tin từ Placement;
- `nova-conductor`: trung gian giữa compute và database;
- `nova-novncproxy`: console VNC qua trình duyệt;
- `nova-compute`: DaemonSet trên compute, điều khiển libvirt.

Khi `nova-compute` khởi động lần đầu, nó tự đăng ký với Placement. Job `nova-cell-setup` (chạy định kỳ) đưa compute node mới vào cell, để scheduler bắt đầu đặt VM lên đó.

## Bước 6 – Kiểm tra các service

```bash
os compute service list
os network agent list
os hypervisor list
os endpoint list --interface public -c "Service Name" -c URL
```

| Kiểm tra | Kết quả mong đợi |
|---|---|
| `compute service list` | `nova-scheduler`, `nova-conductor` và `nova-compute` (mỗi compute một dòng) đều `enabled` / `up` |
| `network agent list` | Mỗi network node có OVN Controller (Gateway) agent và OVN Metadata agent, `Alive = True` |
| `hypervisor list` | Mỗi compute node một hypervisor, `State = up` |
| Endpoint public | URL dạng `https://<service>.api.<domain>` |

## Bước 7 – Máy ảo đầu tiên

### Mạng

Mạng provider `public` nằm trên `physnet1`, cùng dải với mạng `br-ex` ở Phần 2. Chừa riêng một dải cho floating IP, không chồng lên địa chỉ đã dùng trên mạng vật lý:

```bash
os network create public --external --share \
  --provider-network-type flat --provider-physical-network physnet1
os subnet create public-subnet --network public \
  --subnet-range 192.168.10.0/24 --gateway 192.168.10.1 \
  --allocation-pool start=192.168.10.150,end=192.168.10.199 --no-dhcp
```

Mạng tenant và router:

```bash
os network create test-net
os subnet create test-subnet --network test-net --subnet-range 172.16.0.0/24
os router create test-router
os router set test-router --external-gateway public
os router add subnet test-router test-subnet
```

`test-net` được tạo với kiểu mặc định là **Geneve**, có MTU 1442. Router làm SNAT cho VM ra ngoài, và cổng ra của nó được OVN đặt trên một gateway chassis.

### Flavor, image, security group

```bash
os flavor create m1.tiny  --vcpus 1 --ram 512  --disk 1
os flavor create m1.small --vcpus 1 --ram 2048 --disk 20

curl -LO https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
kubectl -n openstack cp cirros-0.6.2-x86_64-disk.img openstack-admin-client:/tmp/cirros.img
os image create "Cirros 0.6.2 64-bit" --disk-format qcow2 --container-format bare \
  --file /tmp/cirros.img --property os_distro=cirros

os security group rule create default --protocol icmp
os security group rule create default --protocol tcp --dst-port 22
```

### Boot và kiểm tra

```bash
os server create test-vm-1 --flavor m1.tiny --image "Cirros 0.6.2 64-bit" --network test-net --wait
os server show test-vm-1 -c status -c addresses -c OS-EXT-SRV-ATTR:host
os console log show test-vm-1 | tail -20

# Floating IP để truy cập từ mạng provider
os floating ip create public
os server add floating ip test-vm-1 <floating-ip>
```

VM ở trạng thái `ACTIVE`, console log hiện màn hình đăng nhập của Cirros, và từ một máy trên mạng `192.168.10.0/24` ping được floating IP: cloud đã hoạt động.

Nếu VM kẹt ở `BUILD` hoặc chuyển sang `ERROR`, xem trường `fault` trong `os server show`, rồi tới log theo thứ tự:

1. `nova-scheduler`: có tìm được host phù hợp không;
2. `nova-conductor`;
3. `nova-compute` trên node được chọn.

Với lỗi mạng, xem log của `neutron-server` và metadata agent.

## Cài thêm các service khác

Script `bin/setup-openstack.sh` của Genestack hỏi lần lượt muốn cài những service nào (Heat, Barbican, Cinder, Designate, Octavia, Masakari, Magnum…), lưu lựa chọn vào `/etc/genestack/openstack-components.yaml`, rồi cài Keystone trước và các service còn lại song song. Mọi service đều theo cùng khuôn mẫu `install-<service>.sh` như trên, và cần có entry trong `endpoints.yaml` nếu muốn đưa ra ngoài qua gateway.

## Tóm tắt

- Mỗi service OpenStack là một Helm release, cài bằng `install-<service>.sh`. Values được nạp theo thứ tự: base → `global_overrides` → override riêng.
- Khai báo `host_fqdn_override` cho endpoint public **trước khi cài**, khớp với hostname của HTTPRoute.
- Thứ tự cài: Keystone → Glance, Placement → tích hợp OVN → Neutron → libvirt, Nova.
- Neutron dùng OVN của kube-ovn. Bridge, interface, physnet và gateway được khai báo bằng annotation `ovn.openstack.org/*` và áp dụng qua DaemonSet `ovn-setup`.
- Trên VM không có nested virtualization, đặt `virt_type: qemu`. Trên máy vật lý, giữ mặc định KVM.

## Tiếp theo

Cloud đã boot được VM, nhưng mọi disk đều là ephemeral, nằm trên compute node. Ở **Phần 6**, chúng ta thêm block storage với **Cinder**, dùng backend LVM qua iSCSI, theo mô hình hybrid chạy ngoài Kubernetes.
