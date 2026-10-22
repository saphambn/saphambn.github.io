---
title: "OpenStack trên Kubernetes với Genestack – Phần 3: Dựng Kubernetes bằng Kubespray và kube-ovn"
date: 2026-10-21 08:03:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, kubespray, kube-ovn, cni]
render_with_liquid: false
---

Ở [Phần 2](/posts/genestack-02-lab-infrastructure/), chúng ta đã có 6 máy ảo, ba network, và một deployer SSH được tới mọi node. Bài này chuẩn bị bộ công cụ Genestack trên deployer, rồi dựng lớp tiếp theo trong kiến trúc: **cluster Kubernetes**, nền móng cho mọi thành phần còn lại của Genestack.

Genestack dùng **Kubespray** (đi kèm dưới dạng git submodule) để cài Kubernetes. Điểm khác so với cách dùng Kubespray thông thường: Kubespray chỉ cài Kubernetes **không kèm CNI**. CNI **kube-ovn** được cài riêng bằng Helm ở bước sau, vì nó vừa là mạng của Kubernetes, vừa là OVN cho Neutron, nên cần được cấu hình cẩn thận hơn.

## Luồng tổng quan

![Luồng dựng Kubernetes: chuẩn bị deployer, inventory, hostname, host-setup, Kubespray, label node, cài kube-ovn](/assets/img/genestack/03-install-flow.svg)

Tất cả các bước đều chạy từ deployer. Như hình cho thấy, sau khi Kubespray chạy xong, các node sẽ ở trạng thái `NotReady`. Đây là điều bình thường, vì cluster chưa có CNI. Node chỉ chuyển sang `Ready` sau khi cài kube-ovn ở bước 6.

Trước sáu bước này là **Bước 0**: đưa mã nguồn Genestack về deployer và chạy bootstrap.

## Bước 0 – Lấy Genestack và chạy bootstrap trên deployer

### Clone repo

Genestack dùng git submodule cho một số thành phần, trong đó có Kubespray (`submodules/kubespray`). Vì vậy phải clone kèm `--recurse-submodules`. Theo quy ước của tài liệu Genestack, mã nguồn nằm ở `/opt/genestack`:

```bash
git clone --recurse-submodules -j4 https://github.com/rackerlabs/genestack /opt/genestack
cd /opt/genestack
```

### Checkout đúng bản release

Nhánh mặc định của repo luôn thay đổi. Để môi trường ổn định và có thể tái tạo, hãy chọn một bản release cố định. Release của Genestack được đánh dấu bằng tag dạng `release-YYYY.N.x`:

```bash
git fetch --tags
git tag --sort=-creatordate | grep '^release-' | head
```

```
release-2026.3.0
release-2026.2.0.2
release-2026.2.0.1
release-2026.2.0
release-2026.1.0
...
```

Checkout tag đã chọn (lab dùng `release-2026.2.0.2`), rồi **cập nhật submodule cho khớp với release đó**:

```bash
git checkout release-2026.2.0.2
git submodule update --init --recursive
```

Đừng bỏ qua lệnh thứ hai. `git checkout` chỉ đổi mã nguồn của repo chính. Submodule vẫn ở commit cũ cho đến khi bạn chạy `git submodule update`, và khi đó Kubespray có thể không khớp với cấu hình của release đã chọn.

Kiểm tra lại:

```bash
git describe --tags          # release-2026.2.0.2
git submodule status         # các submodule không có dấu '+' ở đầu dòng
```

Dấu `+` ở đầu dòng trong `git submodule status` nghĩa là submodule đang ở commit khác với commit mà release yêu cầu.

### Chạy bootstrap

Luôn chạy bootstrap **sau khi** đã checkout đúng release. Script dựng cấu trúc `/etc/genestack` dựa trên nội dung của bản đang checkout:

```bash
/opt/genestack/bootstrap.sh
```

Nếu chạy bằng `sudo`, hãy dùng `sudo -E` để giữ biến môi trường. Mặc định script chờ cloud-init chạy xong trước khi làm việc. Có thể bỏ qua bước chờ này bằng `--no-wait-cloud-init`.

`bootstrap.sh` làm các việc sau:

| Hạng mục | Chi tiết |
|---|---|
| Gói hệ thống | Cài các gói cần thiết. Log nằm ở `~/genestack-base-package-install.log` |
| Công cụ | Cài `helm`, `yq`, `cmctl` |
| Python virtualenv | Tạo `~/.venvs/genestack`, cài Ansible theo `requirements.txt` và các Ansible collection |
| `/etc/genestack/provider` | Ghi provider Kubernetes, mặc định là `kubespray` |
| `/etc/genestack/inventory/` | Chép inventory mẫu (chỉ khi thư mục còn trống) |
| `/etc/genestack/kustomize/` | Tạo symlink tới `base-kustomize` cho từng service, và tạo `overlay/kustomization.yaml` mặc định |
| `/etc/genestack/helm-configs/` | Tạo thư mục override rỗng cho từng service, cùng `global_overrides/` |
| `/etc/genestack/manifests/`, `gateway-api/` | Chép manifest mẫu |

Script được viết để chạy lại an toàn: những gì đã tồn tại sẽ được bỏ qua. Vì thế, khi nâng cấp Genestack lên release mới, hãy chạy lại `bootstrap.sh` sau khi checkout, để tạo symlink và thư mục cho các service mới xuất hiện trong release đó.

### Cài kubectl

`bootstrap.sh` không cài `kubectl`. Hãy cài bản trùng version với Kubernetes sẽ triển khai:

```bash
KVER=v1.35.4
curl -LO "https://dl.k8s.io/release/${KVER}/bin/linux/amd64/kubectl"
install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
kubectl version --client
```

### Nạp môi trường

Trong mỗi phiên làm việc mới trên deployer, hãy nạp môi trường trước khi chạy bất kỳ lệnh nào của Genestack:

```bash
source /opt/genestack/scripts/genestack.rc
```

File `genestack.rc` kích hoạt virtualenv `~/.venvs/genestack`, trỏ `ANSIBLE_INVENTORY` tới `/etc/genestack/inventory`, và đặt sẵn một số biến cho Ansible như số fork và tắt kiểm tra host key. Có thể thêm dòng trên vào `~/.bashrc` cho tiện.

Kiểm tra nhanh:

```bash
ansible --version | head -1
helm version --short
ls /etc/genestack     # gateway-api  helm-configs  inventory  kustomize  manifests  provider ...
```

## Yêu cầu với node

Tài liệu Genestack nêu vài yêu cầu tối thiểu. Ubuntu 24.04 đáp ứng sẵn phần lớn trong số đó:

| Yêu cầu | Ghi chú | Lab |
|---|---|---|
| Ít nhất 2 interface mạng | Một cho quản trị, một cho tunnel OVN | 3 interface |
| Kernel ≥ 5.17, khuyến nghị ≥ 6.2 | Cần cho các tính năng của OVS/OVN | 6.8 |
| Module `br_netfilter` | Kubespray đặt các sysctl liên quan tới bridge | `host-setup.yml` sẽ nạp |
| `/tmp` không mount `noexec` | Kubespray dùng `/tmp` để tải và chạy file | Mặc định của Ubuntu là ổn |

## Bước 1 – Inventory

Inventory nằm ở `/etc/genestack/inventory/inventory.yaml`. Genestack có sẵn file mẫu `inventory.yaml.example` ngay cạnh đó. Inventory của lab như sau (đã lược bớt các nhóm dành cho các bài sau):

```yaml
all:
  hosts:
    openstack-controller-1: {ansible_host: 10.0.0.243, ip: 10.0.0.243, access_ip: 10.0.0.243}
    openstack-controller-2: {ansible_host: 10.0.0.110, ip: 10.0.0.110, access_ip: 10.0.0.110}
    openstack-controller-3: {ansible_host: 10.0.0.174, ip: 10.0.0.174, access_ip: 10.0.0.174}
    openstack-compute-1:    {ansible_host: 10.0.0.154, ip: 10.0.0.154, access_ip: 10.0.0.154}
    openstack-compute-2:    {ansible_host: 10.0.0.247, ip: 10.0.0.247, access_ip: 10.0.0.247}
  children:
    k8s_cluster:
      vars:
        cluster_name: cluster.local
        kube_ovn_iface: enp10s0
        kube_ovn_default_interface_name: enp10s0
        kube_ovn_central_hosts: "{{ groups['ovn_network_nodes'] }}"
        kube_service_addresses: 10.233.0.0/18
        kube_pods_subnet: 10.236.0.0/14
      children:
        kube_control_plane:
          hosts: {openstack-controller-1: null, openstack-controller-2: null, openstack-controller-3: null}
        etcd:
          hosts: {openstack-controller-1: null, openstack-controller-2: null, openstack-controller-3: null}
        kube_node:
          hosts:
            openstack-controller-1: null
            openstack-controller-2: null
            openstack-controller-3: null
            openstack-compute-1: null
            openstack-compute-2: null
        openstack_control_plane:
          hosts: {openstack-controller-1: null, openstack-controller-2: null, openstack-controller-3: null}
        ovn_network_nodes:
          hosts: {openstack-controller-1: null, openstack-controller-2: null, openstack-controller-3: null}
        openstack_compute_nodes:
          hosts: {openstack-compute-1: null, openstack-compute-2: null}
```

Một vài điểm cần chú ý:

- **`ip` và `access_ip`** là địa chỉ trên `br-host`. Kubernetes dùng chúng làm địa chỉ của node (`InternalIP`), nên phải là IP thuộc mạng quản trị, không phải IP overlay.
- **`kube_control_plane` và `etcd`** gồm 3 node để etcd có quorum và API server có dự phòng.
- **`kube_node`** gồm cả 5 node, tức là controller cũng chạy workload. Đây là điều bắt buộc trong Genestack, vì toàn bộ control plane của OpenStack chạy dưới dạng pod trên controller.
- **Các nhóm như `openstack_control_plane`, `ovn_network_nodes`, `openstack_compute_nodes`** không phải của Kubespray mà dành cho các playbook của Genestack. Biến `kube_ovn_*` trỏ kube-ovn vào interface `enp10s0` của mạng overlay.

### Các biến quan trọng trong group_vars

Bên cạnh inventory, thư mục `/etc/genestack/inventory/group_vars/` chứa cấu hình chi tiết cho Kubespray. Những giá trị dưới đây là mặc định của Genestack. Nên đọc qua để hiểu, nhưng thường không cần sửa:

| Biến | Giá trị | Ý nghĩa |
|---|---|---|
| `kube_version` | `1.35.4` | Version Kubernetes |
| `kube_network_plugin` | `none` | Kubespray **không** cài CNI. kube-ovn sẽ được cài sau |
| `kube_proxy_mode` | `ipvs` | kube-proxy dùng IPVS thay vì iptables |
| `kube_proxy_strict_arp` | `true` | Bắt buộc khi dùng MetalLB ở chế độ L2 |
| `kube_pods_subnet` | `10.236.0.0/14` | Dải IP của pod |
| `kube_service_addresses` | `10.233.0.0/18` | Dải ClusterIP của Service |
| `dns_mode` / `enable_nodelocaldns` | `coredns` / `true` | Có cache DNS trên từng node, tại `169.254.25.10` |
| `container_manager` | `containerd` | Container runtime |
| `etcd_deployment_type` | `host` | etcd chạy như systemd service trên controller, không chạy dạng pod |

Hai dải `kube_pods_subnet` và `kube_service_addresses` được khai báo lại trong `group_vars/k8s_cluster/zz-genestack-cli.yml`. Tên file bắt đầu bằng `zz-` để được nạp sau cùng và luôn thắng. Nếu muốn đổi dải IP, hãy sửa ở cả file này và ở cấu hình kube-ovn (bước 6). Hai bên phải khớp nhau.

**Về IPVS:** ở chế độ này, kernel giữ một bảng kết nối với timeout riêng cho kết nối TCP rảnh, mặc định là 900 giây. Đây là con số cần nhớ khi cấu hình connection pool cho database (đã nói ở [Phần 1](/posts/genestack-01-architecture/)).

## Bước 2 – Hostname

Kubernetes dùng hostname làm tên node, và Ansible dùng tên trong inventory. Hai giá trị này nên trùng nhau. Genestack có sẵn lệnh để đồng bộ:

```bash
ansible -m shell -a 'hostnamectl set-hostname {{ inventory_hostname }}' --become all
ansible -m shell -a "grep 127.0.0.1 /etc/hosts | grep -q {{ inventory_hostname }} || sed -i 's/^127.0.0.1.*/127.0.0.1 {{ inventory_hostname }} localhost.localdomain localhost/' /etc/hosts" --become all
```

Tài liệu Genestack khuyến nghị dùng **FQDN** làm tên node, với hậu tố trùng `cluster_name` (ví dụ `controller-1.cluster.local`), khai báo qua biến `kube_override_hostname`. Lab dùng tên ngắn như `openstack-controller-1` cho dễ đọc. Dù chọn cách nào, hãy **quyết định trước khi cài**, vì đổi tên node sau khi cluster đã chạy rất phiền phức.

## Bước 3 – Chuẩn bị host

Playbook `host-setup.yml` đưa các node về trạng thái mà Kubernetes và OpenStack cần:

```bash
cd /opt/genestack/ansible/playbooks
ansible-playbook host-setup.yml
```

Những việc chính playbook này làm:

- cài TuneD và áp profile tối ưu hiệu năng;
- kiểm tra version kernel, nạp các module cần thiết và cấu hình để chúng được nạp lại khi khởi động;
- áp các sysctl tuning, tắt ICMP redirect;
- trên compute node: chuẩn bị cho KVM (nhóm `kvm`, udev rule), cài `open-iscsi` và multipath, đặt iSCSI initiator name;
- cấu hình `lvm.conf` và sysstat.

Playbook đọc một số giá trị từ `/etc/genestack/inventory/group_vars/all/all.yml`. Hãy xem qua file này trước khi chạy.

## Bước 4 – Chạy Kubespray

Kubespray nằm trong submodule của Genestack (lab dùng Kubespray v2.31.0). Chạy playbook từ thư mục đó:

```bash
cd /opt/genestack/submodules/kubespray
ansible-playbook cluster.yml --become
```

Playbook có thể mất vài chục phút, phần lớn thời gian dành cho việc tải binary và container image về các node.

### Lấy kubeconfig về deployer

Kubespray tạo file kubeconfig quản trị tại `/etc/kubernetes/admin.conf` trên các controller. Chép nó về deployer:

```bash
mkdir -p ~/.kube
scp root@10.0.0.243:/etc/kubernetes/admin.conf ~/.kube/config
kubectl get nodes
```

File kubeconfig này trỏ thẳng tới API server của controller-1 (`https://10.0.0.243:6443`). Nếu controller-1 bị tắt, `kubectl` trên deployer sẽ không kết nối được, dù cluster vẫn chạy bình thường. Khi đó chỉ cần sửa trường `server` sang IP của một controller khác.

Lúc này `kubectl get nodes` sẽ cho thấy cả 5 node ở trạng thái **`NotReady`**, vì chưa có CNI. Hãy tiếp tục sang bước 5.

## Bước 5 – Gắn label cho node

Genestack quyết định thành phần nào chạy ở đâu dựa hoàn toàn vào **label** trên node. Các label cần thiết:

| Label | Gắn cho | Dùng để |
|---|---|---|
| `openstack-control-plane=enabled` | controller | API, scheduler, database, message bus của OpenStack |
| `openstack-network-node=enabled` | controller **và** compute | Thành phần mạng của Neutron/OVN |
| `openstack-compute-node=enabled` | compute | `nova-compute`, libvirt |
| `openstack-storage-node=enabled` | node chứa storage | Thành phần storage của OpenStack |
| `longhorn.io/storage-node=enabled` | node chứa dữ liệu Longhorn | Longhorn chỉ chạy và lưu replica ở đây |
| `node-role.kubernetes.io/worker=worker` | node chạy workload | Một số dịch vụ hạ tầng chọn node theo label này |
| `kube-ovn/role=master` | controller | Nơi chạy OVN central (NB/SB database) |
| `ovn.kubernetes.io/ovs_dp_type=kernel` | mọi node | Dùng datapath OVS trong kernel (không dùng DPDK) |

Compute node cần nhãn `openstack-network-node` vì với OVN, mỗi compute đều tự xử lý mạng cho VM của nó: `ovn-controller` và metadata agent phải chạy trên mọi node có VM.

Label `openstack-network-node` **không** quyết định node có làm gateway hay không. Vai trò gateway (nơi đặt cổng ra mạng ngoài của router Neutron, xử lý SNAT) do một cấu hình riêng quyết định: annotation `ovn.openstack.org/gateway=enabled` trên node. Khi cấu hình OVN cho Neutron (Phần 5), annotation này được chuyển thành `ovn-cms-options=enable-chassis-as-gw` của chassis trong OVN. Có hai mô hình phổ biến:

- **Mọi node đều là gateway**, kể cả compute. Lab dùng mô hình này: cả 5 node đều bật `enable-chassis-as-gw`. Traffic ra ngoài được phân tán, nhưng compute phải nối được vào mạng provider (`br-ex`).
- **Gateway tập trung** trên một nhóm node riêng (network node hoặc controller). Mô hình này dễ kiểm soát traffic ra ngoài hơn, đổi lại các node gateway có thể thành điểm nghẽn.

Cần ít nhất một node gateway trong cluster.

Lệnh gắn label cho lab:

```bash
CTRL="openstack-controller-1 openstack-controller-2 openstack-controller-3"
COMP="openstack-compute-1 openstack-compute-2"

kubectl label node $CTRL openstack-control-plane=enabled openstack-network-node=enabled \
  openstack-storage-node=enabled longhorn.io/storage-node=enabled node-role.kubernetes.io/worker=worker
kubectl label node $COMP openstack-compute-node=enabled openstack-network-node=enabled

# Nhãn cho kube-ovn
kubectl label node -l node-role.kubernetes.io/control-plane kube-ovn/role=master
kubectl label node -l ovn.kubernetes.io/ovs_dp_type!=userspace ovn.kubernetes.io/ovs_dp_type=kernel
```

Kiểm tra lại:

```bash
kubectl get nodes -L openstack-control-plane,openstack-compute-node,openstack-network-node,longhorn.io/storage-node,kube-ovn/role
```

**Mẹo:** một số label có giá trị rỗng. Ví dụ, Kubespray tự gắn `node.kubernetes.io/exclude-from-external-load-balancers` cho các node control-plane. Với cách xem `-L <key>` ở trên, label rỗng hiển thị y như node không có label. Để biết chắc node nào có một label, hãy dùng `kubectl get nodes -l <key>`. Riêng label này sẽ ảnh hưởng tới MetalLB, và chúng ta sẽ xử lý nó ở Phần 4a.

## Bước 6 – Cài kube-ovn

### Quy hoạch subnet trước khi cài

kube-ovn làm việc với nhiều dải IP cùng lúc. Một số dải là thật (gắn lên interface), một số chỉ là ảo (dùng cho định tuyến và load balancing). Tất cả phải được chọn **trước khi cài**: đổi dải pod hay join sau khi cluster đã chạy là một thao tác gây gián đoạn, cần dựng lại pod (Genestack có hướng dẫn riêng trong `docs/infrastructure-kube-ovn-re-ip.md`).

Các dải trong lab:

| Dải | CIDR | Khai báo ở | Dùng cho |
|---|---|---|---|
| Mạng host | 10.0.0.0/24 | Cloud bên dưới | IP của node (`br-host`), cũng là nơi MetalLB lấy VIP |
| Mạng overlay | 192.168.1.0/24 | Cloud bên dưới | Tunnel Geneve giữa các node (`IFACE`) |
| Mạng provider | 192.168.10.0/24 | Cloud bên dưới | Mạng external của Neutron |
| **Pod** – subnet `ovn-default` | 10.236.0.0/14 | `kube_pods_subnet` (Kubespray) và `ipv4.POD_CIDR` (kube-ovn) | IP của mọi pod. Gateway `10.236.0.1` được loại khỏi dải cấp phát |
| **Service** | 10.233.0.0/18 | `kube_service_addresses` (Kubespray) và `ipv4.SVC_CIDR` (kube-ovn) | ClusterIP, chỉ là địa chỉ ảo |
| **Join** – subnet `join` | 100.64.0.0/16 | Mặc định của chart kube-ovn | Nối node với pod network. Mỗi node nhận một IP trên interface `ovn0` |

Các nguyên tắc cần giữ:

1. **Không chồng lấn.** Sáu dải trên phải hoàn toàn tách biệt nhau, và không trùng với bất kỳ mạng nào mà node hay người dùng cần truy cập: mạng văn phòng, VPN, DNS server, registry nội bộ… Nếu một dải pod trùng với mạng của DNS server, chẳng hạn, pod sẽ định tuyến nhầm vào bên trong cluster và không bao giờ tới được DNS server đó.
2. **Khớp giữa Kubespray và kube-ovn.** Dải pod và service được khai báo ở hai nơi (bảng trên). Hai bên lệch nhau thì kube-proxy và kube-ovn sẽ hiểu cluster theo hai cách khác nhau.
3. **Cẩn thận với dải join `100.64.0.0/16`.** Dải này nằm trong `100.64.0.0/10`, dải dành cho CGNAT. Một số nhà mạng, một số giải pháp VPN như Tailscale, và một số hạ tầng cloud cũng dùng dải này. Nếu môi trường của bạn có, hãy đổi dải join trước khi cài.
4. **Đủ rộng.** `/14` cho pod (khoảng 262 nghìn địa chỉ) là rất dư cho lab. Với production, hãy tính theo số node nhân số pod tối đa trên mỗi node, cộng thêm dư địa để mở rộng.

Trên mỗi node, kube-ovn tạo route tới dải pod qua gateway của dải join:

```
$ ip route | grep ovn0
10.236.0.0/14 via 100.64.0.1 dev ovn0 proto static src 10.0.0.243
100.64.0.0/16 dev ovn0 proto kernel scope link src 100.64.0.3
```

Subnet `ovn-default` bật `natOutgoing`: khi pod truy cập ra ngoài cluster, traffic được SNAT thành IP của node. Vì vậy hạ tầng bên ngoài không cần biết route tới dải pod.

**Còn mạng của OpenStack thì sao?** Neutron dùng chung hệ OVN với kube-ovn. Mỗi router Neutron xuất hiện trong kube-ovn như một **VPC** riêng, tách biệt với VPC `ovn-cluster` của Kubernetes:

```
$ kubectl get vpc
NAME                                           SUBNETS
neutron-e5b86752-07ac-437f-a42d-9238b1c47bec   ["neutron-1c8a637c-…","neutron-024aa300-…"]
ovn-cluster                                    ["join","ovn-default"]
```

Nhờ tách VPC, về mặt kỹ thuật dải IP của tenant có thể trùng với dải pod. Dù vậy, nên tránh đặt mạng tenant trùng với các dải trong bảng trên, để việc đọc route và gỡ lỗi về sau dễ hơn.

Một chi tiết nhỏ: base config của Genestack đặt `PINGER_EXTERNAL_ADDRESS` là `208.67.222.222` (OpenDNS). `kube-ovn-pinger` dùng địa chỉ này để kiểm tra kết nối ra Internet từ mọi node. Trong môi trường không có Internet, hãy đổi sang một địa chỉ nội bộ luôn trả lời ping, để tránh báo động giả.

### Cấu hình

Giống mọi thành phần cài bằng Helm trong Genestack, kube-ovn có base config trong `/opt/genestack/base-helm-configs/kube-ovn/` và override của bạn trong `/etc/genestack/helm-configs/kube-ovn/`. Base config mặc định dùng một bridge tên `br-overlay` trên host. Lab nối thẳng interface `enp10s0`, nên override như sau:

```yaml
# /etc/genestack/helm-configs/kube-ovn/kube-ovn-helm-overrides.yaml
---
global:
  images:
    kubeovn:
      tag: v1.15.4
networking:
  ENABLE_SSL: false
  IFACE: "enp10s0"           # interface mang tunnel Geneve
  vlan:
    VLAN_INTERFACE_NAME: ""  # lab không dùng VLAN provider qua kube-ovn
```

`IFACE` là giá trị quan trọng nhất: interface mà OVS dùng để tạo tunnel giữa các node. Hai dải `POD_CIDR` và `SVC_CIDR` trong base config (`10.236.0.0/14`, `10.233.0.0/18`) phải trùng với dải đã khai báo cho Kubespray ở bước 1.

### Cài đặt

```bash
/opt/genestack/bin/install-kube-ovn.sh
```

Script này:

1. đọc version chart từ `/etc/genestack/helm-chart-versions.yaml`;
2. tìm các node có label `kube-ovn/role=master` (dừng lại báo lỗi nếu không có node nào), để lấy danh sách IP cho OVN central;
3. tạo secret TLS `kube-ovn-tls` nếu chưa có;
4. chạy `helm upgrade --install` với base config rồi tới override.

Vài phút sau, các pod kube-ovn chạy lên và node chuyển sang `Ready`.

## Kết quả: cluster trông như thế nào

![Thành phần Kubernetes trên từng node: controller chạy etcd, API server, OVN central; mọi node chạy kubelet, kube-proxy, ovs-ovn, kube-ovn-cni, nodelocaldns](/assets/img/genestack/03-cluster-layout.svg)

- **Trên 3 controller:** etcd (systemd), các static pod của control plane (`kube-apiserver`, `kube-scheduler`, `kube-controller-manager`), `ovn-central` (database NB/SB của OVN) và `kube-ovn-controller` (cấp IP, quản lý subnet).
- **Trên 2 compute:** Kubespray đặt một pod `nginx-proxy` làm load balancer cục bộ. Kubelet trên compute gọi API server qua `localhost`, và `nginx-proxy` chuyển tiếp tới cả 3 controller, nên mất một controller không làm compute mất kết nối.
- **Trên mọi node:** kubelet và containerd, `kube-proxy` (IPVS), `ovs-ovn` (Open vSwitch), `kube-ovn-cni`, và `nodelocaldns`.

kube-ovn tạo hai subnet mặc định:

```
$ kubectl get subnets.kubeovn.io
NAME          PROVIDER   VPC           PROTOCOL   CIDR            ...   GATEWAYTYPE
join          ovn        ovn-cluster   IPv4       100.64.0.0/16   ...
ovn-default   ovn        ovn-cluster   IPv4       10.236.0.0/14   ...   distributed
```

- **`ovn-default`** là dải IP cho pod.
- **`join`** là dải nối giữa node và pod network. Interface `ovn0` trên mỗi node nhận một IP trong dải `100.64.0.0/16` này, và MTU của nó là 1400 để chừa chỗ cho header Geneve.

Mặc định kube-ovn bật `enable-lb`: traffic từ pod tới ClusterIP được xử lý bởi load balancer của OVN, còn traffic từ tiến trình chạy trên host network thì đi qua kube-proxy IPVS. Cả hai cùng tồn tại trong cluster.

## Checklist

```bash
git -C /opt/genestack describe --tags
kubectl get nodes -o wide
kubectl get subnets.kubeovn.io
kubectl -n kube-system get pods -o wide | grep -E 'ovn|ovs'
kubectl get nodes -l kube-ovn/role=master
```

| Kiểm tra | Kết quả mong đợi |
|---|---|
| Release Genestack | Đúng tag đã chọn, ví dụ `release-2026.2.0.2` |
| Node | 5 node `Ready`, version `v1.35.4`, `INTERNAL-IP` thuộc `10.0.0.0/24` |
| Subnet | `join` (100.64.0.0/16) và `ovn-default` (10.236.0.0/14) |
| OVN central | 3 pod `ovn-central`, mỗi controller một pod |
| Data plane | `ovs-ovn` và `kube-ovn-cni` chạy trên cả 5 node |
| Label | Đủ label như bảng ở bước 5 |

Output mẫu của lab:

```
NAME                     STATUS   ROLES                  VERSION   INTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
openstack-compute-1      Ready    <none>                 v1.35.4   10.0.0.154    Ubuntu 24.04.5 LTS   6.8.0-142-generic  containerd://2.2.3
openstack-compute-2      Ready    <none>                 v1.35.4   10.0.0.247    Ubuntu 24.04.5 LTS   6.8.0-142-generic  containerd://2.2.3
openstack-controller-1   Ready    control-plane,worker   v1.35.4   10.0.0.243    Ubuntu 24.04.5 LTS   6.8.0-142-generic  containerd://2.2.3
openstack-controller-2   Ready    control-plane,worker   v1.35.4   10.0.0.110    Ubuntu 24.04.5 LTS   6.8.0-142-generic  containerd://2.2.3
openstack-controller-3   Ready    control-plane,worker   v1.35.4   10.0.0.174    Ubuntu 24.04.5 LTS   6.8.0-142-generic  containerd://2.2.3
```

## Nâng cấp Kubernetes về sau

Kubespray nâng cấp Kubernetes theo từng nhóm node. Quy trình dùng trong lab, tham khảo thêm `docs/k8s-kubespray-upgrade.md` của Genestack:

```bash
cd /opt/genestack/submodules/kubespray
V=1.35.4   # version đích

# Thu thập facts mới cho mọi node
ansible-playbook playbooks/facts.yml --become

# Nâng cấp control plane và etcd trước
ansible-playbook cluster.yml -e upgrade_cluster_setup=true -e kube_version=$V --become \
  --limit "kube_control_plane:etcd"

# Sau đó tới các node còn lại
ansible-playbook cluster.yml -e upgrade_cluster_setup=true -e kube_version=$V --become \
  --limit 'k8s_cluster:!kube_control_plane:!etcd'
```

Nhớ cập nhật `kube_version` trong `group_vars/k8s_cluster/k8s-cluster.yml` để lần chạy Kubespray tiếp theo không đưa cluster về version cũ.

## Tóm tắt

- Trên deployer: clone Genestack kèm submodule, checkout một tag `release-*` cố định, chạy `git submodule update`, rồi mới chạy `bootstrap.sh`. Cài thêm `kubectl` và nạp `genestack.rc` mỗi phiên.
- Genestack dùng Kubespray trong submodule, với `kube_network_plugin: none`. CNI kube-ovn được cài riêng bằng Helm.
- Inventory dùng IP quản trị cho node. Controller vừa là control plane vừa chạy workload.
- kube-proxy chạy IPVS với `strict_arp: true`, để sẵn sàng cho MetalLB.
- Node chỉ `Ready` sau khi cài kube-ovn. Label vai trò và label `kube-ovn/role=master` phải có trước khi chạy `install-kube-ovn.sh`.
- `IFACE` của kube-ovn trỏ vào interface overlay. Dải pod và service phải khớp giữa Kubespray và kube-ovn.

## Tiếp theo

Ở **Phần 4a**, chúng ta sẽ dựng lớp hạ tầng nền đầu tiên trên cluster vừa có: MetalLB, cert-manager và Envoy Gateway để đưa API ra ngoài. Phần 4b sẽ tiếp tục với Longhorn, MariaDB Galera và RabbitMQ.
