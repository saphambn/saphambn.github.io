---
title: "OpenStack trên Kubernetes với Genestack – Phần 2: Chuẩn bị hạ tầng lab"
date: 2026-10-21 08:02:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, lab, networking, port-security]
render_with_liquid: false
---

Ở [Phần 1](/posts/genestack-01-architecture/), chúng ta đã có bức tranh tổng thể về kiến trúc Genestack. Bài này bắt tay vào việc đầu tiên: **chuẩn bị máy và mạng** để sẵn sàng cài Kubernetes ở Phần 3.

Lab của series chạy trên các máy ảo của một OpenStack cloud khác – tức là OpenStack *lồng* trong OpenStack. Cách làm này tiện cho việc học: tạo và xoá máy trong vài phút, không cần phần cứng riêng. Nhưng nó cũng có vài yêu cầu riêng về mạng mà nếu bỏ qua, các bước sau sẽ không chạy. Nếu bạn dùng máy vật lý, phần lớn nội dung vẫn áp dụng được, chỉ cần bỏ qua các mục dành riêng cho cloud.

## Tổng quan lab

![Topology lab: deployer, 3 controller, 2 compute nối vào 3 network br-host, br-overlay và br-ex](/assets/img/genestack/02-topology.svg)

Lab gồm 6 máy ảo chạy Ubuntu 24.04:

| Máy | Số lượng | vCPU | RAM | Vai trò |
|---|---|---|---|---|
| deployer | 1 | 2 | 6 GB | Chạy Ansible, `kubectl`, `helm`; điểm SSH duy nhất từ Internet |
| openstack-controller | 3 | 8 | 16 GB | Control plane của Kubernetes và OpenStack, OVN, Longhorn, MariaDB, storage |
| openstack-compute | 2 | 8 | 16 GB | `nova-compute`, libvirt, agent mạng |

Chỉ deployer có floating IP. Mọi thao tác với các node còn lại đều đi qua deployer.

## Sizing: lab và khuyến nghị

Cấu hình trên là mức tối thiểu để chạy được. Controller là nơi chật nhất, vì phải gánh cùng lúc:

- Kubernetes control plane (etcd, API server, scheduler, controller-manager);
- hàng chục pod API của OpenStack, mỗi service thường có 2–3 replica;
- MariaDB Galera, với buffer pool mặc định của Genestack là 4 GB;
- RabbitMQ, memcached, Longhorn;
- các agent observability. Riêng OpenTelemetry collector có thể dùng khoảng 2 core trên mỗi node.

| Vai trò | Lab | Khuyến nghị cho môi trường thử nghiệm nghiêm túc |
|---|---|---|
| deployer | 2 vCPU · 6 GB | 2–4 vCPU · 8 GB |
| controller | 8 vCPU · 16 GB | **8+ vCPU · 32 GB** |
| compute | 8 vCPU · 16 GB | Tuỳ số VM định chạy |

Với 16 GB RAM, controller vẫn chạy được, nhưng gần như không còn dư cho việc bật thêm service.

### Ảo hoá lồng nhau

Compute node trong lab là máy ảo, và cloud bên dưới **không bật nested virtualization**: trong VM không có `/dev/kvm`, CPU cũng không có cờ `vmx`/`svm`. Vì vậy Nova chạy ở chế độ `virt_type = qemu`, tức là giả lập hoàn toàn bằng phần mềm. VM vẫn boot được, nhưng chậm hơn nhiều so với KVM. Như vậy là đủ để học và kiểm tra chức năng. Để đo hiệu năng, bạn cần máy vật lý hoặc một cloud có hỗ trợ nested virtualization.

Kiểm tra trên compute node:

```bash
ls -l /dev/kvm                          # có file này nghĩa là dùng được KVM
grep -cE '(vmx|svm)' /proc/cpuinfo      # > 0 nghĩa là CPU có hỗ trợ ảo hoá
```

## Mạng: vì sao cần ba network

| Network | Dải IP | Interface trên node | Port security | Dùng cho |
|---|---|---|---|---|
| `br-host` | 10.0.0.0/24 | `enp3s0` | **Bật** | SSH, Kubernetes API, VIP của MetalLB, iSCSI |
| `br-overlay` | 192.168.1.0/24 | `enp10s0` | Tắt | Tunnel Geneve của kube-ovn |
| `br-ex` | 192.168.10.0/24 | `enp11s0`, không gán IP | Tắt | Provider network của Neutron |

**`br-host`** là mạng quản trị. Mọi thứ "của hạ tầng" đi qua đây: SSH, traffic giữa các thành phần Kubernetes, và VIP mà MetalLB dùng để đưa API ra ngoài.

**`br-overlay`** chỉ dùng cho tunnel. kube-ovn đóng gói traffic giữa các pod (và sau này là giữa các VM của tenant) trong tunnel Geneve. Tách riêng mạng này giúp traffic dữ liệu không tranh băng thông với traffic quản trị, đồng thời cho phép tinh chỉnh MTU riêng.

**`br-ex`** là đường ra ngoài cho các VM do OpenStack bên trong tạo ra. Interface `enp11s0` trên node **không được gán IP**: Neutron/OVN gắn nó vào bridge `br-ex` và tự quản lý traffic trên đó.

### Port security và VIP của MetalLB

Đây là điểm quan trọng nhất của bài nếu bạn chạy lab trên một OpenStack cloud.

Neutron mặc định bật **port security** cho mọi port: gói tin đi ra từ một VM chỉ được chấp nhận nếu địa chỉ IP và MAC nguồn khớp với IP/MAC đã gán cho port đó. Cơ chế này chống giả mạo địa chỉ, nhưng nó xung đột trực tiếp với cách MetalLB hoạt động ở chế độ L2: một node sẽ trả lời ARP và nhận traffic cho VIP `10.0.0.200`, một địa chỉ không thuộc port nào cả. Neutron coi đó là traffic giả mạo và chặn lại.

Có ba cách xử lý:

| Cách | Ưu điểm | Nhược điểm |
|---|---|---|
| Thêm **allowed-address-pairs** cho đúng dải VIP | Vẫn giữ port security cho phần còn lại | Phải cập nhật khi đổi dải VIP |
| allowed-address-pairs `0.0.0.0/0` | Đơn giản, không cần nhớ dải VIP | Gần như tắt chống giả mạo trên port |
| Tắt port security trên port | Đơn giản nhất | Không dùng được security group trên port đó |

Lab dùng cách thứ hai cho `br-host`. Nếu muốn chặt chẽ hơn, hãy chỉ khai báo đúng dải VIP như ví dụ ở phần lệnh bên dưới. Lưu ý phải thêm allowed-address-pairs cho **mọi node có thể nhận VIP**, không chỉ một node.

Với `br-overlay` và `br-ex`, lab tắt hẳn port security ở mức network:

- trên `br-ex`, các VM của OpenStack bên trong sẽ gửi traffic với IP và MAC của chính chúng, mà cloud bên ngoài không hề biết tới;
- trên `br-overlay` chỉ có traffic tunnel giữa các node, không cần lọc.

### Dành riêng dải IP cho VIP

MetalLB lấy VIP từ chính subnet của `br-host`. Nếu dải VIP nằm trong allocation pool của DHCP, sẽ có lúc Neutron cấp trùng một địa chỉ VIP cho máy mới tạo, và hai thiết bị cùng giữ một IP. Hãy thu hẹp allocation pool ngay khi tạo subnet:

| Dải | Mục đích |
|---|---|
| 10.0.0.10 – 10.0.0.199 | DHCP cấp cho các VM |
| 10.0.0.200/32 | VIP của Envoy Gateway (mọi API) |
| 10.0.0.208/28 | VIP cho MariaDB, RabbitMQ |

### MTU

Cả ba network trong lab đều dùng MTU 1500. Tunnel Geneve thêm header cho mỗi gói tin, nên kube-ovn tự giảm MTU của pod network xuống **1400** (bạn sẽ thấy giá trị này trên interface `ovn0`). Với lab, như vậy là đủ. Ở production, nên bật jumbo frame (MTU 9000) trên mạng overlay để pod và VM vẫn dùng được MTU 1500 đầy đủ.

## Ổ đĩa

![Bố trí ổ đĩa trên controller: root, ephemeral, swap từ flavor; Longhorn và Cinder LVM là volume gắn thêm](/assets/img/genestack/02-disks.svg)

Flavor của lab cung cấp sẵn ba ổ: root 160 GB, ephemeral 128 GB và swap 8 GB. Ngoài ra, mỗi controller được gắn thêm hai volume:

- **Longhorn** (50 GB): nơi lưu persistent volume cho các pod có state như MariaDB, RabbitMQ, image của Glance.
- **Cinder LVM** (100 GB): volume group `cinder-volumes`, nơi Cinder tạo volume cho VM của tenant (Phần 6).

Nên dùng volume riêng thay vì ổ ephemeral cho hai mục đích trên. Ổ ephemeral gắn liền với vòng đời của VM, nên có thể mất khi resize hoặc migrate. Swap sẽ được Kubespray tắt khi cài Kubernetes.

## Tạo hạ tầng bằng OpenStack CLI

Các lệnh dưới đây chạy trên máy có `openstack` CLI trỏ tới cloud bên dưới. Tên flavor, image và external network tuỳ từng nhà cung cấp, nên được để ở dạng `<...>`.

### Network, subnet và router

```bash
# br-host: mạng quản trị, giữ port security, chừa dải cho VIP
openstack network create br-host
openstack subnet create br-host-subnet --network br-host \
  --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 \
  --allocation-pool start=10.0.0.10,end=10.0.0.199 \
  --dns-nameserver <dns-server>

# br-overlay: tunnel Geneve, tắt port security, không cần gateway
openstack network create br-overlay --disable-port-security
openstack subnet create br-overlay-subnet --network br-overlay \
  --subnet-range 192.168.1.0/24 --gateway none

# br-ex: provider network cho OpenStack bên trong
openstack network create br-ex --disable-port-security
openstack subnet create br-ex-subnet --network br-ex \
  --subnet-range 192.168.10.0/24 --gateway none --no-dhcp

# Router để br-host ra Internet (cho apt, pull image) và gắn floating IP
openstack router create router-1
openstack router set router-1 --external-gateway <external-network>
openstack router add subnet router-1 br-host-subnet
```

Đặt `--gateway none` cho `br-overlay` và `br-ex` giúp node chỉ có **một** default route, qua `br-host`.

### Security group

```bash
openstack security group create genestack-nodes
# Cho phép mọi traffic giữa các node trong cùng group
openstack security group rule create genestack-nodes --ingress --protocol any --remote-group genestack-nodes
# SSH và ICMP từ ngoài vào (nên giới hạn về IP của bạn)
openstack security group rule create genestack-nodes --ingress --protocol tcp --dst-port 22 --remote-ip <your-ip>/32
openstack security group rule create genestack-nodes --ingress --protocol icmp
```

Security group này chỉ áp lên port `br-host`, vì hai network còn lại đã tắt port security.

### Port và máy ảo

Tạo port trước rồi mới tạo máy là cách dễ kiểm soát nhất: mỗi port có cấu hình bảo mật riêng, và thứ tự port quyết định tên interface trên máy (`enp3s0`, `enp10s0`, `enp11s0`).

```bash
FLAVOR=<flavor-8vcpu-16gb>
IMAGE="Ubuntu 24.04"
KEY=<keypair>

for n in controller-1 controller-2 controller-3 compute-1 compute-2; do
  openstack port create "$n-host" --network br-host --security-group genestack-nodes \
    --allowed-address ip-address=10.0.0.200/32 \
    --allowed-address ip-address=10.0.0.208/28
  openstack port create "$n-overlay" --network br-overlay --disable-port-security
  openstack port create "$n-ex" --network br-ex --disable-port-security

  openstack server create "openstack-$n" --flavor "$FLAVOR" --image "$IMAGE" --key-name "$KEY" \
    --port "$n-host" --port "$n-overlay" --port "$n-ex"
done

# Deployer: chỉ cần br-host, cộng floating IP
openstack port create deployer-host --network br-host --security-group genestack-nodes
openstack server create deployer --flavor <flavor-2vcpu-6gb> --image "$IMAGE" --key-name "$KEY" \
  --port deployer-host
openstack floating ip create <external-network>
openstack server add floating ip deployer <floating-ip>
```

### Volume cho Longhorn và Cinder

```bash
for i in 1 2 3; do
  openstack volume create --size 50  "longhorn-$i"
  openstack volume create --size 100 "cinder-lvm-controller-$i"
  openstack server add volume "openstack-controller-$i" "longhorn-$i"
  openstack server add volume "openstack-controller-$i" "cinder-lvm-controller-$i"
done
```

Không format hai volume này. Longhorn và LVM sẽ được cấu hình ở các bài sau.

## Chuẩn bị deployer

SSH vào deployer qua floating IP, rồi:

**1. Tạo SSH key và phân phối tới các node.** Ansible (dùng cho Kubespray và các playbook của Genestack) cần SSH được từ deployer tới mọi node:

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
for ip in 10.0.0.243 10.0.0.110 10.0.0.174 10.0.0.154 10.0.0.247; do
  ssh-copy-id root@$ip
done
```

Nếu image không cho đăng nhập root bằng mật khẩu, hãy đưa public key của deployer vào cloud-init `user-data` lúc tạo máy.

Việc lấy mã nguồn Genestack, checkout đúng release và chạy `bootstrap.sh` trên deployer sẽ được làm ở đầu Phần 3 (Bước 0), ngay trước khi cài Kubernetes.

## Checklist trước khi sang Phần 3

Chạy từ deployer, hoặc trên từng node:

```bash
# 1. SSH không cần mật khẩu tới mọi node
for ip in 10.0.0.243 10.0.0.110 10.0.0.174 10.0.0.154 10.0.0.247; do ssh -o BatchMode=yes root@$ip hostname; done

# 2. Các node thấy nhau trên mạng overlay
ping -c2 192.168.1.100

# 3. MTU: gói 1472 byte + 28 byte header = 1500, phải đi qua được khi cấm phân mảnh
ping -c2 -M do -s 1472 192.168.1.100

# 4. Đồng bộ thời gian (quan trọng cho etcd và Galera)
timedatectl show -p NTPSynchronized        # NTPSynchronized=yes

# 5. enp11s0 đã UP nhưng không có IP; chỉ có một default route qua enp3s0
ip -br addr show enp11s0
ip route | grep default

# 6. Volume đã gắn và chưa format
lsblk
```

| Hạng mục | Kết quả mong đợi |
|---|---|
| SSH | In ra hostname của từng node, không hỏi mật khẩu |
| Overlay | Ping được giữa mọi cặp node |
| MTU | Không có lỗi `Message too long` |
| Thời gian | `NTPSynchronized=yes` trên mọi node |
| Mạng provider | `enp11s0` ở trạng thái `UP`, không có địa chỉ IP |
| Ổ đĩa | Có hai ổ mới, chưa có filesystem |

## Tóm tắt

- Lab gồm 1 deployer, 3 controller, 2 compute. Controller nên có 32 GB RAM nếu muốn dư dả.
- Không có nested virtualization thì Nova chạy `virt_type = qemu`: đủ để học, không đủ để đo hiệu năng.
- Ba network tách biệt: quản trị (`br-host`), tunnel (`br-overlay`), provider (`br-ex`, không gán IP trên node).
- Trên cloud OpenStack, **phải** cho phép VIP của MetalLB qua port security bằng allowed-address-pairs, và chừa dải VIP ra khỏi DHCP.
- Dùng volume riêng cho Longhorn và Cinder LVM, không dùng ổ ephemeral.
- Deployer phải SSH được tới mọi node bằng key, không cần mật khẩu.

## Tiếp theo

Ở **Phần 3**, chúng ta sẽ khai báo inventory trong `/etc/genestack/inventory`, gắn label vai trò cho từng node, và dùng Kubespray để dựng cluster Kubernetes với kube-ovn làm CNI.
