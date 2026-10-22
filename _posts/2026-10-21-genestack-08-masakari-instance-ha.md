---
title: "OpenStack trên Kubernetes với Genestack – Phần 7: Instance HA với Masakari"
date: 2026-10-21 08:08:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, masakari, high-availability, nova]
render_with_liquid: false
---

Đến [Phần 6](/posts/genestack-06-cinder-lvm/), cloud đã có compute, mạng và block storage. Control plane có tính sẵn sàng cao nhờ Kubernetes: pod API chết thì được tạo lại, node controller chết thì pod chuyển sang node khác. Nhưng **máy ảo** thì không: khi một compute node chết, mọi VM trên đó dừng theo, và mặc định OpenStack không tự làm gì cả.

**Masakari** là service lấp vào khoảng trống đó: nó phát hiện compute node, tiến trình VM hoặc hệ điều hành trong VM gặp sự cố, rồi tự phục hồi VM. Bài này cài Masakari, cấu hình nó, và thử tắt một compute node để xem VM được dựng lại trên node còn lại.

## Masakari hoạt động thế nào

![Masakari: ba loại monitor trên compute gửi notification tới masakari-api; masakari-engine phục hồi VM qua Nova API; host phải thuộc một failover segment](/assets/img/genestack/08-masakari-components.svg)

Masakari gồm hai phần:

**Monitor** chạy trên mọi compute node, dạng DaemonSet. Có ba loại:

| Monitor | Phát hiện | Phục hồi |
|---|---|---|
| `masakari-host-monitor` | Cả compute node không hoạt động | Evacuate VM sang compute khác |
| `masakari-instance-monitor` | Tiến trình QEMU của một VM chết bất thường | Hard reboot VM |
| `masakari-introspective-instance-monitor` | Hệ điều hành trong VM treo, phát hiện qua QEMU guest agent | Tắt rồi bật lại VM |

**Control plane** chạy trên controller:

- `masakari-api` nhận notification từ monitor và quản lý cấu hình;
- `masakari-engine` chạy workflow phục hồi, gọi Nova API để disable compute service, evacuate hoặc reboot VM.

### Host monitor trong Genestack

Masakari truyền thống dùng Pacemaker/Corosync để phát hiện host chết. Genestack dùng driver **`kubernetes`**: host monitor hỏi Kubernetes API trạng thái của các node compute, vì Kubernetes vốn đã theo dõi sức khoẻ node qua kubelet. Cấu hình mặc định:

```ini
[host]
monitoring_driver = kubernetes
monitoring_interval = 30      # giây giữa hai lần kiểm tra
monitoring_samples = 3        # số mẫu lỗi liên tiếp trước khi báo
disable_ipmi_checks = true

[kubernetes]
monitoring_node_labels = openstack-compute-node=enabled
```

Mỗi compute node chạy một host monitor, và các monitor theo dõi chéo lẫn nhau. Khi một node bị Kubernetes đánh dấu `NotReady` đủ 3 lần kiểm tra liên tiếp, một monitor trên node khác sẽ gửi notification. Vì vậy cần ít nhất **2 compute node**. Đó cũng là điều kiện tối thiểu để có chỗ evacuate VM sang.

### Failover segment

Masakari chỉ xử lý notification của những host nằm trong một **failover segment**. Segment là một nhóm compute node cùng chung chính sách phục hồi:

| Recovery method | Hành vi |
|---|---|
| `auto` | Evacuate VM, để Nova scheduler tự chọn node đích |
| `reserved_host` | Evacuate VM sang các host được đánh dấu *reserved* (dự phòng, bình thường không chạy VM) |
| `auto_priority` | Thử `auto` trước, thất bại thì dùng reserved host |
| `rh_priority` | Thử reserved host trước, thất bại thì dùng `auto` |

`reserved_host` đảm bảo luôn có sẵn tài nguyên trống để nhận VM, đổi lại phải "để không" một số node. Với cluster nhỏ, `auto` là lựa chọn hợp lý.

## Cài đặt

Mật khẩu của Masakari (database, RabbitMQ, Keystone) đã được `create-secrets.sh` tạo ở Phần 4b. Cài chart:

```bash
/opt/genestack/bin/install-masakari.sh
```

Thêm entry `instance_ha` vào `global_overrides/endpoints.yaml` (Phần 5) nếu muốn đưa API Masakari ra ngoài qua gateway. Khi chỉ dùng CLI từ trong cluster thì không cần.

Kiểm tra:

```bash
kubectl -n openstack get deploy,ds | grep masakari
```

```
deployment.apps/masakari-api                               2/2
deployment.apps/masakari-engine                            1/1
daemonset.apps/masakari-host-monitor                       2
daemonset.apps/masakari-instance-monitor                   2
daemonset.apps/masakari-introspective-instance-monitor     2
```

Các DaemonSet chạy trên node có label `openstack-compute-node=enabled`, nên trong lab có 2 pod cho mỗi loại.

Log của host monitor cho thấy nó đang theo dõi node nào:

```
$ kubectl -n openstack logs ds/masakari-host-monitor --tail=1
… Monitoring data: {'openstack-compute-1': deque(['True', 'True', 'True'], maxlen=3),
                    'openstack-compute-2': deque(['True', 'True', 'True'], maxlen=3)}
```

## Cấu hình

**Ngay sau khi cài, Masakari chưa bảo vệ VM nào.** Hai điều kiện sau phải được thoả.

### 1. Tạo segment và thêm host

`os` là hàm gọi `openstack` CLI qua pod admin client (Phần 5).

```bash
os segment create --description "compute HA" compute-ha auto COMPUTE

os segment host create openstack-compute-1 COMPUTE SSH compute-ha
os segment host create openstack-compute-2 COMPUTE SSH compute-ha

os segment host list compute-ha
```

Tên host phải **trùng chính xác** với tên host của `nova-compute` (cột `Host` trong `os compute service list`). Nếu khác, notification sẽ không khớp với host nào và bị bỏ qua.

Tham số `SSH` là trường `control_attributes`, một chuỗi mô tả cách điều khiển host, dùng cho mục đích quản lý. Nó **không** có nghĩa là Masakari sẽ SSH vào node để fencing.

### 2. Chọn VM được bảo vệ

Cấu hình mặc định của Genestack có:

```ini
[host_failure]
evacuate_all_instances = false
```

Nghĩa là khi host chết, Masakari **chỉ** evacuate những VM có metadata `HA_Enabled=True`:

```bash
os server set --property HA_Enabled=True <vm>
```

Muốn mọi VM trên host lỗi đều được evacuate, đặt giá trị này thành `true` qua Helm override rồi chạy lại `install-masakari.sh`:

```yaml
# /etc/genestack/helm-configs/masakari/masakari-helm-overrides.yaml
conf:
  masakari:
    host_failure:
      evacuate_all_instances: true
```

Gắn nhãn theo từng VM linh hoạt hơn: chỉ những workload thực sự cần HA mới chiếm tài nguyên dự phòng trên các node còn lại.

### Introspection (tuỳ chọn)

Introspective monitor cần **QEMU guest agent** chạy bên trong VM. Image phải có property:

```bash
os image set --property hw_qemu_guest_agent=yes <image>
```

Và VM phải có `HA_Enabled=True`. Có thể tinh chỉnh tần suất kiểm tra qua override:

```yaml
conf:
  masakarimonitors:
    introspectiveinstancemonitor:
      guest_monitor_interval: 10
      guest_monitor_timeout: 5
```

Ở production, hãy cân nhắc kỹ các ngưỡng này: một VM đang quá tải có thể phản hồi guest agent chậm, và bị Masakari khởi động lại oan.

## Thử nghiệm: tắt một compute node

Chuẩn bị một VM **boot từ volume** (Phần 6), có `HA_Enabled=True`, đang chạy trên `openstack-compute-2`:

```bash
os server show test-vm-bfv -c status -c OS-EXT-SRV-ATTR:host -c properties
# status: ACTIVE · host: openstack-compute-2 · properties: HA_Enabled='True'
```

Rồi **tắt hẳn** compute-2: shutdown hệ điều hành, hoặc stop máy ảo/máy chủ từ bên ngoài. Theo dõi:

```bash
kubectl get nodes -w                         # compute-2 → NotReady
os notification list                         # notification COMPUTE_HOST xuất hiện
os server event list test-vm-bfv             # lock → evacuate → unlock
os server show test-vm-bfv -c status -c OS-EXT-SRV-ATTR:host
```

![Timeline: notification, disable nova-compute sau 1 giây, lock và evacuate sau 94 giây, VM ACTIVE trên compute còn lại sau 138 giây](/assets/img/genestack/08-evacuation-timeline.svg)

Kết quả trong lab:

| Thời điểm | Sự kiện |
|---|---|
| Trước T | Kubernetes đánh dấu compute-2 `NotReady`. Host monitor trên compute-1 ghi nhận đủ 3 mẫu lỗi |
| T | Notification `COMPUTE_HOST`, sự kiện `STOPPED` |
| T + 1 s | `DisableComputeServiceTask`: `nova-compute` trên compute-2 bị disable |
| T + 94 s | Masakari lock VM rồi gọi evacuate (chờ Nova xác nhận host đã down) |
| T + 138 s | VM `ACTIVE` trên compute-1, được unlock. Notification chuyển sang `finished` |

Chi tiết từng bước của workflow:

```bash
os notification show <notification-uuid> -f yaml -c status -c recovery_workflow_details
```

Các task `DisableComputeServiceTask`, `PrepareHAEnabledInstancesTask`, `EvacuateInstancesTask` đều ở trạng thái `SUCCESS`. VM không có `HA_Enabled` thì vẫn nằm trên compute-2 ở trạng thái lỗi, đúng với cấu hình.

Trên compute-1 có thêm một iSCSI session tới storage node chứa volume (`iscsiadm -m session`), và console log cho thấy VM đã boot lại.

## Những điều cần biết trước khi dùng thật

**Dữ liệu và loại disk.** Evacuate nghĩa là Nova dựng lại VM trên node khác:
- VM **boot từ volume** (Cinder) được gắn lại đúng volume cũ, nên **giữ nguyên dữ liệu**. Với Cinder LVM, điều kiện là storage node chứa volume vẫn còn sống.
- VM dùng **disk ephemeral** (boot từ image) được dựng lại từ image gốc, nên **mất toàn bộ dữ liệu** đã ghi trên disk.

Tài liệu Genestack cũng nhấn mạnh: muốn evacuate có ý nghĩa, VM phải dùng storage dùng chung.

**Fencing.** Genestack đặt `disable_ipmi_checks = true`, tức là Masakari không tự tắt node lỗi qua IPMI. Nếu một node chỉ **mất mạng** nhưng vẫn chạy, VM cũ trên đó vẫn tiếp tục ghi vào volume, trong khi bản evacuate trên node mới cũng ghi vào cùng volume đó. Hai bản cùng ghi sẽ làm hỏng dữ liệu. Khi thử nghiệm, hãy **tắt hẳn** node. Ở production, cần có cơ chế fencing (IPMI, tắt nguồn qua PDU, hoặc qua API của hạ tầng) để chắc chắn node lỗi đã dừng hẳn.

**Tài nguyên.** Các node còn lại phải đủ CPU và RAM cho VM được chuyển sang. Nếu không, VM sẽ rơi vào `ERROR`. Hãy tính trước dung lượng dự phòng, hoặc dùng reserved host.

**Công cụ quản trị.** Những pod tiện ích chạy trên node lỗi (ví dụ pod admin client) cũng dừng theo. Nên đặt chúng trên controller, hoặc có sẵn cách khác để chạy CLI.

## Sau khi node lỗi hoạt động lại

Masakari **không** tự đưa host trở lại phục vụ. Sau khi phục hồi, host lỗi vẫn ở trạng thái:

- `nova-compute` bị **disabled**: scheduler không đặt VM mới lên đó;
- host trong segment có `on_maintenance = True`: Masakari bỏ qua host này.

Khi node đã khoẻ trở lại:

```bash
os compute service set --enable openstack-compute-2 nova-compute
os segment host update compute-ha openstack-compute-2 --on_maintenance False
```

Khi `nova-compute` trên node đó khởi động lại, nó tự dọn các bản VM cũ đã được evacuate sang node khác.

## Tóm tắt

- Masakari có ba loại monitor (host, instance, introspection) chạy trên compute, cùng API và engine trên controller.
- Genestack dùng driver `kubernetes` cho host monitor. Mặc định kiểm tra 30 giây một lần, cần 3 mẫu lỗi liên tiếp. Cần tối thiểu 2 compute.
- Masakari chỉ hoạt động khi host thuộc một **segment**, và chỉ evacuate VM có **`HA_Enabled=True`** (trừ khi bật `evacuate_all_instances`).
- Trong lab, VM boot từ volume được dựng lại trên compute khác khoảng 2 phút 20 giây sau notification.
- VM boot từ volume giữ dữ liệu; VM ephemeral thì không. Không có fencing thì phải chắc chắn node lỗi đã tắt hẳn.
- Sau sự cố, phải tự enable lại `nova-compute` và tắt `on_maintenance` cho host.
