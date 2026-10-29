---
title: "OpenStack trên Kubernetes với Genestack – Phần 6: Cinder với LVM và iSCSI"
date: 2026-10-21 08:07:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, cinder, lvm, iscsi, block-storage]
render_with_liquid: false
---

Ở [Phần 5](/posts/genestack-05-openstack-core-services/), cloud đã boot được máy ảo, nhưng disk của mọi VM đều là **ephemeral**: nằm trên compute node và mất theo VM. Bài này thêm **block storage** với **Cinder**: volume độc lập với VM, có thể gắn, tháo, chuyển sang VM khác, và dùng làm ổ boot.

Cinder hỗ trợ nhiều backend: Ceph RBD, NetApp, các storage array… Bài này dùng backend tham chiếu là **LVM qua iSCSI**. Backend này đơn giản và chỉ cần ổ đĩa cục bộ, nên phù hợp để học và thử nghiệm. Nó cũng là ví dụ rõ nhất cho một đặc điểm của Genestack: **mô hình hybrid**, khi một phần của OpenStack chạy ngoài Kubernetes.

## Kiến trúc

![Cinder LVM hybrid: cinder-api và cinder-scheduler trong Kubernetes; cinder-volume và tgt chạy trên host với volume group LVM; compute dùng iscsid để kết nối](/assets/img/genestack/06-cinder-architecture.svg)

Cinder gồm ba thành phần chính, chạy ở hai nơi khác nhau:

| Thành phần | Chạy ở đâu | Vai trò |
|---|---|---|
| `cinder-api` | Pod trong Kubernetes | Nhận request, ghi trạng thái vào database |
| `cinder-scheduler` | Pod trong Kubernetes | Chọn storage node (backend host) cho volume mới |
| `cinder-volume` | **systemd trên storage node** | Tạo, xoá LV trong LVM và export qua iSCSI |

`cinder-volume` với LVM không chạy trong container vì nó cần thao tác trực tiếp lên LVM và iSCSI target của kernel trên host. Genestack cài phần này bằng một **Ansible playbook**:

- tạo virtualenv `/opt/cinder` chứa mã nguồn Cinder;
- cài iSCSI target `tgt`;
- đọc cấu hình từ Secret `cinder-etc` của chart Cinder, rồi ghi ra `/etc/cinder/` trên host;
- tạo systemd unit `cinder-volume` (và `cinder-backup`).

Tiến trình trên host vẫn nói chuyện với RabbitMQ và MariaDB trong cluster qua tên DNS như `rabbitmq.openstack.svc.cluster.local`. Điều này chạy được vì storage node trong lab cũng là node Kubernetes, và Kubespray đã trỏ DNS của host vào nodelocaldns `169.254.25.10` với search domain `svc.cluster.local`. Nếu storage node **không** phải node Kubernetes, bạn phải tự cấu hình DNS cho host đó trỏ về nodelocaldns hoặc CoreDNS.

### Những điều cần hiểu trước khi bắt đầu

**LVM không replicate.** Mỗi storage node chạy một `cinder-volume` độc lập và được đăng ký thành một backend host riêng (ví dụ `openstack-controller-2@lvm-ssd-1`). Nhiều node cùng khai báo `volume_backend_name: lvm-ssd` chỉ tạo thành một **pool** để scheduler chọn chỗ đặt volume. Đó không phải cụm replica:

- mỗi volume nằm trên **đúng một** storage node;
- node đó chết thì volume không truy cập được cho tới khi node sống lại; mất ổ đĩa là mất dữ liệu;
- thêm node giúp tăng dung lượng và phân tán I/O, không tăng độ bền dữ liệu.

Nếu cần dữ liệu được replicate, hãy dùng backend khác như Ceph RBD hoặc một storage array.

**Không nên chạy chung host với Longhorn ở production.** Cả Longhorn và LVM/iSCSI đều dùng iSCSI trên host, và tài liệu Genestack khuyên tách riêng. Lab chạy chung trên 3 controller cho gọn và vẫn hoạt động, nhưng production nên có storage node riêng.

## Bước 1 – Chuẩn bị volume group

Trên mỗi storage node, dành một ổ đĩa riêng cho Cinder. Lab dùng volume 100 GB gắn ở `/dev/vde` (Phần 2):

```bash
pvcreate --metadatasize 2048 /dev/vde
vgcreate cinder-volumes /dev/vde
vgs cinder-volumes
```

Tên `cinder-volumes` phải trùng với `volume_group` trong cấu hình backend ở bước 2.

Khi tạo volume đầu tiên, driver LVM tự dựng một **thin pool** `cinder-volumes-pool` chiếm khoảng 95% VG. Nhờ đó volume là **thin provisioned**: một volume 10 GB chỉ chiếm chỗ theo dữ liệu thực sự được ghi, và tạo snapshot hay clone rất nhanh.

## Bước 2 – Cấu hình backend trong chart Cinder

Cấu hình backend được khai báo trong Helm override của Cinder. Chart dùng nó để sinh Secret `cinder-etc`, và playbook ở bước 4 cũng đọc Secret này:

```yaml
# /etc/genestack/helm-configs/cinder/cinder-helm-overrides.yaml
conf:
  cinder:
    DEFAULT:
      default_availability_zone: az1
      default_volume_type: lvm-ssd
      enabled_backends: lvm-ssd-1
  backends:
    lvm-ssd-1:
      volume_driver: cinder.volume.drivers.lvm.LVMVolumeDriver
      volume_group: cinder-volumes
      volume_backend_name: lvm-ssd
      target_protocol: iscsi
      target_helper: tgtadm
```

| Khoá | Ý nghĩa |
|---|---|
| `lvm-ssd-1` | Tên **backend**: một mục cấu hình, gắn với một driver |
| `volume_backend_name: lvm-ssd` | Tên **pool** mà volume type sẽ trỏ tới |
| `volume_group` | VG trên storage node |
| `target_helper: tgtadm` | Dùng iSCSI target `tgt`, khớp với gói mà playbook cài |
| `default_volume_type` | Volume type mặc định khi người dùng không chỉ định |

Tài liệu mẫu có thể dùng `lioadm` (LIO target của kernel). Playbook `cinder_volumes` của Genestack cài `tgt` và cấu hình `/etc/tgt/conf.d/cinder.conf` để nạp target từ `/var/lib/cinder/volumes/`. Vì vậy hãy dùng **`tgtadm`** cho khớp.

## Bước 3 – Volume type và chart Cinder

Volume type nối lựa chọn của người dùng (`--type lvm-ssd`) với pool backend, thông qua extra spec `volume_backend_name`. Tạo nó trước khi cài hoặc cập nhật chart:

```bash
os volume type create lvm-ssd --property volume_backend_name=lvm-ssd
/opt/genestack/bin/install-cinder.sh
```

`os` là hàm gọi `openstack` CLI qua pod admin client, đã tạo ở Phần 5.

Chart cài `cinder-api` và `cinder-scheduler`, cùng các job khởi tạo database và đăng ký endpoint. Nhớ thêm entry `volume` và `volumev3` trong `global_overrides/endpoints.yaml` (Phần 5) để catalog trỏ tới `https://cinder.api.<domain>`.

## Bước 4 – Cài cinder-volume lên storage node

### Inventory

Khai báo storage node trong nhóm `cinder_storage_nodes` của inventory:

```yaml
# /etc/genestack/inventory/inventory.yaml (trích)
        cinder_storage_nodes:
          vars:
            cinder_backend_name: lvm-ssd-1
            cinder_worker_name: lvm
            storage_network_multipath: false
            storage_network_interface: ansible_enp3s0
          hosts:
            openstack-controller-1: null
            openstack-controller-2: null
            openstack-controller-3: null
```

**`storage_network_interface`** quyết định IP mà iSCSI target quảng bá cho compute (`target_ip_address`). Giá trị là tên **Ansible fact** của interface: `ansible_` cộng tên interface, dấu `-` đổi thành `_`. Ví dụ `enp3s0` thành `ansible_enp3s0`, `br-host` thành `ansible_br_host`. Giá trị mặc định của role là `ansible_br_storage`, giả định có một bridge storage riêng. Nếu node không có bridge đó, hãy khai báo rõ.

Lab không có mạng storage riêng nên dùng `enp3s0`, mạng quản trị mà compute cũng nằm trên. Ở production, nên tách mạng storage để:
- traffic I/O không tranh băng thông với API và database;
- dùng được jumbo frame (MTU 9000);
- bật multipath: khai báo thêm `storage_network_interface_secondary` và đặt `storage_network_multipath: true`.

### Chạy playbook

```bash
source /opt/genestack/scripts/genestack.rc
ansible-playbook /opt/genestack/ansible/playbooks/deploy-cinder-volume.yaml \
  -e cinder_backend_name=lvm-ssd-1 \
  -e cinder_worker_name=lvm \
  -e cinder_release_branch=stable/2025.1
```

**Luôn chỉ định `cinder_release_branch`** cho khớp với version OpenStack của cluster (Cinder trong chart là 2025.1). Playbook cài Cinder từ git theo nhánh này, và giá trị mặc định của role có thể là một nhánh cũ hơn.

Mỗi lần thay đổi override Cinder (bước 2), hãy chạy lại `install-cinder.sh` rồi tới playbook này, để cấu hình trên host được cập nhật theo Secret mới.

Kết quả trên mỗi storage node:

| Thành phần | Chi tiết |
|---|---|
| `/opt/cinder` | Virtualenv chứa Cinder 2025.1 |
| `/etc/cinder/cinder.conf`, `backends.conf` | Sinh từ Secret `cinder-etc`. `host` được đặt bằng tên node, `target_ip_address` lấy theo `storage_network_interface` |
| `cinder-volume.service` | systemd, enable khi khởi động |
| `tgt.service` | iSCSI target, lắng nghe port 3260 |

Playbook cũng cài `cinder-backup.service`. Service này chỉ hữu ích khi bạn cấu hình backend backup (Swift, Ceph, NFS…) cho Cinder. Đây là chủ đề riêng, nằm ngoài phạm vi bài này.

## Bước 5 – Bật iSCSI cho Nova

Compute node cần iSCSI initiator để kết nối tới volume:

1. **Trên host:** cài `open-iscsi` và bật `iscsid`. Playbook `host-setup.yml` (Phần 3) làm việc này cho các node trong nhóm compute khi biến `enable_iscsi: true`:
   ```yaml
   openstack_compute_nodes:
     vars:
       enable_iscsi: true
   ```
2. **Trong chart Nova:** bật iSCSI cho `nova-compute`, để pod dùng iscsid của host:
   ```yaml
   # /etc/genestack/helm-configs/nova/nova-helm-overrides.yaml
   conf:
     enable_iscsi: true
   ```
   Rồi chạy lại `/opt/genestack/bin/install-nova.sh`.

Mỗi host có một iSCSI initiator name riêng trong `/etc/iscsi/initiatorname.iscsi`. Nếu các VM được tạo từ cùng một image, hãy kiểm tra tên này không bị trùng giữa các node.

Trên image Ubuntu cloud, file này ban đầu chỉ có dòng `GenerateName=yes`. IQN thật chỉ được sinh ra (bởi `iscsi-init.service`) khi `iscsid` khởi động lần đầu. Nếu `iscsid` chưa từng chạy, nova-compute gửi connector không có initiator, và mọi thao tác gắn volume hoặc boot-from-volume đều lỗi `Connector doesn't have required information: initiator`. Kiểm tra trên từng compute trước khi đi tiếp:

```bash
systemctl is-active iscsid
grep -v '^#' /etc/iscsi/initiatorname.iscsi    # phải là InitiatorName=iqn....
# nếu vẫn là GenerateName=yes:
systemctl enable --now iscsid.socket iscsid.service
```

## Kiểm tra

```bash
os volume service list
```

```
+------------------+----------------------------------+------+---------+-------+
| Binary           | Host                             | Zone | Status  | State |
+------------------+----------------------------------+------+---------+-------+
| cinder-scheduler | cinder-volume-worker             | az1  | enabled | up    |
| cinder-volume    | openstack-controller-1@lvm-ssd-1 | az1  | enabled | up    |
| cinder-volume    | openstack-controller-2@lvm-ssd-1 | az1  | enabled | up    |
| cinder-volume    | openstack-controller-3@lvm-ssd-1 | az1  | enabled | up    |
+------------------+----------------------------------+------+---------+-------+
```

Mỗi storage node xuất hiện dưới dạng `<node>@<backend>`. Một node ở trạng thái `down` thường do `cinder-volume` không khởi động được, ví dụ vì thiếu VG `cinder-volumes`. Xem log bằng `journalctl -u cinder-volume -n 50`.

## Vòng đời của một volume

![Vòng đời volume: cinder-api, scheduler, cinder-volume tạo LV; khi gắn, nova-compute gọi Cinder, tgt tạo target, iscsid login, libvirt gắn vào VM](/assets/img/genestack/06-volume-flow.svg)

Thử toàn bộ đường đi bằng một VM **boot từ volume**:

```bash
# Tạo volume từ image (Cinder tải image từ Glance rồi ghi vào LV)
os volume create test-boot-vol --image "Cirros 0.6.2 64-bit" --size 2 --type lvm-ssd --bootable
os volume show test-boot-vol -c status -c os-vol-host-attr:host

# Boot VM từ volume đó
os server create test-vm-bfv --flavor m1.tiny --volume test-boot-vol --network test-net --wait

# Thêm một volume dữ liệu
os volume create test-data-vol --size 1 --type lvm-ssd
os server add volume test-vm-bfv test-data-vol
```

Rồi kiểm tra từng chặng:

```bash
# Trên storage node chứa volume (theo os-vol-host-attr:host)
lvs cinder-volumes
tgtadm --lld iscsi --mode target --op show | grep -E "^Target|IP Address|Backing store path"

# Trên compute node đang chạy VM
iscsiadm -m session

# VM boot được
os console log show test-vm-bfv | tail
```

Kết quả mong đợi:

- LV mang tên ID của volume nằm trong thin pool `cinder-volumes-pool`;
- `tgtadm` thấy target `iqn.2010-10.org.openstack:<volume-id>`, kèm IP của compute đang kết nối;
- `iscsiadm` trên compute có session tới `<storage-ip>:3260`;
- console log của VM hiện màn hình đăng nhập.

### Khi có lỗi

| Triệu chứng | Nơi xem đầu tiên |
|---|---|
| Volume `error` ngay khi tạo, `No valid host was found` | Volume type có extra spec `volume_backend_name` khớp backend chưa; `cinder-volume` có `up` không |
| Volume `error` khi tạo từ image | Log `cinder-volume` trên storage node; Glance có trả được dữ liệu image không |
| Kẹt ở `attaching` | `iscsid` trên compute; compute có tới được `<storage-ip>:3260` không; log `nova-compute` |
| `Connector doesn't have required information: initiator` | `/etc/iscsi/initiatorname.iscsi` trên compute vẫn là `GenerateName=yes`: bật `iscsid` (xem Bước 5) |
| `tgtadm` không có target | `journalctl -u cinder-volume` trên storage node |

## Tóm tắt

- Cinder LVM chạy hybrid: API và scheduler là pod, `cinder-volume` cùng `tgt` là systemd service trên storage node, cài bằng playbook `deploy-cinder-volume.yaml`.
- Mỗi storage node là một backend host độc lập. Cùng `volume_backend_name` nghĩa là cùng pool, không phải replica.
- Dùng `target_helper: tgtadm`. Chỉ định `cinder_release_branch` khớp version cluster. Khai báo `storage_network_interface` dưới dạng tên Ansible fact.
- Compute cần `iscsid` trên host và `conf.enable_iscsi: true` trong chart Nova.
- Kiểm tra từng chặng: LV → target `tgt` → session iSCSI → console của VM.

## Tiếp theo

Ở **Phần 7**, chúng ta làm cho máy ảo có tính sẵn sàng cao với **Masakari**: tự phát hiện compute node gặp sự cố và dựng lại VM trên node khác.
