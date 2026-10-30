---
title: "OpenStack trên Kubernetes với Genestack – Phần 14: Thay image của một service bằng image tự build"
date: 2026-10-30 10:00:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, magnum, cluster-api, container-image]
render_with_liquid: false
---

Mọi service OpenStack trong Genestack chạy bằng image của rackerlabs (`ghcr.io/rackerlabs/genestack-images/<service>`). Sẽ có lúc bạn cần chạy code của mình: một bản vá chưa được merge, một tính năng đang phát triển, một thư viện cần nâng sớm. Bài này trình bày cách thay image của một service bằng image tự build, an toàn và rollback được.

Ví dụ xuyên suốt bài là Magnum ở [Phần 11](/posts/genestack-12-magnum/): thay driver `magnum-capi-helm` 1.4.0 bằng nhánh `nodegroup-server-group` của repo [`saphambn/magnum-capi-helm`](https://github.com/saphambn/magnum-capi-helm/tree/nodegroup-server-group). Nhánh này cho phép mỗi nodegroup boot vào một **Nova server group**, để các node nằm rải trên nhiều hypervisor. Cách làm áp dụng được cho mọi service khác.

## Genestack chọn image ở đâu

Image là một Helm value, nằm trong `images.tags` của chart. Genestack đặt giá trị mặc định ở base override, và bạn ghi đè trong thư mục của mình:

| Nơi | Vai trò |
|---|---|
| `/opt/genestack/base-helm-configs/<svc>/<svc>-helm-overrides.yaml` | Image mặc định của rackerlabs. **Không sửa** file này: nó bị thay khi cập nhật Genestack |
| `/etc/genestack/helm-configs/<svc>/*.yaml` | Override của bạn. Script cài đặt đọc mọi file ở đây, đè lên base |
| `/opt/genestack/bin/install-<svc>.sh` | Chạy `helm upgrade --install` với base, rồi tới override của bạn |

Xem service dùng những key image nào:

```bash
grep -A20 '^images:' /opt/genestack/base-helm-configs/magnum/magnum-helm-overrides.yaml
```

```yaml
images:
  tags:
    bootstrap: "ghcr.io/rackerlabs/genestack-images/openstack-client:latest"
    db_init: "ghcr.io/rackerlabs/genestack-images/openstack-client:latest"
    dep_check: "ghcr.io/rackerlabs/genestack-images/kubernetes-entrypoint:latest"
    ks_endpoints: "ghcr.io/rackerlabs/genestack-images/openstack-client:latest"
    magnum_api: "ghcr.io/rackerlabs/genestack-images/magnum:2025.1-latest"
    magnum_conductor: "ghcr.io/rackerlabs/genestack-images/magnum:2025.1-latest"
    magnum_db_sync: "ghcr.io/rackerlabs/genestack-images/magnum:2025.1-latest"
```

Các key `bootstrap`, `db_init`, `ks_*`… là job phụ, dùng image `openstack-client`, không cần đổi. Ba key dùng image `magnum` là những key cần thay. Driver chỉ chạy trong `magnum-conductor`, nhưng hãy đổi **cả ba** cùng lúc: `magnum_db_sync` chạy migration database, và code của API, conductor, migration nên luôn cùng một bản.

Kiểm tra cluster đang chạy image nào, và driver nào nằm trong đó:

```bash
kubectl -n openstack get sts magnum-conductor \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n openstack exec magnum-conductor-0 -c magnum-conductor -- \
  sh -c 'pip show magnum-capi-helm | grep ^Version; pip freeze | grep -i capi'
```

```text
ghcr.io/rackerlabs/genestack-images/magnum:2025.1-latest
Version: 1.4.0
magnum-capi-helm @ git+https://opendev.org/openstack/magnum-capi-helm@a8b82c2480a286a10b60ecc1d94c6cd9f1d01823
```

Đây là mốc để so sánh sau khi thay image.

## Image tự build cần giữ những gì

Chart của openstack-helm không chạy `ENTRYPOINT` của image. Nó mount script khởi động của nó vào `/tmp` rồi chạy script đó (`/tmp/magnum-api.sh start`, `/tmp/magnum-conductor.sh`). Script gọi các lệnh trong virtualenv. Vì vậy image mới phải giữ nguyên "hợp đồng" của image gốc:

| Yếu tố | Image gốc | Hậu quả nếu khác |
|---|---|---|
| Virtualenv | `/var/lib/openstack`, có trong `PATH` | Script không tìm thấy `magnum-conductor`, `uwsgi` |
| User | UID/GID `42424` | Không đọc được config, không ghi được thư mục làm việc |
| `/etc/magnum` | symlink tới `/var/lib/openstack/etc/magnum`, thuộc user `magnum` | Lỗi quyền khi chart mount config |
| Công cụ đi kèm | `helm` ở `/usr/local/bin` | Driver CAPI không cài được chart cho cluster |
| Nhánh OpenStack | Magnum `stable/2025.1` (cloud đang chạy `release-2026.2.0.2`) | `db_sync` chạy migration của bản khác, có thể không quay lại được |

Điểm cuối cùng quan trọng nhất. Image mới phải cùng nhánh OpenStack với cloud. Đổi nhánh OpenStack là một lần **nâng cấp** ([Phần 13](/posts/genestack-14-upgrade/)), không phải thay image.

Cách đơn giản nhất để giữ đúng mọi thứ là **dựa trên chính image gốc** và chỉ thay phần bạn sửa:

```dockerfile
FROM ghcr.io/rackerlabs/genestack-images/magnum:2025.1-latest
USER 0
# Image không có git, pbr cần version đặt sẵn
ENV PBR_VERSION=1.4.1.dev23
RUN pip install --no-deps --force-reinstall \
      https://github.com/saphambn/magnum-capi-helm/archive/refs/heads/nodegroup-server-group.tar.gz
USER 42424:42424
```

```bash
docker build -t ghcr.io/saphambn/magnum:2025.1-sg-$(date +%Y%m%d%H%M) --push .
```

- `--no-deps` giữ nguyên mọi thư viện khác của image gốc. Nếu nhánh của bạn thêm thư viện mới, hãy cài thêm chúng một cách tường minh, với version nằm trong `upper-constraints` của nhánh OpenStack.
- `PBR_VERSION` cần vì file `.tar.gz` của GitHub không chứa `.git`, và `pbr` không tự tính được version.
- Nhớ trả lại `USER 42424:42424` ở cuối.
- Thẻ `2025.1-latest` của image nền thay đổi hằng tuần. Muốn build lặp lại được, hãy ghi lại digest của image nền (`docker buildx imagetools inspect`) và dùng `FROM ...@sha256:<digest>`.

Nếu cần image build đầy đủ như của rackerlabs (từ đầu, có quét CVE), có thể fork repo [`genestack-images`](https://github.com/rackerlabs/genestack-images) và build lại `ContainerFiles/magnum`. Image nhận được vẫn phải giữ đúng các yếu tố ở bảng trên.

## Thay image

Tạo một file override riêng cho image, ví dụ `/etc/genestack/helm-configs/magnum/magnum-image-overrides.yaml`. Để image ở file riêng giúp rollback chỉ bằng cách xoá một file:

```yaml
images:
  tags:
    magnum_api: "ghcr.io/saphambn/magnum:2025.1-sg-202610301000"
    magnum_conductor: "ghcr.io/saphambn/magnum:2025.1-sg-202610301000"
    magnum_db_sync: "ghcr.io/saphambn/magnum:2025.1-sg-202610301000"
```

**Mỗi lần build, dùng một tag mới**, không đè lên tag cũ. Chart đặt `pull_policy: IfNotPresent`: node đã có image cùng tag sẽ không kéo bản mới. Tag không đổi thì `helm upgrade` cũng không thấy thay đổi, và pod không được rollout lại.

Nếu tính năng mới có cấu hình, đặt luôn ở đây. Ví dụ policy server group mặc định cho mọi cluster:

```yaml
conf:
  magnum:
    capi_helm_cluster_labels:
      server_group_policy: soft-anti-affinity
```

Chạy lại script cài đặt:

```bash
/opt/genestack/bin/install-magnum.sh
kubectl -n openstack rollout status deploy/magnum-api
kubectl -n openstack rollout status sts/magnum-conductor
```

Kiểm tra lại như ở đầu bài:

```bash
kubectl -n openstack exec magnum-conductor-0 -c magnum-conductor -- \
  sh -c 'pip show magnum-capi-helm | grep ^Version'
os coe service list
```

Version phải là bản của bạn (`1.4.1.dev23`), và `magnum-conductor` phải `up`.

Cluster Kubernetes đang chạy không bị ảnh hưởng khi conductor được thay. Chúng là Helm release trên management cluster, và conductor mới tiếp tục quản lý chúng.

## Thử tính năng mới

Tạo một nodegroup để driver tự tạo server group theo policy:

```bash
os coe nodegroup create k8s-lab ng-aa --node-count 2 --flavor m1.large \
  --labels server_group_policy=soft-anti-affinity
os server group list
```

Driver tạo server group tên `magnum-<cluster-uuid>-ng-aa` trong project của cluster. Trên management cluster, `OpenStackMachineTemplate` của nodegroup phải có `serverGroup`:

```bash
kubectl -n magnum-<project-id> get openstackmachinetemplate -o yaml | grep -A2 serverGroup
```

Khi node đã lên, kiểm tra chúng nằm trên các hypervisor khác nhau:

```bash
os server group show magnum-<cluster-uuid>-ng-aa -c members
os server list --long -c Name -c Host
```

Cuối cùng xoá nodegroup (`os coe nodegroup delete k8s-lab ng-aa`) và kiểm tra server group cũng bị xoá theo.

Lab chỉ có hai compute node, vì vậy hãy dùng `soft-anti-affinity`. Với `anti-affinity` cứng, một nodegroup N node cần **N+1** hypervisor, vì rolling upgrade của Cluster API tạo thêm một node trước khi xoá node cũ.

## Rollback

Xoá file override image rồi chạy lại script cài đặt:

```bash
mv /etc/genestack/helm-configs/magnum/magnum-image-overrides.yaml /root/
/opt/genestack/bin/install-magnum.sh
```

Pod quay về image của rackerlabs. Có hai điều cần lưu ý trước khi rollback:

- **Migration database.** Nếu image mới có migration mà image cũ không có, `db_sync` của image mới đã đổi schema. Image cũ có thể không chạy được trên schema đó. Hãy xem nhánh của bạn có thêm file trong `db/sqlalchemy/alembic/versions` không. Nếu có, backup database `magnum` trước khi thay image.
- **Tài nguyên do code mới tạo.** Ví dụ server group mà driver mới tạo: driver cũ không biết tới chúng, nên sẽ không xoá chúng khi xoá cluster. Hãy xoá nodegroup có dùng tính năng mới trước khi rollback, hoặc dọn tay sau đó.

## Tóm tắt

- Image của mỗi service là một Helm value trong `images.tags`. Ghi đè nó trong một file riêng ở `/etc/genestack/helm-configs/<svc>/`, không sửa base override.
- Image mới phải giữ đúng hợp đồng của image gốc: virtualenv `/var/lib/openstack`, UID 42424, công cụ đi kèm, và cùng nhánh OpenStack. Cách dễ nhất là build dựa trên chính image gốc.
- Mỗi lần build dùng một tag mới. Kiểm tra trước và sau bằng `pip show` trong pod.
- Rollback bằng cách bỏ file override và chạy lại script. Cẩn thận với migration database và tài nguyên mà code mới đã tạo.
