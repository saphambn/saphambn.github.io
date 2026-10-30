---
title: "OpenStack trên Kubernetes với Genestack – Phần 15: DNS-as-a-Service với Designate và BIND9"
date: 2026-10-30 16:00:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, designate, bind9, dns, neutron]
render_with_liquid: false
---

Bài này thêm **DNS-as-a-Service** cho cloud: người dùng tự tạo zone, recordset qua API, và Neutron tự tạo bản ghi A/PTR cho floating IP, cho port của VM. Service trong OpenStack là **Designate** (service type `dns`), backend là **BIND9**.

Genestack chỉ cài Designate. **Genestack không cài DNS server.** Bạn tự dựng BIND9, đưa key `rndc` cho Designate, và mở đường để BIND9 kéo zone từ `designate-mdns`. Bài đi qua đủ ba phần đó trên lab (`release-2026.2.0.2`, OpenStack 2025.1), kèm những chỗ cấu hình mặc định của Genestack chưa chạy được với BIND9.

## Designate hoạt động thế nào

| Thành phần | Chạy ở đâu | Việc |
|---|---|---|
| `designate-api` | Pod | API v2, port 9001 |
| `designate-central` | Pod | Ghi zone/recordset vào database, kiểm tra quota, quyền |
| `designate-worker` | Pod | Gọi backend: chạy `rndc addzone`/`delzone` tới BIND9, rồi chờ BIND9 có serial mới |
| `designate-producer` | Pod | Tác vụ định kỳ: tăng serial, dọn zone bị xoá |
| `designate-mdns` | Pod | "Mini DNS": là **primary** của mọi zone. Gửi NOTIFY, trả lời AXFR cho BIND9 |
| BIND9 | Ngoài Kubernetes | **Secondary** của mọi zone, trả lời truy vấn DNS của người dùng |

![Luồng của Designate với BIND9: API, central, worker chạy rndc addzone trên BIND9, BIND9 kéo zone từ mdns qua VIP LoadBalancer, mdns gửi NOTIFY, worker kiểm tra serial](/assets/img/genestack/15-designate-flow.svg)

Luồng khi tạo một zone:

1. `central` lưu zone, `worker` chạy `rndc addzone` trên BIND9. Zone được thêm dạng `type slave`, `masters` là địa chỉ của `designate-mdns`.
2. BIND9 kéo zone từ `mdns` bằng AXFR.
3. Mỗi lần recordset đổi, `mdns` gửi NOTIFY, BIND9 kéo lại zone.
4. `worker` hỏi SOA trên BIND9. Khi serial khớp, zone chuyển sang `ACTIVE`.

Các kết nối cần mở:

| Từ | Tới | Port | Việc |
|---|---|---|---|
| Pod `designate-worker` | BIND9 | TCP 953 | `rndc` |
| Pod `designate-worker` | BIND9 | UDP/TCP 53 | Kiểm tra serial |
| Pod `designate-mdns` | BIND9 | UDP 53 | NOTIFY |
| BIND9 | `designate-mdns` | TCP/UDP 5354 | AXFR, SOA |
| Người dùng | BIND9 | UDP/TCP 53 | Truy vấn DNS |

Kết nối thứ tư là chỗ khó nhất: BIND9 nằm ngoài cluster nhưng phải gọi được vào pod `mdns`. Chart tạo Service `minidns` kiểu `NodePort` với port ngẫu nhiên trong dải 30000–32767, không dùng được làm `masters` một cách ổn định. Bài này tạo thêm một Service `LoadBalancer` qua MetalLB, IP cố định, port 5354.

## Thông số của lab

| Biến | Giá trị trong lab | Ghi chú |
|---|---|---|
| `BIND_IP` | `10.0.0.53` | IP thứ hai của deployer. `10.0.0.10:53` đã bị `dnsmasq` của lab giữ |
| `MDNS_VIP` | `10.0.0.215` | IP trống trong pool MetalLB `primary` (`10.0.0.208/28`) |
| `NODE_CIDR` | `10.0.0.0/24` | Dải IP node. Pod ra ngoài được SNAT thành IP node (`natOutgoing` của kube-ovn) |
| `NS_NAME` | `ns1.gslab.example.` | Tên NS ghi vào mọi zone |
| Zone thử | `demo.gslab.example.` | |

Kiểm tra `MDNS_VIP` chưa ai dùng:

```bash
kubectl get svc -A -o jsonpath='{range .items[*]}{.status.loadBalancer.ingress[0].ip}{"\n"}{end}' | sort -u
```

Production nên dùng **ít nhất hai** máy BIND9 riêng. Mỗi máy là một `nameserver` và một `target` trong pool. Bài làm với một máy trước, rồi thêm máy thứ hai (`10.0.0.93`) ở phần [Mở rộng lên hai BIND9](#mở-rộng-lên-hai-bind9).

## Bước 1 – Thêm IP cho BIND9 trên deployer

Port của deployer trên DFW3 có port security, nên phải cho phép IP mới trước. Chạy trên máy có openrc của DFW3:

```bash
openstack port set deployer-host-gslab --allowed-address ip-address=10.0.0.53
```

Trên deployer, thêm IP vào `enp3s0`:

```bash
cat > /etc/netplan/60-bind9-ip.yaml <<'EOF'
network:
  version: 2
  ethernets:
    enp3s0:
      addresses: [10.0.0.53/32]
EOF
chmod 600 /etc/netplan/60-bind9-ip.yaml
netplan apply
ip -4 a show enp3s0
ip route | grep 10.0.0.0/24
```

IP mới phải là **`/32`**, không phải `/24`. Với `/24`, kernel coi `10.0.0.53` là địa chỉ chính, `10.0.0.10` thành `secondary`, và route `10.0.0.0/24` đổi sang `src 10.0.0.53`. Từ đó mọi kết nối từ deployer tới các node đều đi ra bằng IP mới. Security group `default` của lab chỉ mở TCP giữa các máy theo `remote-group`, và remote-group chỉ chứa IP cố định của port, **không chứa allowed-address-pair**. Kết quả: `ping` vẫn thông (ICMP mở cho mọi nguồn), nhưng `kubectl` tới `10.0.0.11:6443` bị treo. Với `/32`, route vẫn là `src 10.0.0.10`. IP mới chỉ dùng để nhận kết nối tới BIND9, và các kết nối BIND9 tự mở ra (AXFR) đi bằng `10.0.0.10`.

Kiểm tra cả hai chiều:

```bash
kubectl get nodes
ssh -n 10.0.0.11 'timeout 5 bash -c "echo > /dev/tcp/10.0.0.53/22" && echo ok'
```

Đừng đặt `transfer-source` hay `notify-source` của BIND9 thành `10.0.0.53`: gói đi ra từ IP này tới node sẽ bị security group chặn.

`dnsmasq` của lab có `listen-address=10.0.0.10` và `bind-interfaces`, nên không chiếm IP mới.

## Bước 2 – Cài BIND9

```bash
apt-get install -y bind9 bind9-utils
```

Tạo key riêng cho Designate. Không dùng chung `rndc-key` mặc định của máy:

```bash
tsig-keygen -a hmac-sha256 designate > /etc/bind/designate.key
chown root:bind /etc/bind/designate.key
chmod 640 /etc/bind/designate.key
```

`/etc/bind/named.conf.options`:

```text
options {
    directory "/var/cache/bind";

    listen-on port 53 { 127.0.0.1; 10.0.0.53; };
    listen-on-v6 { none; };

    // Chỉ authoritative, không làm resolver
    recursion no;
    allow-query { any; };
    allow-transfer { none; };

    // NOTIFY từ pod mdns đi qua SNAT, tới với IP của node
    allow-notify { 10.0.0.0/24; };

    // Cho phép rndc addzone/delzone
    allow-new-zones yes;

    dnssec-validation no;
    minimal-responses yes;
};
```

`/etc/bind/named.conf.local`:

```text
include "/etc/bind/rndc.key";
include "/etc/bind/designate.key";

controls {
    inet 127.0.0.1 port 953 allow { 127.0.0.1; } keys { "rndc-key"; };
    inet 10.0.0.53 port 953 allow { 10.0.0.0/24; } keys { "designate"; };
};
```

Khi đã có khối `controls`, BIND9 không tự bật `rndc` local nữa, nên phải khai báo lại dòng `127.0.0.1` với `rndc-key`.

```bash
named-checkconf
systemctl restart named
systemctl enable named
ss -lntup | grep named
rndc status | head -3
rndc -s 10.0.0.53 -k /etc/bind/designate.key status | head -3
```

Zone do `rndc addzone` thêm vào được lưu trong `/var/cache/bind/_default.nzd`. Restart BIND9 không mất zone.

## Bước 3 – Đưa key cho Designate

Tên Secret và tên file trong Secret phải đúng là `rndc-key-secret` và `rndc.key`:

```bash
kubectl -n openstack create secret generic rndc-key-secret \
  --from-file=rndc.key=/etc/bind/designate.key
```

## Bước 4 – Override cho Designate

Base override của `release-2026.2.0.2` chưa chạy được với BIND9:

| Vấn đề | Hậu quả | Sửa |
|---|---|---|
| Pool mặc định là `pdns4`, IP `127.0.0.1` | Không có backend thật | Ghi đè `conf.pools` |
| Không mount `rndc.key` vào pod | `rndc` không có key | Thêm `pod.mounts` cho `worker` |
| Chart mount `emptyDir` đè lên `/etc/designate`, che mất `rootwrap.conf` | `rndc` (chạy qua `sudo designate-rootwrap`) lỗi | `root_helper` trỏ tới `/etc/designate-rootwrap.conf`, có trong image từ cuối 09/2026 ([genestack#1827](https://github.com/rackerlabs/genestack/pull/1827)) |
| Image `designate_service_cleaner` là `kernelpanic53/rackerlabs-designate:2026-master` của một tài khoản Docker Hub cá nhân | Rủi ro supply chain. Lệnh `designate-manage service clean` chỉ có từ 2025.2 | Tắt cron job trên 2025.1 |
| Không có endpoint public cho `dns` trong `global_overrides` | Endpoint public sai | Đặt `endpoints.dns` |

Tạo `/etc/genestack/helm-configs/designate/designate-bind9-overrides.yaml`:

```yaml
manifests:
  # 2025.1 chưa có "designate-manage service clean".
  # Từ 2025.2: bật lại và dùng image ghcr.io/rackerlabs/genestack-images/designate:<version>-latest
  cron_job_service_cleaner: false

network:
  mdns:
    node_port:
      # Dùng Service LoadBalancer ở bước 5 thay cho NodePort ngẫu nhiên
      enabled: false

pod:
  mounts:
    designate_worker:
      init_container: null
      designate_worker:
        volumeMounts:
          - name: rndc-key
            mountPath: /etc/designate/rndc.key
            subPath: rndc.key
            readOnly: true
        volumes:
          - name: rndc-key
            secret:
              secretName: rndc-key-secret

conf:
  designate:
    DEFAULT:
      root_helper: sudo designate-rootwrap /etc/designate-rootwrap.conf
  pools: |
    - name: default
      description: BIND9 pool
      attributes: {}
      ns_records:
        - hostname: ns1.gslab.example.
          priority: 1
      nameservers:
        - host: 10.0.0.53
          port: 53
      targets:
        - type: bind9
          description: BIND9 on gslab-deployer
          masters:
            - host: 10.0.0.215
              port: 5354
          options:
            host: 10.0.0.53
            port: 53
            rndc_host: 10.0.0.53
            rndc_port: 953
            rndc_key_file: /etc/designate/rndc.key

endpoints:
  dns:
    host_fqdn_override:
      public:
        tls: {}
        host: designate.api.openstack.example.com
    port:
      api:
        public: 443
    scheme:
      public: https
```

Ghi chú về `conf.pools`:

- Init container của `designate-worker` dựng `pools.yaml` bằng `eval "echo \"...\""` rồi chạy `designate-manage pool update`. **Không dùng dấu `"`, `$` hay backtick** trong khối này.
- Tên pool `default` không đổi được sau khi đã có zone.
- Production có nhiều BIND9: thêm mỗi máy một mục trong `nameservers` và một mục trong `targets`.

Mount chỉ cần cho `designate-worker`, vì backend chạy trong worker. Base override của Genestack `main` mount key vào mọi pod; cách nào cũng chạy được.

## Bước 5 – Service LoadBalancer cho mdns

Thêm Service vào kustomize overlay của Designate, để nó được cài cùng Helm release. Tạo `/etc/genestack/kustomize/designate/overlay/designate-mdns-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: designate-mdns-lb
  namespace: openstack
  annotations:
    metallb.io/address-pool: primary
    metallb.io/loadBalancerIPs: 10.0.0.215
spec:
  type: LoadBalancer
  selector:
    release_group: designate
    application: designate
    component: mdns
  ports:
    - name: mdns-tcp
      port: 5354
      targetPort: 5354
      protocol: TCP
    - name: mdns-udp
      port: 5354
      targetPort: 5354
      protocol: UDP
```

Thêm file vào `/etc/genestack/kustomize/designate/overlay/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
  - designate-mdns-lb.yaml
```

## Bước 6 – Cài Designate

```bash
/opt/genestack/bin/install-designate.sh
kubectl -n openstack get pods -l application=designate
kubectl -n openstack get svc designate-mdns-lb minidns
```

`designate-mdns-lb` phải có `EXTERNAL-IP` là `10.0.0.215`, và `minidns` là `ClusterIP`.

### Kiểm tra pool và rndc

Pool được nạp trong init container của worker:

```bash
kubectl -n openstack logs deploy/designate-worker -c designate-worker-init | tail -5
os dns service list
```

Gọi `rndc` từ trong pod worker, đúng như cách worker gọi. Lệnh này kiểm tra cùng lúc key, rootwrap và đường mạng tới BIND9:

```bash
kubectl -n openstack exec deploy/designate-worker -c designate-worker -- \
  ls -l /etc/designate/rndc.key /etc/designate-rootwrap.conf
kubectl -n openstack exec deploy/designate-worker -c designate-worker -- \
  sudo designate-rootwrap /etc/designate-rootwrap.conf \
  rndc -s 10.0.0.53 -p 953 -k /etc/designate/rndc.key status
```

Kiểm tra BIND9 gọi được mdns:

```bash
dig @10.0.0.215 -p 5354 example.invalid. SOA
```

mdns trả lời `REFUSED` hoặc `NOTAUTH` cho zone chưa có là đúng. Lỗi `timed out` nghĩa là đường tới VIP chưa thông.

### Tạo zone và recordset

```bash
os zone create --email admin@gslab.example demo.gslab.example.
os zone list -c name -c status -c action
```

Chờ zone chuyển từ `PENDING` sang `ACTIVE`, thường dưới 30 giây. Trên deployer, xem BIND9 nhận zone:

```bash
journalctl -u named --since "5 min ago" | grep -E "addzone|demo.gslab|transfer|notify"
```

Thêm một bản ghi rồi truy vấn thẳng BIND9:

```bash
os recordset create demo.gslab.example. www --type A --record 192.0.2.10
dig @10.0.0.53 www.demo.gslab.example. +short
dig @10.0.0.53 demo.gslab.example. SOA +short
os zone show demo.gslab.example. -c serial
```

Serial trên BIND9 phải bằng serial của Designate.

### Endpoint public

Listener `designate-https` và HTTPRoute `custom-designate-gateway-route` đã có từ lúc cài Gateway ([Phần 4a](/posts/genestack-04a-infrastructure-network/)). Kiểm tra catalog và gọi qua gateway:

```bash
os endpoint list --service dns -c Interface -c URL
curl -sk https://designate.api.openstack.example.com/ | head -c 200; echo
```

Nếu máy client chưa phân giải được `designate.api.openstack.example.com`, thêm tên này vào DNS của lab (dnsmasq trên deployer) hoặc `/etc/hosts`, trỏ về IP gateway `10.0.0.200`.

## Bước 7 – Tích hợp với Neutron

Neutron gọi Designate để tạo bản ghi khi người dùng gán tên cho floating IP hoặc port. Chart Neutron tự xử lý phần thông tin đăng nhập khi `external_dns_driver: designate`:

- Job `ks-user` tạo user `neutron_designate` trong project `service`, mật khẩu lấy từ Secret `designate-admin` (`install-neutron.sh` đã truyền sẵn).
- Secret `neutron-ks-etc` có thêm file `neutron_designate.conf` chứa section `[designate]` (auth URL, user, mật khẩu). File này được nạp sau `neutron.conf`.

Vì vậy override chỉ cần URL của Designate và các tuỳ chọn. Không cần chép mật khẩu vào file như tài liệu của Genestack.

Tạo `/etc/genestack/helm-configs/neutron/neutron-designate-overrides.yaml`:

```yaml
conf:
  neutron:
    DEFAULT:
      # Domain cho DNS nội bộ của Neutron. Phải là FQDN có dấu chấm cuối.
      # Không dùng cluster.local: trùng domain của Kubernetes.
      dns_domain: gslab.internal.
      external_dns_driver: designate
    designate:
      url: http://designate-api.openstack.svc.cluster.local:9001/v2
      allow_reverse_dns_lookup: true
      ipv4_ptr_zone_prefix_size: 24
      ipv6_ptr_zone_prefix_size: 116
  plugins:
    ml2_conf:
      ml2:
        # Giữ nguyên danh sách của base (port_security,qos,fwaas_v2), thêm hai driver DNS
        extension_drivers: "port_security,qos,fwaas_v2,dns_domain_ports,subnet_dns_publish_fixed_ip"
```

`extension_drivers` là một chuỗi: giá trị trong file của bạn **thay hẳn** giá trị của base. Trước khi ghi, kiểm tra lại danh sách của base trên phiên bản bạn đang chạy:

```bash
grep -n extension_drivers /opt/genestack/base-helm-configs/neutron/neutron-helm-overrides.yaml
```

Cài lại Neutron và kiểm tra cấu hình đã vào pod:

```bash
/opt/genestack/bin/install-neutron.sh
kubectl -n openstack exec deploy/neutron-server -c neutron-server -- \
  sh -c 'ls /etc/neutron/neutron.conf.d/; grep -v password /etc/neutron/neutron.conf.d/neutron_designate.conf'
kubectl -n openstack exec deploy/neutron-server -c neutron-server -- \
  grep -E '^(external_dns_driver|dns_domain|extension_drivers)' /etc/neutron/neutron.conf /etc/neutron/plugins/ml2/ml2_conf.ini
os user show neutron_designate --domain service -c name -c enabled
```

### Thử ba trường hợp

Zone phải có sẵn và **thuộc project của người dùng**. Neutron không tự tạo zone thuận; nó chỉ tự tạo zone ngược (PTR), trong project của `neutron_designate`.

**1. Floating IP có tên riêng.** Tên và domain đặt thẳng trên floating IP:

```bash
os floating ip create public --dns-domain demo.gslab.example. --dns-name fip1
os recordset list demo.gslab.example. -c name -c type -c records
dig @10.0.0.53 fip1.demo.gslab.example. +short
```

**2. Floating IP của VM.** Network có `dns_domain`, port của VM có `dns_name` (Nova đặt bằng hostname của VM). Khi gán floating IP, bản ghi được tạo theo tên VM:

```bash
os network set test-net --dns-domain demo.gslab.example.
os server create vm-dns1 --image "Cirros 0.6.2 64-bit" --flavor m1.tiny --network test-net --wait
os port list --server vm-dns1 -c ID -c "DNS Name" -c "DNS Assignment"
FIP=$(os floating ip create public -f value -c floating_ip_address)
os server add floating ip vm-dns1 "$FIP"
dig @10.0.0.53 vm-dns1.demo.gslab.example. +short
```

**3. Fixed IP.** Bật `dns_publish_fixed_ip` trên subnet để publish cả IP riêng của VM:

```bash
os subnet set test-subnet --dns-publish-fixed-ip
os server create vm-dns2 --image "Cirros 0.6.2 64-bit" --flavor m1.tiny --network test-net --wait
dig @10.0.0.53 vm-dns2.demo.gslab.example. +short
```

Bản ghi ngược của floating IP nằm trong zone `10.168.192.in-addr.arpa.`:

```bash
os zone list --all-projects -c name -c project_id -c status
dig @10.0.0.53 -x "$FIP" +short
```

Xoá VM, nhả floating IP rồi kiểm tra các bản ghi A và PTR cũng bị xoá.

## Mở rộng lên hai BIND9

Một BIND9 là một điểm lỗi duy nhất: máy đó tắt thì mọi zone ngừng trả lời. Phần này thêm máy thứ hai, `ns2`, IP `10.0.0.93`. Với Designate, mỗi máy BIND9 cần hai mục trong pool:

| Mục trong pool | Designate dùng để | Mỗi BIND9 cần |
|---|---|---|
| `nameservers` | Hỏi SOA, kiểm tra serial đã cập nhật | Một mục |
| `targets` | Chạy `rndc addzone`/`delzone` | Một mục, `rndc_host` riêng |
| `ns_records` | Ghi bản ghi NS vào mọi zone | Một tên (`ns1`, `ns2`) |

Hai BIND9 **không** truyền zone cho nhau. Mỗi máy là secondary độc lập, cùng kéo zone từ VIP của mdns.

![Designate với hai BIND9: worker chạy rndc và hỏi SOA trên cả ns1 và ns2, mdns gửi NOTIFY tới cả hai, mỗi BIND9 tự kéo zone qua VIP 10.0.0.215:5354, client hỏi ns1 hoặc ns2](/assets/img/genestack/15-designate-two-bind9.svg)

### Chuẩn bị ns2

`ns2` là một VM riêng, có IP cố định `10.0.0.93` trên port của chính nó. Vì vậy không cần IP `/32` hay allowed-address-pair như trên deployer. Port của VM phải nằm trong security group `default` của lab: rule `remote-group` cho node gọi được TCP 953 và 53 của nó.

Cài BIND9 như ở bước 2, với hai khác biệt:

- **Dùng chung key `designate`.** Chép `/etc/bind/designate.key` từ ns1 sang, không tạo key mới. Cả hai target dùng cùng file `/etc/designate/rndc.key` trong pod, nên không phải đổi Secret.
- **IP trong `listen-on` và `controls` là `10.0.0.93`.**

```bash
# Trên deployer (ns1)
scp /etc/bind/designate.key 10.0.0.93:/etc/bind/designate.key

# Trên ns2, sau khi apt-get install bind9 bind9-utils
chown root:bind /etc/bind/designate.key && chmod 640 /etc/bind/designate.key
```

`named.conf.options` trên ns2 giống ns1, chỉ đổi dòng:

```text
    listen-on port 53 { 127.0.0.1; 10.0.0.93; };
```

`named.conf.local` trên ns2 giống ns1, chỉ đổi dòng:

```text
    inet 10.0.0.93 port 953 allow { 10.0.0.0/24; } keys { "designate"; };
```

```bash
named-checkconf && systemctl restart named && systemctl enable named
```

Kiểm tra từ pod worker, trước khi sửa pool:

```bash
kubectl -n openstack exec deploy/designate-worker -c designate-worker -- \
  sudo designate-rootwrap /etc/designate-rootwrap.conf \
  rndc -s 10.0.0.93 -p 953 -k /etc/designate/rndc.key status
```

Bước này quan trọng: nếu thêm ns2 vào pool khi worker chưa gọi được nó, mọi thay đổi zone sau đó sẽ chuyển sang `ERROR` (xem `threshold_percentage` bên dưới).

### Sửa pool

Trong `/etc/genestack/helm-configs/designate/designate-bind9-overrides.yaml`, thay `conf.pools` và thêm `service:worker`:

```yaml
conf:
  designate:
    DEFAULT:
      root_helper: sudo designate-rootwrap /etc/designate-rootwrap.conf
    service:worker:
      # 100: zone chỉ ACTIVE khi mọi BIND9 đã có serial mới (mặc định)
      threshold_percentage: 100
  pools: |
    - name: default
      description: BIND9 pool
      attributes: {}
      ns_records:
        - hostname: ns1.gslab.example.
          priority: 1
        - hostname: ns2.gslab.example.
          priority: 2
      nameservers:
        - host: 10.0.0.53
          port: 53
        - host: 10.0.0.93
          port: 53
      targets:
        - type: bind9
          description: BIND9 ns1 on gslab-deployer
          masters:
            - host: 10.0.0.215
              port: 5354
          options:
            host: 10.0.0.53
            port: 53
            rndc_host: 10.0.0.53
            rndc_port: 953
            rndc_key_file: /etc/designate/rndc.key
        - type: bind9
          description: BIND9 ns2
          masters:
            - host: 10.0.0.215
              port: 5354
          options:
            host: 10.0.0.93
            port: 53
            rndc_host: 10.0.0.93
            rndc_port: 953
            rndc_key_file: /etc/designate/rndc.key
```

`threshold_percentage` quyết định một thay đổi cần bao nhiêu phần trăm BIND9 thành công:

- **100** (mặc định): zone chỉ `ACTIVE` khi **cả hai** BIND9 đã có serial mới. Một máy hỏng thì zone chuyển `ERROR`, dù máy còn lại vẫn đúng. Bạn biết ngay có máy hỏng, nhưng người dùng thấy lỗi.
- **50**: một máy là đủ. Người dùng không thấy lỗi khi một máy hỏng, nhưng máy hỏng có thể lệch dữ liệu mà không ai biết, trừ khi có giám sát riêng (so serial của hai máy).

Lab giữ 100. Production chọn theo việc bạn có giám sát serial từng BIND9 hay chưa.

Áp dụng. Init container của worker chạy lại `designate-manage pool update` khi pod khởi động:

```bash
/opt/genestack/bin/install-designate.sh
kubectl -n openstack rollout status deploy/designate-worker
kubectl -n openstack logs deploy/designate-worker -c designate-worker-init | tail -5
```

Nếu chỉ `pools` thay đổi mà pod worker không được tạo lại, thì restart nó: `kubectl -n openstack rollout restart deploy/designate-worker`.

### Đồng bộ zone đã có sang ns2

Zone tạo trước khi có ns2 chưa nằm trên ns2. Có hai cơ chế đưa chúng sang:

1. **Thêm `ns2` vào `ns_records`.** Khi pool có NS mới, Designate thêm bản ghi NS `ns2.gslab.example.` vào mọi zone của pool. Mỗi zone vì vậy được cập nhật, và worker gửi cập nhật tới mọi target.
2. **Backend bind9 tự tạo zone khi cập nhật.** Khi cập nhật zone, backend chạy `rndc showzone`. Nếu zone chưa có trên máy đó, nó chạy `addzone` thay cho `modzone`.

Kiểm tra:

```bash
os zone list -c name -c status -c serial
os recordset list demo.gslab.example. --type NS -c records
for ns in 10.0.0.53 10.0.0.93; do
  echo -n "$ns: "; dig @$ns demo.gslab.example. SOA +short
done
```

Hai BIND9 phải trả cùng serial, bằng serial trong Designate. Nếu một zone vẫn chưa có trên ns2, cập nhật zone đó để kích worker:

```bash
os zone set demo.gslab.example. --description "resync ns2"
```

### Thử khi một BIND9 hỏng

```bash
ssh 10.0.0.93 systemctl stop named
os recordset create demo.gslab.example. test-ha --type A --record 192.0.2.20
os zone show demo.gslab.example. -c status -c action     # thường ERROR với threshold 100
dig @10.0.0.53 test-ha.demo.gslab.example. +short          # ns1 vẫn có bản ghi
ssh 10.0.0.93 systemctl start named
os zone set demo.gslab.example. --description "recover"  # đẩy lại cho ns2
dig @10.0.0.93 test-ha.demo.gslab.example. +short
```

Producer cũng tự thử lại các zone `ERROR` theo định kỳ, nên nếu đợi vài phút, zone thường về `ACTIVE` mà không cần làm gì.

## Dọn dẹp và rollback

```bash
os server delete vm-dns1 vm-dns2
os floating ip list -c ID -c "Floating IP Address"    # xoá các FIP vừa tạo
os subnet unset test-subnet --dns-publish-fixed-ip
os network unset test-net --dns-domain
os zone delete demo.gslab.example.
```

Gỡ tích hợp Neutron: xoá `neutron-designate-overrides.yaml`, chạy lại `install-neutron.sh`. Gỡ Designate:

```bash
helm -n openstack uninstall designate
kubectl -n openstack delete secret rndc-key-secret
```

Trên BIND9, các zone đã thêm bằng `rndc addzone` vẫn còn trong `_default.nzd`. Xoá bằng `rndc delzone -clean <zone>`, hoặc dừng `named` và xoá file đó.

## Lưu ý cho production

- **Ít nhất hai BIND9**, đặt ở hai failure domain, khai báo cả hai trong `nameservers` và `targets`. Đăng ký NS và glue ở domain cha để Internet tìm được chúng.
- **Tăng replica**: base override đặt mọi thành phần 1 replica, PDB `min_available: 0`. Tối thiểu 2 cho `api`, `central`, `worker`, `mdns`, `producer`.
- **Giới hạn AXFR bằng TSIG**: VIP của mdns trả zone cho bất kỳ ai gọi được tới nó. Nếu VIP nằm trong mạng có người khác dùng, cấu hình TSIG cho AXFR.
- **Không dùng image của tài khoản cá nhân**: thay `designate_service_cleaner` bằng image `genestack-images` cùng phiên bản (từ 2025.2), hoặc tắt cron job.
- **Theo dõi** zone kẹt ở `PENDING` hoặc `ERROR`: rule Prometheus có trong tài liệu [Designate exporter](https://docs.rackspacecloud.com/openstack-designate-exporter/) của Genestack.

## Tóm tắt

- Designate là primary (qua `mdns`), BIND9 là secondary. Worker thêm zone bằng `rndc addzone`, BIND9 kéo zone qua AXFR từ mdns.
- Genestack không dựng DNS server. Tự cài BIND9 với `allow-new-zones yes`, key `rndc` riêng, `allow-notify` cho dải IP node.
- Trên `release-2026.2.0.2` phải tự ghi đè: pool BIND9, mount `rndc.key`, `root_helper` mới, tắt cron job service cleaner, endpoint public.
- Đưa mdns ra ngoài cluster bằng một Service `LoadBalancer` có IP cố định trong kustomize overlay.
- Tích hợp Neutron chỉ cần `external_dns_driver`, URL của Designate và hai extension driver; chart tự tạo user `neutron_designate`.
