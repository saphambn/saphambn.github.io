---
title: "OpenStack trên Kubernetes với Genestack – Phần 4a: MetalLB, cert-manager và Envoy Gateway"
date: 2026-10-21 08:04:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, metallb, envoy-gateway, gateway-api, cert-manager]
render_with_liquid: false
---

Ở [Phần 3](/posts/genestack-03-kubernetes-kubespray/), chúng ta đã có một cluster Kubernetes 5 node chạy kube-ovn. Trước khi cài bất kỳ service OpenStack nào, cluster cần thêm một **lớp hạ tầng nền**: cách đưa API ra ngoài, chứng chỉ TLS, storage, database và message bus.

Lớp này khá lớn, nên được chia làm hai bài:

- **Phần 4a (bài này)** – đường vào của API: **MetalLB** cấp IP, **cert-manager** cấp chứng chỉ, **Envoy Gateway** định tuyến request theo hostname.
- **Phần 4b** – nơi giữ state: Longhorn, MariaDB Galera, RabbitMQ, memcached.

Thứ tự trong series được nhóm theo chức năng. Script cài tự động `bin/setup-infrastructure.sh` của Genestack cài theo thứ tự: kube-ovn → Longhorn → kube-prometheus-stack → cert-manager → MetalLB → namespace `openstack` → Envoy Gateway → MariaDB, RabbitMQ… Nếu một chart báo thiếu CRD như `ServiceMonitor`, hãy cài kube-prometheus-stack trước rồi quay lại.

| Thành phần | Version trong lab | Namespace |
|---|---|---|
| cert-manager | v1.19.5 | `cert-manager` |
| MetalLB | v0.15.2 | `metallb-system` |
| Envoy Gateway | v1.7.0 | `envoyproxy-gateway-system` (controller), `envoy-gateway` (Gateway) |

Version của từng chart nằm trong `/etc/genestack/helm-chart-versions.yaml`.

## cert-manager

cert-manager cấp và tự gia hạn chứng chỉ TLS cho các listener HTTPS của gateway. Nó cũng cấp chứng chỉ cho webhook của một số operator cài ở Phần 4b. Vì vậy nên cài cert-manager đầu tiên.

```bash
/opt/genestack/bin/install-cert-manager.sh
kubectl -n cert-manager wait --timeout=5m deploy/cert-manager --for=condition=available
```

Kiểm tra: ba deployment `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook` đều ở trạng thái `1/1`.

Các `ClusterIssuer` dùng cho gateway sẽ được tạo ở bước Envoy Gateway, nên lúc này chưa cần cấu hình gì thêm.

## MetalLB

### MetalLB giải quyết bài toán gì

Trên cloud công cộng, Service kiểu `LoadBalancer` được cloud provider cấp IP. Trên hạ tầng tự dựng thì không có ai làm việc đó, và Service sẽ nằm mãi ở trạng thái `<pending>`. **MetalLB** lấp vào chỗ trống này: nó cấp IP từ các dải bạn khai báo, rồi "quảng bá" IP đó ra mạng để client tìm được.

Genestack dùng MetalLB ở **chế độ L2**:

![MetalLB L2: một speaker làm chủ VIP và trả lời ARP, các node khác đứng chờ](/assets/img/genestack/04a-metallb-l2.svg)

- MetalLB chạy một **speaker** (DaemonSet) trên mỗi node. Các speaker bầu ra một node làm **chủ** cho mỗi VIP.
- Khi client hỏi ARP "ai giữ 10.0.0.200?", chỉ speaker chủ trả lời, bằng địa chỉ MAC của node mình. Từ đó mọi traffic tới VIP đi vào node đó, rồi kube-proxy chuyển tiếp tới pod phù hợp.
- Khi node chủ chết, một speaker khác nhận VIP và gửi gratuitous ARP để client cập nhật.

Chế độ L2 đơn giản, không cần router hỗ trợ BGP. Đổi lại, toàn bộ traffic của một VIP đi qua **một node** tại một thời điểm, và việc chuyển sang node khác mất vài giây.

### Điều kiện để MetalLB L2 hoạt động

Bốn điều kiện sau phải đồng thời thoả. Ba điều kiện đầu đã được chuẩn bị ở các bài trước:

| Điều kiện | Vì sao | Chuẩn bị ở |
|---|---|---|
| `kube_proxy_strict_arp: true` | Với IPVS, **mọi** node đều gắn VIP lên interface `kube-ipvs0`. Không bật strict ARP thì node nào cũng trả lời ARP cho VIP | Phần 3 |
| allowed-address-pairs trên port Neutron | Nếu chạy trên OpenStack cloud, port security sẽ chặn traffic tới VIP | Phần 2 |
| Node nằm trong `nodeSelectors` của `L2Advertisement` | Chỉ node được chọn mới được announce VIP | File manifest bên dưới |
| Node không bị loại bởi label `node.kubernetes.io/exclude-from-external-load-balancers` | MetalLB tôn trọng label chuẩn này của Kubernetes | Ngay bên dưới |

Điều kiện cuối cần chú ý riêng. Kubespray tự gắn label `exclude-from-external-load-balancers` cho các node control-plane, trong khi manifest mẫu của Genestack lại chỉ cho announce trên node có label `worker`, mà trong mô hình 3 controller thì đó chính là các control-plane. Nếu không xử lý, **không còn node nào đủ điều kiện**: VIP vẫn được cấp, nhưng không ai trả lời ARP. Có hai cách xử lý:

**Cách 1 – Gỡ label.** Script `setup-infrastructure.sh` của Genestack làm theo cách này:

```bash
kubectl label node --all node.kubernetes.io/exclude-from-external-load-balancers-
```

Nhược điểm: lần chạy Kubespray sau có thể gắn lại label.

**Cách 2 – Cho MetalLB bỏ qua label.** Cách này bền vững hơn, vì nằm trong cấu hình Helm:

```yaml
# /etc/genestack/helm-configs/metallb/metallb-helm-overrides.yaml
---
speaker:
  # Bỏ qua label node.kubernetes.io/exclude-from-external-load-balancers
  ignoreExcludeLB: true
```

Override này làm chart thêm flag `--ignore-exclude-lb` cho speaker. Lab dùng cách 2.

### Cài MetalLB

```bash
kubectl apply -f /etc/genestack/manifests/metallb/metallb-namespace.yaml
/opt/genestack/bin/install-metallb.sh
kubectl -n metallb-system wait --timeout=5m deploy/metallb-controller --for=condition=available
```

Namespace `metallb-system` được gắn label Pod Security ở mức `privileged`, vì speaker cần chạy với host network và quyền gửi gói ARP.

Có thể kiểm tra override đã vào chart chưa:

```bash
kubectl -n metallb-system get ds metallb-speaker -o yaml | grep ignore-exclude
```

### Khai báo dải VIP

Bootstrap đã chép một manifest mẫu ra `/etc/genestack/manifests/metallb/metallb-openstack-service-lb.yml`. Hãy sửa dải IP và interface cho khớp môi trường. Bản của lab:

```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: gateway-api-external
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.200/32          # VIP cho Envoy Gateway – mọi API
  autoAssign: false
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: openstack-external-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - gateway-api-external
  nodeSelectors:
  - matchLabels:
      node-role.kubernetes.io/worker: worker
  interfaces:
  - enp3s0                 # chỉ announce trên mạng br-host
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: primary
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.208/28          # VIP cho MariaDB, RabbitMQ…
  autoAssign: false
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: openstack-internal-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - primary
  nodeSelectors:
  - matchLabels:
      node-role.kubernetes.io/worker: worker
  interfaces:
  - enp3s0
```

```bash
kubectl apply -f /etc/genestack/manifests/metallb/metallb-openstack-service-lb.yml
```

Nếu lệnh trên báo lỗi webhook của MetalLB, đó là do webhook chưa sẵn sàng. Chờ vài giây rồi chạy lại.

Ba điểm cần hiểu trong manifest:

- **`autoAssign: false`**: MetalLB không tự cấp IP từ pool này cho mọi Service `LoadBalancer`. Service phải chỉ định rõ pool bằng annotation `metallb.io/address-pool: <tên pool>`. Gateway của Genestack và Service của MariaDB đều đã có sẵn annotation này. Nhờ vậy, một Service `LoadBalancer` lạ không thể "lấy mất" VIP của API.
- **`interfaces: [enp3s0]`**: chỉ announce trên mạng quản trị. Nếu không giới hạn, speaker có thể trả lời ARP trên cả interface overlay hay provider.
- **`nodeSelectors`**: quyết định node nào được giữ VIP. Lab chọn 3 controller, nên compute node không bao giờ nhận traffic API.

**Chọn dải VIP:** lab đặt VIP ngay trong subnet `10.0.0.0/24` của br-host. Cách này đơn giản: client cùng subnet tìm thấy VIP bằng ARP mà không cần thêm route. Tài liệu Genestack gợi ý một cách khác: dùng một dải **không trùng** với subnet của các interface trên node, để sau này dễ chuyển sang chế độ BGP. Đổi lại, cách đó cần cấu hình route trên router để client tới được dải VIP. Dù chọn cách nào, nhớ loại dải VIP khỏi DHCP (Phần 2).

## Namespace `openstack`

Các HTTPRoute của gateway nằm trong namespace `openstack`, nên cần tạo namespace này trước khi cấu hình gateway:

```bash
kubectl apply -k /etc/genestack/kustomize/openstack/base
```

## Envoy Gateway

### Gateway API và Envoy Gateway

Genestack dùng **Gateway API**, thế hệ kế tiếp của Ingress trong Kubernetes, với **Envoy Gateway** là controller. Mô hình gồm ba lớp object:

- **GatewayClass** (`eg`): loại gateway, gắn với controller Envoy Gateway.
- **Gateway** (`flex-gateway`): một "cổng vào" cụ thể, gồm các **listener**. Mỗi listener là một tổ hợp port, giao thức, hostname và chứng chỉ TLS.
- **HTTPRoute**: quy tắc chuyển request từ một listener tới Service phía sau.

Khi có Gateway, Envoy Gateway controller tự tạo một Deployment Envoy proxy và một Service `LoadBalancer` cho nó. Chính Service này nhận VIP `10.0.0.200` từ MetalLB.

![Các object đưa API Keystone ra ngoài: ClusterIssuer, Certificate, Gateway và listener, HTTPRoute, Service keystone-api, Envoy proxy, MetalLB](/assets/img/genestack/04a-gateway-objects.svg)

### Cài Envoy Gateway

```bash
/opt/genestack/bin/install-envoy-gateway.sh
kubectl -n envoyproxy-gateway-system wait --timeout=5m deploy/envoy-gateway --for=condition=available
```

Chart được kéo từ registry OCI `oci://docker.io/envoyproxy/gateway-helm`, đi kèm CRD của Gateway API.

### Cấu hình gateway, listener và route

Script `setup-envoy-gateway.sh` dựng toàn bộ phần cấu hình. Tham số quan trọng nhất là `--domain`: mọi API sẽ có hostname dạng `<service>.<domain>`.

```bash
/opt/genestack/bin/setup-envoy-gateway.sh --domain api.openstack.example.com
```

Script làm các việc sau:

1. apply `kustomize/envoyproxy-gateway`: GatewayClass `eg`, Gateway `flex-gateway`, các ClusterIssuer nội bộ (`flex-gateway-issuer` self-signed, `public-endpoint-issuer` dạng CA), cùng các policy của Envoy;
2. sinh **một listener HTTPS cho mỗi service OpenStack** (keystone, nova, glance, neutron, placement, cinder, skyline, novnc…) và các listener wildcard `*.<domain>` cho HTTP/HTTPS. Template nằm ở `etc/gateway-api/listeners/` và `etc/gateway-api/routes/`, trong đó chuỗi `your.domain.tld` được thay bằng domain bạn truyền vào;
3. apply HTTPRoute cho từng service. Route trỏ tới Service API tương ứng, ví dụ `keystone-api:5000`.

Lúc này các Service API chưa tồn tại, vì OpenStack chưa được cài. Route vẫn được tạo, và sẽ tự hoạt động khi các service được cài ở Phần 5.

### Chọn issuer cho chứng chỉ

Mỗi listener HTTPS cần một chứng chỉ. cert-manager đọc annotation `cert-manager.io/cluster-issuer` trên Gateway để biết dùng issuer nào:

| Cách chạy | Issuer | Phù hợp khi |
|---|---|---|
| Không có `--email` | `flex-gateway-issuer` (self-signed) | Lab, domain nội bộ, không truy cập được từ Internet |
| `--email <địa chỉ>` | `letsencrypt-prod` (ACME, mặc định HTTP-01) | Domain public, và gateway nhận được traffic port 80 từ Internet |
| `--email` + `--challenge dns01 --dns-plugin <tên>` | Let's Encrypt qua DNS-01 | Domain public, nhưng gateway không mở ra Internet |

Với ACME HTTP-01, Let's Encrypt phải truy cập được `http://<service>.<domain>/.well-known/acme-challenge/...` từ Internet. Nếu domain chỉ tồn tại trong mạng nội bộ, các `Certificate` sẽ không bao giờ chuyển sang `Ready`. Trong lúc chờ, cert-manager đặt tạm một chứng chỉ tự ký với issuer `cert-manager.local`. HTTPS vẫn chạy được, nhưng chứng chỉ đó không được gia hạn hay quản lý đúng nghĩa. Với lab dùng domain nội bộ như `example.com`, hãy chạy script **không kèm `--email`**.

Nếu đã lỡ chạy với `--email` trên một domain nội bộ, có thể chuyển về issuer self-signed:

```bash
kubectl -n envoy-gateway annotate --overwrite gateway/flex-gateway \
  cert-manager.io/cluster-issuer=flex-gateway-issuer
# Xoá các Certificate đang lỗi để cert-manager tạo lại với issuer mới
kubectl -n envoy-gateway get certificate -o name | xargs kubectl -n envoy-gateway delete
```

DNS-01 hỗ trợ nhiều nhà cung cấp DNS (Cloudflare, Route53, Azure DNS, Google Cloud DNS, RFC2136…). Xem `setup-envoy-gateway.sh --help <plugin>` để biết tham số cho từng nhà cung cấp.

### DNS phía client

Client cần phân giải mọi hostname `*.<domain>` về VIP. Cách gọn nhất là tạo một bản ghi wildcard trên DNS server:

```
*.api.openstack.example.com.   IN  A   10.0.0.200
```

Trong lab không có DNS server, có thể thêm vào `/etc/hosts` của máy client. Danh sách hostname lấy từ các HTTPRoute:

```bash
kubectl get httproute -A -o jsonpath='{range .items[*]}{.spec.hostnames[0]}{"\n"}{end}' | sort -u
```

## Kiểm tra

```bash
kubectl -n envoy-gateway get gateway flex-gateway
kubectl -n envoyproxy-gateway-system get svc
kubectl get servicel2statuses -A
kubectl -n envoy-gateway get certificate
```

| Kiểm tra | Kết quả mong đợi |
|---|---|
| Gateway | `PROGRAMMED=True`, `ADDRESS=10.0.0.200` |
| Service Envoy proxy | Kiểu `LoadBalancer`, `EXTERNAL-IP=10.0.0.200` |
| `ServiceL2Status` | Có bản ghi cho Service Envoy, chỉ rõ node đang giữ VIP |
| Certificate | Tất cả `READY=True` |

Output mẫu:

```
$ kubectl -n envoy-gateway get gateway flex-gateway
NAME           CLASS   ADDRESS      PROGRAMMED   AGE
flex-gateway   eg      10.0.0.200   True         17d

$ kubectl -n metallb-system get ipaddresspool
NAME                   AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
gateway-api-external   false         false             ["10.0.0.200/32"]
primary                false         false             ["10.0.0.208/28"]
```

Thử kết nối từ một máy cùng mạng với VIP. Lúc này chưa có service OpenStack nào, nên mã trả về là lỗi từ Envoy (thường là `404` hoặc `503`). Điều quan trọng là kết nối TLS được thiết lập:

```bash
curl -vk --resolve keystone.api.openstack.example.com:443:10.0.0.200 \
  https://keystone.api.openstack.example.com/ 2>&1 | grep -E "subject|issuer|HTTP/"
```

Một vài hành vi trông giống lỗi nhưng thực ra bình thường:

- **`ping 10.0.0.200` báo "port unreachable".** Với IPVS, VIP chỉ nhận các port đã khai báo trong Service. Hãy kiểm tra bằng `curl` hoặc `nc`.
- **`curl https://10.0.0.200` (gọi bằng IP) bị reset ngay sau TLS handshake.** Envoy chọn listener theo SNI, nên phải gọi bằng hostname.

Nếu VIP hoàn toàn không phản hồi (`ip neigh` trên client hiện `INCOMPLETE`), hãy kiểm tra lần lượt bốn điều kiện trong bảng ở mục MetalLB. Đặc biệt, `kubectl get servicel2statuses -A` rỗng nghĩa là không có node nào được phép announce.

## Ghi chú khi nâng cấp

- Helm **không cập nhật CRD** nằm trong thư mục `crds/` của chart khi nâng cấp. Khi nâng Envoy Gateway lên version mới, hãy apply CRD của version đó trước. Dùng `--server-side`, vì CRD của Gateway API quá lớn so với giới hạn annotation của client-side apply:
  ```bash
  kubectl apply --server-side --force-conflicts -f <thư-mục-chart>/crds/gatewayapi-crds.yaml
  kubectl apply --server-side --force-conflicts -f <thư-mục-chart>/crds/generated
  ```
- Đổi dải VIP của MetalLB là thao tác gây gián đoạn. Quy trình: xoá `IPAddressPool` cũ, apply manifest mới, rồi restart `metallb-controller`. Sau đó client phải cập nhật DNS hoặc file hosts sang IP mới.

## Tóm tắt

- cert-manager cài đầu tiên, để cấp chứng chỉ cho gateway và webhook.
- MetalLB L2 cấp VIP và trả lời ARP từ một node. Cần strict ARP, allowed-address-pairs, node nằm trong `nodeSelectors`, và xử lý label `exclude-from-external-load-balancers` (lab dùng `ignoreExcludeLB: true`).
- Pool để `autoAssign: false`. Service chọn pool qua annotation `metallb.io/address-pool`.
- `setup-envoy-gateway.sh --domain <domain>` tạo Gateway `flex-gateway`, một listener và một HTTPRoute cho mỗi service OpenStack.
- Domain nội bộ thì không dùng `--email`, để gateway dùng issuer self-signed. ACME chỉ phù hợp với domain public.

## Tiếp theo

Ở **Phần 4b**, chúng ta hoàn thiện lớp hạ tầng nền với phần giữ state: Longhorn cho persistent volume, MariaDB Galera qua mariadb-operator, RabbitMQ và memcached.
