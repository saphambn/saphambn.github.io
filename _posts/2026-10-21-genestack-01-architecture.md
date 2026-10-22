---
title: "OpenStack trên Kubernetes với Genestack – Phần 1: Kiến trúc"
date: 2026-10-21 08:01:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, architecture, kube-ovn, metallb]
render_with_liquid: false
---

Ở [Phần 0](/posts/genestack-00-series-intro/), mình đã giới thiệu Genestack và lab của series. Trước khi gõ lệnh cài đặt đầu tiên, bài này dành để trả lời một câu hỏi: **khi mọi thứ đã chạy, cụm Genestack trông như thế nào?**

Hiểu kiến trúc không chỉ để vẽ sơ đồ cho đẹp. Khi cài đặt, cấu hình hay vận hành, bạn sẽ liên tục cần biết một thành phần nằm ở lớp nào, chạy trên node nào và traffic đi qua những đâu. Bài này là "bản đồ" để quay lại tra cứu trong suốt series.

## Bức tranh tổng thể: năm lớp

Có thể nhìn một cụm Genestack như năm lớp chồng lên nhau, lớp trên dựa vào lớp dưới:

![Kiến trúc Genestack theo lớp: máy chủ, Kubernetes, hạ tầng nền, OpenStack, observability, và các thành phần chạy thẳng trên host](/assets/img/genestack/01-layers.svg)

**Lớp 0 – Máy chủ.** Trong lab là 6 VM: một deployer, ba controller, hai compute. Ở production đây thường là máy vật lý, và compute node cần hỗ trợ ảo hoá phần cứng.

**Lớp 1 – Kubernetes.** Genestack dùng **Kubespray** để dựng cluster, CNI là **kube-ovn**. Đây là lựa chọn quan trọng nhất của kiến trúc, mình sẽ nói kỹ ở phần mạng.

**Lớp 2 – Hạ tầng nền.** Đây là những thứ mà OpenStack truyền thống bắt bạn tự dựng (HAProxy, Keepalived, Galera, RabbitMQ cluster…). Trong Genestack, chúng được thay bằng các thành phần Kubernetes-native, phần lớn chạy dưới dạng **operator**:

| Thành phần | Vai trò | Namespace |
|---|---|---|
| MetalLB | Cấp IP cho Service kiểu `LoadBalancer` (chế độ L2) | `metallb-system` |
| Envoy Gateway | Cổng vào cho mọi API qua Gateway API, route theo hostname | `envoy-gateway`, `envoyproxy-gateway-system` |
| cert-manager | Cấp chứng chỉ TLS | `cert-manager` |
| Longhorn | Persistent volume phân tán cho workload có state | `longhorn-system` |
| mariadb-operator | Dựng và quản lý MariaDB Galera 3 node | `mariadb-system` (operator), `openstack` (cluster) |
| RabbitMQ operator | Message bus cho các service OpenStack | `rabbitmq-system` (operator), `openstack` (cluster) |
| memcached | Cache token và dữ liệu cho các API | `openstack` |

**Lớp 3 – OpenStack.** Mỗi service là một Helm release từ chart của **OpenStack-Helm**, tất cả nằm trong namespace `openstack`. Mỗi service lại tách thành nhiều Deployment/DaemonSet: ví dụ Nova có `nova-api-osapi`, `nova-conductor`, `nova-scheduler`, `nova-novncproxy` và DaemonSet `nova-compute` trên compute node. Ngay cả hypervisor cũng chạy trong pod: `libvirt` là một DaemonSet trên các node có label `openstack-compute-node=enabled`.

**Lớp 4 – Observability.** Prometheus, node-exporter, OpenTelemetry collector, Loki… Lớp này hay bị xem nhẹ, nhưng không chỉ phục vụ người vận hành: dashboard Skyline cũng lấy biểu đồ monitor từ chính Prometheus ở đây.

Có một chi tiết dễ bỏ qua: **không phải mọi thứ đều chạy trong Kubernetes** (khung nét đứt bên phải hình). `cinder-volume` với backend LVM được cài thẳng lên host bằng Ansible và chạy như một systemd service, cùng với iSCSI target `tgt`, vì iSCSI target không hợp với môi trường container. Trên compute node, iSCSI initiator `iscsid` cũng chạy ở host. Mô hình "hybrid" này có hệ quả riêng về mạng, sẽ được nói ở phần dưới.

## Vai trò node được điều khiển bằng label

Genestack không gắn cứng "node này là controller". Nơi mỗi thành phần chạy được quyết định bằng **label** trên node Kubernetes:

| Label | Ý nghĩa | Trong lab |
|---|---|---|
| `openstack-control-plane=enabled` | Chạy API, scheduler, conductor, database, message bus | 3 controller |
| `openstack-network-node=enabled` | Chạy thành phần mạng của Neutron/OVN | 3 controller + 2 compute |
| `openstack-compute-node=enabled` | Chạy `nova-compute`, libvirt | 2 compute |
| `openstack-storage-node=enabled` | Node dành cho storage của OpenStack | 3 controller |
| `longhorn.io/storage-node=enabled` | Longhorn được phép chạy và lưu dữ liệu | 3 controller |

Trong lab, ba controller gánh gần như mọi vai trò: control plane Kubernetes, control plane OpenStack, OVN central, Longhorn, Galera và cả `cinder-volume`. Cách dồn này tiết kiệm máy, nhưng có hai cái giá:

- **Tài nguyên chật**: MariaDB với buffer pool 4 GB, OpenTelemetry collector ăn khoảng 2 core mỗi node, cộng hàng chục pod API… 16 GB RAM cho controller là khá sát.
- **Xung đột giữa các thành phần**: Longhorn và `cinder-volume` LVM cùng dùng iSCSI trên một host – tài liệu Genestack khuyên tránh điều này ở production.

Một mẹo nhỏ khi làm việc với label: một số label có **giá trị rỗng** (ví dụ label `node.kubernetes.io/exclude-from-external-load-balancers` mà Kubespray tự gắn cho node control-plane). Khi xem bằng `kubectl get nodes -L <key>`, cột hiển thị trống y như khi node không có label. Hãy dùng `kubectl get nodes -l <key>` hoặc `--show-labels` để chắc chắn.

## Mạng: ba interface, hai "thế giới"

Mỗi node trong lab có ba interface:

| Interface | Dải mạng | Dùng cho |
|---|---|---|
| `enp3s0` | 10.0.0.0/24 | Mạng host/management: SSH, Kubernetes API, VIP của MetalLB, iSCSI |
| `enp10s0` | 192.168.1.0/24 | Tunnel Geneve của kube-ovn (traffic giữa các pod và giữa các VM) |
| `enp11s0` | không có IP | Provider network của Neutron (bridge `br-ex`) |

Điểm đặc biệt của Genestack là **kube-ovn đóng hai vai cùng lúc**:

1. **CNI của Kubernetes**: mọi pod nhận IP từ dải `10.236.0.0/14`, traffic giữa các pod đi qua tunnel Geneve trên `enp10s0`.
2. **Backend OVN của Neutron**: network, router và floating IP của tenant cũng được hiện thực bằng chính OVN đó.

Nói cách khác, mạng của Kubernetes và mạng của các VM trong OpenStack dùng chung một hệ OVN/Open vSwitch. Lợi ích là bớt một tầng SDN phải vận hành. Đổi lại, khi OVS trên một node gặp sự cố, cả pod lẫn VM trên node đó đều bị ảnh hưởng.

### Traffic từ client vào API

Tất cả API của OpenStack được đưa ra ngoài qua **một VIP duy nhất**:

![Đường đi của request: client, DNS, VIP MetalLB, Envoy Gateway, rồi tới Service của từng API](/assets/img/genestack/01-api-traffic.svg)

MetalLB chọn một node "nhận" VIP và trả lời ARP cho nó. Envoy Gateway (`flex-gateway`) đọc SNI/Host header để chọn HTTPRoute, rồi chuyển request tới Service của API tương ứng.

Hai hệ quả cần nhớ khi kiểm tra kết nối:

- **Phải gọi bằng hostname, không gọi bằng IP.** Envoy chọn route dựa vào hostname (SNI), nên `curl https://10.0.0.200` sẽ bị reset ngay sau TLS handshake. Muốn thử nhanh mà không có DNS, hãy dùng `curl --resolve <hostname>:443:10.0.0.200 https://<hostname>/`.
- **Ping VIP không chứng minh được gì.** Với kube-proxy ở chế độ IPVS, VIP chỉ nhận các port đã khai báo trong Service. Ping sẽ nhận về "port unreachable" dù mọi thứ vẫn bình thường.

Ngoài VIP cho API, MetalLB còn cấp VIP cho MariaDB (`10.0.0.208–210`) và RabbitMQ (`10.0.0.211`) để truy cập từ bên ngoài cluster khi cần.

### Traffic từ service tới database: hai con đường

Đây là chi tiết ít được nhắc tới nhưng nên biết ngay từ đầu. Các service OpenStack kết nối MariaDB qua Service `mariadb-cluster-primary.openstack.svc`. Tuỳ client chạy ở đâu, gói tin đi theo hai con đường khác nhau:

![Hai con đường tới database: pod đi qua OVN load balancer, tiến trình trên host đi qua kube-proxy IPVS](/assets/img/genestack/01-db-paths.svg)

| Client | Con đường | Timeout cho kết nối idle |
|---|---|---|
| Pod trên pod network (gần như mọi service) | **OVN load balancer** của kube-ovn (`enable-lb=true`) | conntrack, 86.400 giây |
| Tiến trình trên host network (ví dụ `cinder-volume` LVM) | **kube-proxy IPVS** | 900 giây (mặc định của kernel) |

Cùng một DNS name, cùng một ClusterIP, nhưng hai chính sách timeout chênh nhau gần 100 lần. Bất kỳ tham số nào liên quan tới thời gian giữ kết nối (như `connection_recycle_time` của oslo.db) đều phải tính cho con đường khắt khe hơn.

## Mô hình cấu hình: base và override

Đây là phần mình khuyên đọc kỹ nhất. Genestack tách cấu hình làm hai nơi:

![Mô hình cấu hình Genestack: /opt/genestack là upstream, /etc/genestack là cấu hình của bạn; base và galera trong /etc là symlink sang /opt](/assets/img/genestack/01-config-model.svg)

**Với các service cài bằng Helm** (đa số service OpenStack, MetalLB, Longhorn…), script `install-<svc>.sh` làm ba việc:

1. đọc version chart từ `/etc/genestack/helm-chart-versions.yaml`;
2. nạp lần lượt mọi file trong `base-helm-configs/<svc>/`, **sau đó** mọi file trong `/etc/genestack/helm-configs/<svc>/` – file nạp sau thắng;
3. chạy `helm upgrade --install`, kèm một post-renderer Kustomize.

Muốn đổi một tham số, bạn tạo file trong `/etc/genestack/helm-configs/<svc>/` rồi chạy lại script. Ví dụ, để MetalLB bỏ qua label exclude nói ở trên:

```yaml
# /etc/genestack/helm-configs/metallb/metallb-helm-overrides.yaml
speaker:
  ignoreExcludeLB: true
```

**Với các thành phần dùng Kustomize** (ví dụ MariaDB cluster), bạn apply từ thư mục `overlay/`. Điều cần nhớ: các thư mục `base/`, `galera/` trong `/etc/genestack/kustomize/<svc>/` thực chất là **symlink trỏ vào `/opt/genestack`**. Mở file `/etc/genestack/kustomize/mariadb-cluster/galera/kustomization.yaml` ra sửa trông có vẻ hợp lý, nhưng thực tế bạn đang sửa mã nguồn upstream. Lần `git pull` tiếp theo sẽ báo conflict hoặc ghi đè mất thay đổi.

Ranh giới này cũng áp dụng cho các thay đổi trực tiếp trên cluster. Một lệnh `SET GLOBAL` trong MariaDB hay một `kubectl patch` có hiệu lực ngay, nhưng sẽ mất ở lần restart hoặc `helm upgrade` kế tiếp. Nguyên tắc: **thay đổi nóng chỉ là tạm thời; trạng thái mong muốn phải luôn nằm trong file override ở `/etc/genestack`.**

## Nơi lưu state

Kubernetes giỏi với workload stateless. Phần khó của OpenStack trên Kubernetes nằm ở state, và Genestack giải quyết bằng ba thành phần:

**Longhorn** cung cấp volume cho các pod cần lưu dữ liệu. Có hai kiểu:
- **RWO** (StorageClass `general`): một pod gắn một volume, dùng cho MariaDB, RabbitMQ…
- **RWX** (StorageClass `general-multi-attach`): nhiều pod cùng gắn, Longhorn hiện thực bằng một pod **share-manager** làm NFS server. Glance dùng kiểu này để mọi pod `glance-api` thấy cùng kho image. Lưu ý rằng share-manager là một thành phần riêng cần được theo dõi: nếu nó không chạy, mọi pod dùng volume RWX đó đều bị ảnh hưởng.

**MariaDB Galera** gồm 3 node do mariadb-operator quản lý. Galera cho phép ghi ở mọi node, nhưng Genestack hướng mọi service vào Service `mariadb-cluster-primary`, Service này chỉ có **một endpoint** là node primary hiện tại. Mô hình single-writer này tránh xung đột certification giữa các node. Đổi lại, mỗi lần primary chuyển sang node khác, các service sẽ lỗi trong vài giây.

**RabbitMQ** chạy dạng cluster, dùng cho RPC và notification giữa các thành phần OpenStack.

## So sánh nhanh với cách triển khai truyền thống

| Tiêu chí | Kolla-Ansible / OpenStack-Ansible | Genestack |
|---|---|---|
| Cài đặt | Playbook Ansible lên từng host | Kubespray + Helm/Kustomize + operator |
| HA cho API | HAProxy + Keepalived tự dựng | MetalLB + Envoy Gateway + nhiều replica |
| Database | Galera tự cấu hình | mariadb-operator |
| Tự phục hồi | systemd/Docker restart, giám sát ngoài | Kubernetes controller |
| Nâng cấp | Chạy lại playbook theo từng node | `helm upgrade`, rolling update |
| Kiến thức cần có | Linux, Ansible, OpenStack | Linux, Ansible, OpenStack **và** Kubernetes, CNI, storage operator |
| Debug | Log trên host | Log pod, sự kiện Kubernetes, *và* log trên host cho phần hybrid |

Không có lựa chọn nào hơn hẳn. Genestack hợp với đội đã quen Kubernetes, muốn tận dụng operator và GitOps. Còn nếu đội bạn chưa vận hành Kubernetes ở quy mô thật, chi phí học thêm là đáng kể.

## Tóm tắt

- Genestack gồm năm lớp: máy chủ → Kubernetes/kube-ovn → hạ tầng nền dạng operator → OpenStack-Helm → observability, cộng vài thành phần hybrid chạy thẳng trên host.
- Vai trò node do **label** quyết định. Hãy cẩn thận với label có giá trị rỗng.
- kube-ovn vừa là CNI vừa là OVN cho Neutron. Mọi API đi qua **một VIP MetalLB** và **Envoy Gateway route theo hostname**.
- Cùng một Service database, pod và tiến trình trên host đi **hai con đường khác nhau**, với timeout khác nhau.
- Mọi tuỳ biến nằm ở `/etc/genestack`. Các thư mục `base/`, `galera/` trong đó là **symlink vào upstream**, đừng sửa.

## Tiếp theo

Ở **Phần 2**, chúng ta bắt đầu dựng lab: chọn kích thước VM, thiết kế ba mạng, và xử lý một điều kiện mà nhiều người bỏ qua khi chạy Genestack trên một OpenStack cloud khác – port security của Neutron sẽ "nuốt" mất VIP của MetalLB nếu không cấu hình đúng.
