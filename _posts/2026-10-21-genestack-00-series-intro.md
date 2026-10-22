---
title: "OpenStack trên Kubernetes với Genestack – Phần 0: Giới thiệu series"
date: 2026-10-21 08:00:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, openstack-helm, devops]
render_with_liquid: false
---

Ai từng vận hành OpenStack đều nhớ cảm giác ấy: hàng chục service, mỗi service vài tiến trình, cấu hình rải khắp các node, và mỗi lần nâng cấp là một lần nín thở. OpenStack mạnh, linh hoạt, nhưng chưa bao giờ được gọi là "dễ vận hành".

Series này hướng dẫn dựng một cụm OpenStack chạy **trên Kubernetes** bằng **Genestack** – từ lúc chuẩn bị máy, cài đặt, vận hành hằng ngày cho tới cách chẩn đoán khi có thành phần không hoạt động. Ngoài các bước cài đặt, series ghi chú cả những điểm cần lưu ý mà tài liệu chính thức chưa nói rõ.

## Vì sao lại là "OpenStack trên Kubernetes"?

Với cách triển khai truyền thống, OpenStack được cài trực tiếp lên máy (package, virtualenv, LXC hay Docker container quản lý bằng Ansible). Cách này đã chứng minh được độ bền qua nhiều năm, nhưng người vận hành phải tự lo khá nhiều thứ:

- **High availability**: tự dựng HAProxy, Keepalived, Pacemaker… và tự theo dõi chúng.
- **Tự phục hồi**: một tiến trình chết thì phải có systemd, monitoring hoặc con người can thiệp.
- **Nâng cấp**: chạy lại playbook dài, rolling từng node, rủi ro cao nếu giữa chừng có lỗi.
- **Cấu hình**: trạng thái mong muốn nằm trong playbook, trạng thái thực tế nằm trên từng máy – và chúng không phải lúc nào cũng khớp.

Kubernetes đã giải quyết đúng những bài toán này cho ứng dụng cloud-native: khai báo trạng thái mong muốn, controller lo phần còn lại; pod chết thì được tạo lại; rolling update là tính năng có sẵn. Ý tưởng đưa control plane của OpenStack lên Kubernetes vì thế rất tự nhiên – và cộng đồng đã làm điều này từ lâu với dự án **OpenStack-Helm**.

Tất nhiên, không có bữa trưa miễn phí. Bạn đổi độ phức tạp của OpenStack lấy độ phức tạp *cộng thêm* của Kubernetes: CNI, storage cho stateful workload, load balancer, operator cho database… Series này sẽ cho bạn thấy rõ cả hai mặt.

## Các lựa chọn để triển khai OpenStack

| Công cụ | Cách triển khai | Ghi chú |
|---|---|---|
| **OpenStack-Ansible** | Ansible, service chạy trong LXC/virtualenv trên host | Trưởng thành, tuỳ biến sâu |
| **Kolla-Ansible** | Ansible, service chạy trong Docker container | Phổ biến, dễ bắt đầu |
| **OpenStack-Helm** | Helm chart trên Kubernetes | Bộ chart nền tảng, cần tự ghép phần hạ tầng |
| **Genestack** | OpenStack-Helm + Kustomize + các Kubernetes operator | "Bộ khung hoàn chỉnh" có quan điểm thiết kế rõ ràng |

## Genestack là gì?

**Genestack** là dự án mã nguồn mở của **Rackspace**, dùng để triển khai OpenStack trên Kubernetes. Có thể hình dung Genestack là một "distribution": nó không viết lại OpenStack-Helm mà ghép OpenStack-Helm với một bộ thành phần hạ tầng đã được chọn sẵn, kèm script cài đặt, cấu hình mặc định và tài liệu vận hành.

Một vài thành phần bạn sẽ gặp xuyên suốt series:

- **Kubespray** để dựng Kubernetes, với **kube-ovn** làm CNI – đồng thời là OVN backend cho Neutron.
- **MetalLB** cấp IP LoadBalancer, **Envoy Gateway** (Gateway API) làm cổng vào cho các API.
- **Longhorn** cung cấp persistent volume cho các thành phần có state.
- **mariadb-operator** dựng MariaDB **Galera** 3 node, cùng operator cho RabbitMQ.
- **kube-prometheus-stack** và OpenTelemetry cho monitoring.
- Các service OpenStack quen thuộc: Keystone, Glance, Placement, Nova, Neutron, Cinder, Heat, Skyline, Masakari…

Điều mình thích nhất ở Genestack là **mô hình cấu hình hai lớp**:

```
/opt/genestack/   ← mã nguồn upstream (base config, script, tài liệu) – không sửa
/etc/genestack/   ← cấu hình riêng của bạn (override, inventory, overlay)
```

Mọi tuỳ biến nằm ở `/etc/genestack`, phần upstream giữ nguyên để nâng cấp bằng `git`. Nghe đơn giản, nhưng như bạn sẽ thấy ở các bài sau, hiểu sai ranh giới này là nguồn gốc của khá nhiều rắc rối.

## Lab của series

Toàn bộ series dựa trên một lab thật, chạy trên **6 máy ảo** của một OpenStack public cloud (tức là OpenStack *lồng* trong OpenStack):

| Node | Số lượng | Vai trò |
|---|---|---|
| deployer | 1 | Máy điều khiển: chạy Ansible, `kubectl`, `helm`; là bastion duy nhất có IP public |
| openstack-controller | 3 | Kubernetes control plane + OpenStack control plane, OVN, Longhorn, MariaDB Galera, cinder-volume |
| openstack-compute | 2 | nova-compute (QEMU), các agent mạng, Masakari monitor |

Mỗi node có ba mạng: một mạng management/host, một mạng riêng cho tunnel Geneve của kube-ovn, và một interface provider không gán IP cho mạng external của Neutron.

Phiên bản chính được dùng:

| Thành phần | Phiên bản |
|---|---|
| Genestack | 2026.2.0.2 |
| OpenStack (image) | 2025.1 |
| Kubernetes | v1.35 |
| Hệ điều hành node | Ubuntu 24.04 |
| MetalLB | 0.15.2 |
| Longhorn | 1.11.1 |
| mariadb-operator | 26.3.0 (MariaDB 11.8) |

Cuối series, cụm sẽ có: tạo network và boot VM, block storage với Cinder LVM/iSCSI (boot-from-volume), dashboard Skyline truy cập từ laptop, trang monitor lấy số liệu từ Prometheus, và **instance HA** – tắt một compute node thì VM tự được dựng lại trên node còn lại.

> **Lưu ý:** đây là lab *nested* để học và thử nghiệm. Hiệu năng, độ ổn định mạng và kích thước node không đại diện cho môi trường production. Ở mỗi bài, mình sẽ ghi rõ đâu là giá trị cho lab và đâu là khuyến nghị cho production.

## Lộ trình series

Series chia thành năm phần:

1. **Nền tảng** – kiến trúc Genestack: các lớp, vai trò node, mạng, và mô hình cấu hình.
2. **Cài đặt** – chuẩn bị hạ tầng VM, dựng Kubernetes, lớp hạ tầng (MetalLB, Envoy Gateway, Longhorn, MariaDB, RabbitMQ), các service OpenStack, Cinder LVM và Skyline.
3. **Vận hành** – instance HA với Masakari, quy trình thay đổi cấu hình an toàn, rolling update cho thành phần có state, backup và nâng cấp.
4. **Chẩn đoán và khắc phục** – cách kiểm tra từng lớp khi có thành phần không hoạt động:
   - MetalLB L2: kiểm tra VIP đã được announce và trả lời ARP hay chưa;
   - Longhorn: theo dõi volume, đặc biệt là volume RWX và share-manager;
   - MariaDB Galera: trạng thái cluster, timeout kết nối, và cách đánh giá một thay đổi cấu hình trước khi áp dụng;
   - bộ lệnh tra cứu nhanh theo từng lớp.
5. **Tổng kết** – các điểm cần lưu ý khi cấu hình, và checklist trước khi nghĩ tới production.

Các bài chẩn đoán đi theo một quy trình chung: từ triệu chứng, kiểm tra lần lượt từng lớp cho tới khi tìm ra nguyên nhân, rồi khắc phục và xác nhận lại.

## Bạn cần gì để theo dõi series?

- **Kiến thức**: Linux và mạng cơ bản (ARP, VLAN/overlay, NAT), làm việc được với `kubectl`, nắm các khái niệm OpenStack như project, network, image, volume.
- **Công cụ**: SSH, `openstack` CLI, `kubectl`, `helm`.
- **Hạ tầng** (nếu muốn làm theo): khoảng 6 VM hoặc máy vật lý. Controller nên có từ 8 vCPU và tốt nhất là 32 GB RAM – lab dùng 16 GB và đã thấy chật.

## Tiếp theo

Ở **Phần 1**, chúng ta sẽ mổ xẻ kiến trúc Genestack: các lớp từ máy chủ tới OpenStack, cách traffic đi từ client vào API, và vì sao cần hiểu rõ ranh giới giữa hai thư mục `/opt/genestack` và `/etc/genestack` trước khi tuỳ biến bất cứ thứ gì.

Hẹn gặp lại ở bài sau!
