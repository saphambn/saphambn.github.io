---
title: "OpenStack trên Kubernetes với Genestack – Phần 10: Load balancer với Octavia"
date: 2026-10-26 10:00:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, octavia, load-balancer, ovn, amphora]
render_with_liquid: false
---

Đến [Phần 9](/posts/genestack-10-replace-controller/), cloud đã có compute, mạng, block storage, instance HA, và chúng ta đã biết cách thêm, thay node. Thành phần tiếp theo mà hầu hết người dùng cloud sẽ hỏi tới là **load balancer**: một VIP đứng trước nhiều VM, chia tải và tự loại VM hỏng ra khỏi vòng.

Trong OpenStack, đó là việc của **Octavia** (service type `load-balancer`). Bài này cài Octavia bằng Genestack, giải thích các thành phần chạy ở đâu và nối với nhau thế nào, rồi tạo thử hai load balancer: một bằng provider **OVN**, một bằng provider **amphora**.

## Hai provider: OVN và amphora

Octavia chỉ là API và bộ điều phối. Việc cân bằng tải thật sự do một **provider driver** đảm nhận. Genestack bật sẵn hai provider:

| | `amphora` (mặc định) | `ovn` |
|---|---|---|
| LB nằm ở đâu | Một (hoặc hai) VM chạy haproxy, gọi là **amphora** | Bản ghi `Load_Balancer` trong OVN NB, `ovn-controller` DNAT ngay trong OVS |
| Tầng | L4 và L7: HTTP, HTTPS, TERMINATED_HTTPS, L7 policy | Chỉ L4: TCP, UDP, SCTP |
| Thuật toán | ROUND_ROBIN, LEAST_CONNECTIONS, SOURCE_IP | Chỉ SOURCE_IP_PORT |
| TLS termination | Có (chứng chỉ lưu trong Barbican) | Không |
| HA | `ACTIVE_STANDBY`: 2 amphora, VRRP | Phân tán sẵn, không có điểm chết riêng |
| Tạo LB mất | Thời gian boot một VM | Vài giây |
| Tài nguyên | 1–2 VM cho mỗi LB | Không tốn VM |
| Client IP tới member | Là IP của amphora (dùng header `X-Forwarded-For` cho HTTP) | Giữ nguyên IP client |

Quy tắc chọn đơn giản: cần L7, HTTPS termination hay sticky session theo cookie thì dùng `amphora`. Chỉ cần chia TCP/UDP thì `ovn` nhẹ và nhanh hơn nhiều.

Provider `ovn` gần như không cần chuẩn bị gì. Phần lớn bài này dành cho `amphora`, vì nó kéo theo cả một hạ tầng riêng: network quản lý, chứng chỉ, image, flavor, security group.

## Kiến trúc Octavia trong Genestack

![Octavia trong Genestack: octavia-api, housekeeping, worker và health-manager trên controller; amphora trên compute; worker và health-manager nối với amphora qua lb-mgmt-net](/assets/img/genestack/11-octavia-architecture.svg)

| Thành phần | Dạng | Chạy ở đâu | Việc |
|---|---|---|---|
| `octavia-api` | Deployment, HPA 2–6 replica | Controller | API, ghi DB, đẩy việc vào RabbitMQ |
| `octavia-driver-agent` | Sidecar trong pod `octavia-api` | Controller | Nhận sự kiện từ OVN SB để cập nhật trạng thái LB của provider `ovn` |
| `octavia-worker` | DaemonSet, `hostNetwork` | Mọi controller | Tạo amphora qua Nova, cấu hình haproxy qua REST API của amphora |
| `octavia-health-manager` | DaemonSet, `hostNetwork` | Mọi controller | Nhận heartbeat từ amphora, kích hoạt failover khi amphora im lặng |
| `octavia-housekeeping` | Deployment | Controller | Dọn bản ghi cũ, xoay vòng chứng chỉ của amphora |

Vài điểm khác với Octavia cài truyền thống:

**Worker và health-manager phải nói chuyện trực tiếp với amphora.** Amphora là VM nằm trên một network của Neutron (`lb-mgmt-net`), còn worker và health-manager là pod trên controller. Để hai bên gặp nhau, Genestack tạo trước cho **mỗi controller** hai port Neutron trên `lb-mgmt-net`, gắn `--host=<tên controller>`:

- `octavia-worker-port-<node>`, device owner `Octavia:worker`;
- `octavia-health-manager-port-<node>`, device owner `Octavia:health-mgr`.

Khi pod khởi động, init container tìm port theo hostname của node, rồi tạo một interface OVS `internal` tên `o-w0` (worker) hoặc `o-hm0` (health-manager) trên `br-int`, gắn `iface-id` của port Neutron. Với OVN, interface đó trông giống hệt cổng của một VM: port được bind vào chassis của controller, DHCP của OVN cấp IP, và từ đó controller ở chung L2 với mọi amphora. Đây là lý do pod phải chạy `hostNetwork`, và vì sao `br-int` do kube-ovn quản lý cũng phục vụ luôn cho Octavia.

**Provider `ovn` dùng chung OVN của kube-ovn.** `install-octavia.sh` tự lấy ClusterIP của Service `ovn-nb` và `ovn-sb` trong `kube-system`, và chọn `tcp:` hay `ssl:` tuỳ cấu hình `ENABLE_SSL` của kube-ovn. Lab không bật SSL cho OVN nên dùng `tcp:`.

**Database và RabbitMQ do operator tạo.** Thay vì job `db-init` và `rabbit-init` của chart, Genestack apply các CR `Database`, `User`, `Grant` của mariadb-operator và `Queue`, `User`, `Permission` của RabbitMQ topology operator (thư mục `base-kustomize/octavia/base`), giống mọi service khác ở Phần 5.

**Không cần Redis.** Chart có khai báo endpoint `valkey`, nhưng Genestack đặt `task_flow.jobboard_enabled: false`, nên worker không dùng jobboard và không cần Redis.

Với cấu hình base của Genestack, amphora mặc định chạy `loadbalancer_topology: ACTIVE_STANDBY`, tức **mỗi LB là 2 VM** (2 vCPU, 1 GB RAM mỗi VM), đặt trên hai compute khác nhau nhờ `enable_anti_affinity`.

## Chuẩn bị

Các lệnh trong bài chạy trên deployer với quyền root, dùng lại hàm `os` từ Phần 5:

```bash
source /opt/genestack/scripts/genestack.rc
os() { kubectl -n openstack exec -i openstack-admin-client -- openstack "$@"; }
```

### Những gì đã có sẵn

Nếu đã làm theo các phần trước, một số thứ Octavia cần đã có từ lâu:

```bash
# Secret do create-secrets.sh tạo ở Phần 4b
kubectl -n openstack get secret octavia-admin octavia-db-password \
  octavia-rabbitmq-password octavia-certificates

# Endpoint public và HTTPRoute (Phần 4a, Phần 5)
grep -A4 'load_balancer:' /etc/genestack/helm-configs/global_overrides/endpoints.yaml
kubectl -n openstack get httproute custom-octavia-gateway-route
```

Secret `octavia-certificates` chứa **passphrase** cho private key của CA mà Octavia dùng để ký chứng chỉ cho amphora. Playbook ở bước sau sẽ đọc passphrase từ Secret này nếu nó đã tồn tại. Giá trị `octavia_key_passwd: not-secure-passphrase` trong defaults của role chỉ được dùng khi Secret chưa có.

Barbican cũng đã chạy trong lab. Barbican chỉ cần khi tạo listener `TERMINATED_HTTPS`: Octavia lấy chứng chỉ của listener từ Barbican.

### Chọn dải cho lb-mgmt-net

Mặc định role dùng `172.16.29.0/24` cho `lb-mgmt-net`, kiểu `geneve`, **không gateway**. Mỗi amphora tốn một IP trong dải này (hai IP mỗi LB với `ACTIVE_STANDBY`), cộng thêm hai IP cho mỗi controller. Kiểm tra dải đó không trùng với subnet nào đã có:

```bash
os subnet list -c Name -c Subnet
```

Trong lab, `test-subnet` là `172.16.0.0/24`, `public-subnet` là `192.168.10.0/24`, nên dải mặc định dùng được. Nếu cần đổi, ghi đè các biến `lb_mgmt_subnet_*` bằng `-e` khi chạy playbook.

### Cho deployer gọi được endpoint public

Đây là điểm dễ vấp nhất. Playbook `octavia-preconf-main.yaml` chạy trên `localhost` (deployer), và **mọi task trong role đều gọi API qua `interface: public`**, tức qua các hostname `*.api.openstack.example.com` của gateway. Deployer trong lab không có DNS cho các tên này, nên phải thêm vào `/etc/hosts` (giống cách làm cho máy client ở Phần 4a):

```bash
GW_IP=$(kubectl -n envoy-gateway get gateway flex-gateway -o jsonpath='{.status.addresses[0].value}')
kubectl get httproute -A -o jsonpath='{range .items[*]}{.spec.hostnames[*]}{"\n"}{end}' \
  | sort -u | sed "s/^/$GW_IP /" >> /etc/hosts

curl -sk -o /dev/null -w '%{http_code}\n' https://keystone.api.openstack.example.com/v3
```

Chứng chỉ của gateway trong lab là tự ký, nên cần bỏ qua kiểm tra TLS cho openstacksdk và `openstack` CLI mà playbook gọi. Chỉ làm vậy trong lab; ở production, hãy trỏ `OS_CACERT` tới CA thật của gateway.

```bash
export OS_INSECURE=true
```

## Bước 1 – Chạy playbook octavia-preconf

Role `octavia_preconf` tạo toàn bộ phần hạ tầng mà provider amphora cần. Mọi tài nguyên OpenStack được tạo trong project **`admin`**:

| Task | Tạo ra |
|---|---|
| Quota | Nâng quota của project `admin` cho 100 amphora: 200 core, 100 GB RAM, 500 port, 50 server group… |
| Network | `lb-mgmt-net` (geneve) và `lb-mgmt-subnet` (`172.16.29.0/24`, DHCP bật, không gateway) |
| Security group | `lb-mgmt-secgroup` cho amphora: TCP 9443, ICMP, SSH từ dải lb-mgmt. `lb-health-mgr-secgroup` và `lb-worker-secgroup`: UDP 5555 |
| Port | `octavia-health-manager-port-<node>` và `octavia-worker-port-<node>` cho mỗi node có label `openstack-control-plane=enabled` |
| Amphora | Keypair `amphora-ssh-key`, flavor private `m1.amphora` (2 vCPU, 1024 MB, 5 GB), image `amphora-ubuntu-noble` với tag `amphora` |
| Chứng chỉ | Server CA, client CA, client cert trong `~/octavia_certs`, đẩy lên Secret `octavia-certs` |
| Helm values | File values cho provider amphora: ID của network, secgroup, flavor, owner của image, danh sách `controller_ip_port_list` |

Chạy playbook, lấy mật khẩu admin từ Secret và truyền URL public của Keystone:

```bash
ansible-playbook /opt/genestack/ansible/playbooks/octavia-preconf-main.yaml \
  -e octavia_os_password="$(kubectl -n openstack get secret keystone-admin -o jsonpath='{.data.password}' | base64 -d)" \
  -e octavia_os_region_name=RegionOne \
  -e octavia_os_auth_url=https://keystone.api.openstack.example.com/v3 \
  -e octavia_helm_file=/tmp/octavia_amphora_provider.yaml
```

Một số điều cần biết về playbook này:

- **Image amphora là bản "test-only".** Role tải `test-only-amphora-x64-haproxy-ubuntu-noble.qcow2` từ tarballs.opendev.org. Bản này đủ cho lab. Ở production, hãy tự build image bằng `diskimage-create.sh` của Octavia, upload với cùng tag `amphora` và owner là project `admin`.
- **Có file đánh dấu trong `/tmp`.** Hai task tạo port chỉ chạy khi chưa có `/tmp/octavia_hm_controller_ip_port_list` và `/tmp/octavia_worker_controller_ip_port_list`. Bước tạo Secret cũng bỏ qua nếu đã có `/tmp/k8s_secret.yml`. Muốn chạy lại phần đó (ví dụ sau khi thêm controller), xoá các file này trước.
- **`/tmp/k8s_secret.yml` chứa private key của CA** ở dạng base64. Xoá nó sau khi playbook chạy xong, và backup thư mục `~/octavia_certs` cùng Secret `octavia-certs` vào nơi an toàn. Mất CA nghĩa là worker không còn nói chuyện được với các amphora đang chạy.

Kiểm tra kết quả:

```bash
os network show lb-mgmt-net -c provider:network_type -c status
os port list --device-owner Octavia:health-mgr --long -c Name -c "Fixed IP Addresses" -c Status
os port list --device-owner Octavia:worker --long -c Name -c "Fixed IP Addresses" -c Status
os image list --tag amphora --long
os flavor show m1.amphora -c vcpus -c ram -c disk
kubectl -n openstack get secret octavia-certs
cat /tmp/octavia_hm_controller_ip_port_list      # 3 cặp <IP>:5555, mỗi controller một cặp
```

Lúc này các port còn ở trạng thái `DOWN`. Chúng chỉ chuyển sang `ACTIVE` khi pod worker và health-manager cắm `o-w0`, `o-hm0` vào `br-int` ở bước 3.

Cuối cùng, chuyển file values vào thư mục override của Octavia để `install-octavia.sh` tự nạp. File chứa passphrase của CA, nên giới hạn quyền đọc:

```bash
mkdir -p /etc/genestack/helm-configs/octavia
mv /tmp/octavia_amphora_provider.yaml /etc/genestack/helm-configs/octavia/
chmod 600 /etc/genestack/helm-configs/octavia/octavia_amphora_provider.yaml
rm -f /tmp/k8s_secret.yml
```

## Bước 2 – Override cho lab

Hai giá trị mặc định không hợp với lab nested:

1. **`ACTIVE_STANDBY`** tạo 2 amphora cho mỗi LB. Với 3 compute 8 vCPU / 16 GB vẫn chạy được, nhưng mỗi lần tạo LB phải chờ boot hai VM. Lab dùng `SINGLE` để tiết kiệm.
2. **Thời gian chờ amphora.** Compute trong lab chạy QEMU thuần, không có KVM (Phần 2), nên một VM Ubuntu boot chậm hơn nhiều so với trên phần cứng thật. Worker mặc định chỉ chờ VM `ACTIVE` trong `amp_active_retries × amp_active_wait_sec` = 30 × 10 giây, và chờ amphora-agent trả lời trong `connection_max_retries × connection_retry_interval` = 120 × 5 giây. Hết thời gian, LB chuyển sang `ERROR`. Nên nới rộng các giá trị này.

```bash
cat > /etc/genestack/helm-configs/octavia/octavia-lab-overrides.yaml <<'EOF'
conf:
  octavia:
    controller_worker:
      loadbalancer_topology: SINGLE
      amp_active_retries: 60
      amp_active_wait_sec: 10
    haproxy_amphora:
      connection_max_retries: 360
      connection_retry_interval: 5
EOF
```

`install-octavia.sh` nạp mọi file `*.yaml` trong `/etc/genestack/helm-configs/octavia/` theo thứ tự tên file, và Helm gộp các map lồng nhau. Vì vậy hai file override trên cùng tồn tại được: mỗi file chỉ đặt những khoá riêng của mình trong `conf.octavia`.

Ở production, giữ `ACTIVE_STANDBY` và các giá trị chờ mặc định. Nếu thấy amphora thường xuyên boot không kịp, hãy tìm nguyên nhân ở compute (image, storage, CPU) thay vì chỉ tăng thời gian chờ.

## Bước 3 – Cài Octavia

```bash
/opt/genestack/bin/install-octavia.sh
```

Script đi theo khuôn chung của Genestack (Phần 5): lấy version chart từ `helm-chart-versions.yaml` (lab: `2025.1.18+12458c92d`), nạp base override, global override, rồi override riêng trong `/etc/genestack/helm-configs/octavia/`. Mật khẩu được lấy từ các Secret và truyền qua `--set`, cùng với passphrase của CA và địa chỉ OVN NB/SB. Sau đó post-renderer kustomize thêm các CR cho database, RabbitMQ, HPA, policy, và sidecar `octavia-driver-agent`.

Theo dõi các pod:

```bash
kubectl -n openstack get pods -o wide | grep octavia
kubectl -n openstack get ds,deploy | grep octavia
```

Khi ổn định, mỗi controller có một pod `octavia-worker` và một pod `octavia-health-manager`. Pod `octavia-api` có 2/2 container (API và driver agent).

### Kiểm tra o-w0 và o-hm0

Đây là phần dễ hỏng nhất, nên kiểm tra kỹ trước khi tạo LB đầu tiên. Trên **mỗi controller**:

```bash
ip -4 addr show o-hm0      # có IP trong 172.16.29.0/24, trùng IP của port health-mgr
ip -4 addr show o-w0       # có IP trong 172.16.29.0/24, trùng IP của port worker
ss -lunp | grep 5555       # octavia-health-manager đang nghe UDP 5555
```

Từ deployer, xem interface trên `br-int` và trạng thái port trong Neutron:

```bash
kubectl ko vsctl openstack-controller-1 list-ports br-int | grep -E 'o-w0|o-hm0'
os port list --device-owner Octavia:health-mgr --long -c Name -c Status -c "Fixed IP Addresses"
```

Port phải ở trạng thái `ACTIVE`. Nếu vẫn `DOWN`, init container không cắm được interface: xem log container `octavia-health-manager-nic-init` (hoặc `octavia-worker-nic-init`) của pod trên node đó. Nếu interface có nhưng không có IP, xem log container chính, nơi chạy `dhclient o-hm0` trước khi khởi động service.

### Kiểm tra API

```bash
os loadbalancer provider list          # amphora và ovn
os endpoint list --service load-balancer
os loadbalancer list                   # rỗng, không lỗi
```

## Bước 4 – Thử load balancer với provider OVN

Bắt đầu với provider `ovn`, vì nó không phụ thuộc vào amphora: nếu bước này chạy được thì API, database, RabbitMQ và kết nối tới OVN NB đều ổn.

Member là hai VM CirrOS trên `test-subnet` từ Phần 5: `test-vm-1` (172.16.0.136) và `test-vm-2` (172.16.0.147). CirrOS không có web server, nhưng có `nc` của BusyBox. Trong console của **mỗi** VM, chạy một HTTP server tối giản trả về hostname:

```sh
while true; do
  echo -e "HTTP/1.0 200 OK\r\n\r\n$(hostname)" | sudo nc -l -p 80
done
```

Security group của member phải cho phép TCP 80. Với provider `ovn`, traffic tới member giữ nguyên **IP của client**, nên rule phải mở cho dải client, không chỉ cho subnet nội bộ.

```bash
os loadbalancer create --name lb-ovn --provider ovn --vip-subnet-id test-subnet --wait
os loadbalancer listener create --name lb-ovn-80 --protocol TCP --protocol-port 80 --wait lb-ovn
os loadbalancer pool create --name lb-ovn-pool --listener lb-ovn-80 \
  --protocol TCP --lb-algorithm SOURCE_IP_PORT --wait
os loadbalancer member create --subnet-id test-subnet --address 172.16.0.136 --protocol-port 80 --wait lb-ovn-pool
os loadbalancer member create --subnet-id test-subnet --address 172.16.0.147 --protocol-port 80 --wait lb-ovn-pool
os loadbalancer healthmonitor create --type TCP --delay 5 --timeout 3 --max-retries 3 --wait lb-ovn-pool

os loadbalancer show lb-ovn -c vip_address -c provisioning_status -c operating_status
```

Load balancer `ACTIVE` gần như ngay lập tức. Ở phía OVN, nó chỉ là một bản ghi `Load_Balancer`, đặt tên theo ID của LB trong Octavia:

```bash
LB_ID=$(os loadbalancer show lb-ovn -c id -f value)
kubectl ko nbctl lb-list | grep "$LB_ID"
```

Gắn floating IP vào port VIP để truy cập từ mạng `public` (`test-net` đã có router ra `public` từ Phần 5):

```bash
VIP_PORT=$(os loadbalancer show lb-ovn -c vip_port_id -f value)
FIP_OVN=$(os floating ip create public -c floating_ip_address -f value)
os floating ip set --port "$VIP_PORT" "$FIP_OVN"

for i in $(seq 1 6); do curl -s http://$FIP_OVN/; done
```

Với `SOURCE_IP_PORT`, mỗi kết nối mới (khác source port) có thể rơi vào member khác nhau, nên kết quả sẽ xen kẽ giữa hai hostname. Thử dừng vòng `nc` trên một VM: sau vài chu kỳ health monitor, member đó chuyển `operating_status` sang `ERROR` và không nhận traffic nữa.

```bash
os loadbalancer member list lb-ovn-pool -c address -c operating_status
```

## Bước 5 – Thử load balancer với provider amphora

```bash
os loadbalancer create --name lb-amp --vip-subnet-id test-subnet     # provider mặc định: amphora
```

Lần này `provisioning_status` sẽ ở `PENDING_CREATE` khá lâu. Trong lúc chờ, theo dõi từng mắt xích:

```bash
os loadbalancer amphora list --loadbalancer lb-amp     # BOOTING → ALLOCATED
os server list --project admin --name amphora          # VM amphora-<uuid> trên compute
kubectl -n openstack logs -f ds/octavia-worker-default -c octavia-worker
```

Log của worker cho biết nó đang ở bước nào: chờ Nova đưa VM sang `ACTIVE`, rồi lặp lại việc gọi `https://<IP lb-mgmt của amphora>:9443`. Những dòng báo không kết nối được trong lúc amphora còn đang boot là bình thường. Chỉ đáng lo khi đã hết số lần thử.

Khi LB `ACTIVE`, tạo listener HTTP, pool và health monitor HTTP. Provider amphora hỗ trợ thuật toán `ROUND_ROBIN` và kiểm tra sức khoẻ ở tầng HTTP:

```bash
os loadbalancer listener create --name lb-amp-http --protocol HTTP --protocol-port 80 --wait lb-amp
os loadbalancer pool create --name lb-amp-pool --listener lb-amp-http \
  --protocol HTTP --lb-algorithm ROUND_ROBIN --wait
os loadbalancer member create --subnet-id test-subnet --address 172.16.0.136 --protocol-port 80 --wait lb-amp-pool
os loadbalancer member create --subnet-id test-subnet --address 172.16.0.147 --protocol-port 80 --wait lb-amp-pool
os loadbalancer healthmonitor create --type HTTP --url-path / --expected-codes 200 \
  --delay 5 --timeout 3 --max-retries 3 --wait lb-amp-pool

VIP_PORT=$(os loadbalancer show lb-amp -c vip_port_id -f value)
FIP_AMP=$(os floating ip create public -c floating_ip_address -f value)
os floating ip set --port "$VIP_PORT" "$FIP_AMP"
for i in $(seq 1 6); do curl -s http://$FIP_AMP/; done      # xen kẽ đều giữa hai VM
```

Lần này member nhìn thấy kết nối đến từ **IP của amphora** trên `test-subnet`, không phải IP của client.

Xem trạng thái amphora và kiểm tra heartbeat:

```bash
os loadbalancer amphora list --loadbalancer lb-amp -c id -c role -c status -c lb_network_ip
os loadbalancer amphora stats show <amphora-id>
kubectl -n openstack logs ds/octavia-health-manager-default -c octavia-health-manager --tail=50
```

### Dọn dẹp sau khi thử

```bash
os loadbalancer delete --cascade --wait lb-ovn
os loadbalancer delete --cascade --wait lb-amp
os floating ip delete "$FIP_OVN" "$FIP_AMP"
```

Xoá LB amphora cũng xoá VM amphora tương ứng.

## Khi có controller mới hoặc bớt controller

Port `o-w0`/`o-hm0` được tạo **theo từng controller**, và danh sách `controller_ip_port_list` được ghi vào values của chart cũng như vào cấu hình của **mọi amphora đang chạy**. Vì vậy mỗi khi thay đổi tập controller (như ở [Phần 8](/posts/genestack-09-add-compute-node/) và [Phần 9](/posts/genestack-10-replace-controller/)), cần cập nhật Octavia.

Khi **thêm** controller, sau khi node đã có label `openstack-control-plane=enabled`:

```bash
# 1. Tạo port cho node mới (port của node cũ được giữ nguyên) và sinh lại values
rm -f /tmp/octavia_hm_controller_ip_port_list /tmp/octavia_worker_controller_ip_port_list /tmp/k8s_secret.yml
ansible-playbook /opt/genestack/ansible/playbooks/octavia-preconf-main.yaml <các -e như ở bước 1>
mv /tmp/octavia_amphora_provider.yaml /etc/genestack/helm-configs/octavia/

# 2. Cập nhật chart: DaemonSet sẽ tự chạy trên controller mới
/opt/genestack/bin/install-octavia.sh

# 3. Đẩy controller_ip_port_list mới vào các amphora đang chạy
for a in $(os loadbalancer amphora list --status ALLOCATED -c id -f value); do
  os loadbalancer amphora configure "$a"
done
```

Bỏ qua bước 3 thì các amphora cũ vẫn chỉ gửi heartbeat cho những controller cũ. Chưa hỏng ngay, nhưng khi các controller cũ lần lượt bị gỡ, health-manager không còn nhận được heartbeat của chúng nữa và sẽ failover hàng loạt.

Khi **gỡ** controller, xoá thêm port của nó trước khi sinh lại values:

```bash
os port delete octavia-health-manager-port-<node-cũ> octavia-worker-port-<node-cũ>
```

## Chẩn đoán nhanh

| Triệu chứng | Kiểm tra | Nguyên nhân thường gặp |
|---|---|---|
| Pod worker/health-manager kẹt ở `Init` | Log container `*-nic-init`, `*-get-port` | Không tìm thấy port `octavia-*-port-<hostname>`: playbook chạy trước khi node có label, hoặc tên node khác hostname |
| Port `Octavia:*` vẫn `DOWN` | `kubectl ko vsctl <node> list-ports br-int` | Interface chưa được cắm vào `br-int`, hoặc chassis của node không có trong OVN SB |
| `o-hm0` không có IP | `ip addr show o-hm0`, DHCP của subnet | `lb-mgmt-subnet` tắt DHCP, hoặc port bị bind sai host |
| LB kẹt `PENDING_CREATE` rồi `ERROR` | Log worker, `os server show` amphora | Amphora boot quá chậm so với thời gian chờ (Bước 2), quota project `admin`, không đủ chỗ cho anti-affinity, image không có tag `amphora` |
| Worker không gọi được 9443 | Từ controller: `curl -k https://<lb_network_ip>:9443` | Security group `lb-mgmt-secgroup`, `o-w0` không có IP, chứng chỉ trong `octavia-certs` không khớp |
| Amphora bị failover liên tục | Log health-manager | Heartbeat UDP 5555 không tới: `controller_ip_port_list` sai, hoặc amphora mang danh sách controller cũ (xem mục trên) |
| LB provider `ovn` không `ACTIVE` | Log container `octavia-driver-agent` trong pod API | Không kết nối được `ovn-nb`/`ovn-sb`, sai `tcp:`/`ssl:` |
| LB `ovn` `ACTIVE` nhưng không vào được VIP | `ops-tools/check_octavia_ovn` | Port VIP chưa được bind vào chassis nào trong OVN SB |

Genestack có sẵn script `/opt/genestack/ops-tools/check_octavia_ovn/check_octavia_ovn.sh` để kiểm tra trường hợp cuối: với mỗi LB, nó tìm port VIP và hỏi OVN xem port đã được bind vào chassis nào chưa. Mặc định script chỉ báo cáo (dry-run). Thêm `--apply` thì nó tự chạy `loadbalancer failover` cho LB lỗi.

## Tóm tắt

- Octavia trong Genestack bật hai provider: `amphora` (mặc định, L4/L7, mỗi LB là 1–2 VM) và `ovn` (chỉ L4, không tốn VM, dùng chung OVN của kube-ovn).
- Worker và health-manager chạy dạng DaemonSet `hostNetwork` trên controller, cắm `o-w0`/`o-hm0` vào `br-int` qua port Neutron tạo sẵn cho từng controller. Worker gọi amphora ở TCP 9443, amphora gửi heartbeat về UDP 5555.
- Playbook `octavia-preconf-main.yaml` tạo `lb-mgmt-net`, security group, port, flavor, image, chứng chỉ và file values. Nó gọi API qua endpoint **public**, nên deployer phải phân giải được hostname của gateway.
- Image amphora của role là bản test-only. Backup `~/octavia_certs` và Secret `octavia-certs`, xoá `/tmp/k8s_secret.yml`.
- Trong lab nested: dùng `SINGLE` và nới thời gian chờ amphora. Production giữ `ACTIVE_STANDBY`.
- Thử provider `ovn` trước để xác nhận API và OVN, rồi mới thử `amphora`.
- Mỗi lần thêm hoặc bớt controller: tạo/xoá port, sinh lại values, cài lại chart, và `amphora configure` cho các amphora đang chạy.
