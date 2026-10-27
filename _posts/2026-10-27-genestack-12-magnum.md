---
title: "OpenStack trên Kubernetes với Genestack – Phần 11: Kubernetes-as-a-Service với Magnum và Cluster API"
date: 2026-10-27 10:00:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, magnum, cluster-api, capi, octavia]
render_with_liquid: false
---

[Phần 10](/posts/genestack-11-octavia/) đã thêm load balancer cho cloud. Với Octavia, Cinder và Barbican đã chạy, cloud có đủ những gì một cluster Kubernetes cần từ hạ tầng: VM, mạng, volume, load balancer cho API và cho Service `type: LoadBalancer`. Bước tự nhiên tiếp theo là cho người dùng tự tạo **cluster Kubernetes** chỉ bằng một lệnh. Đó là việc của **Magnum** (service type `container-infra`).

Bài này cài Magnum theo cách Genestack khuyến nghị: dùng driver **Cluster API (CAPI)** thay cho driver Heat cũ. Driver này cần một **management cluster** chạy các controller của CAPI. Bài trình bày hai cách có nó: dùng luôn cluster Kubernetes của Genestack (lab), hoặc dựng một cluster riêng (production), kèm phân tích được mất của từng cách. Mạng, DNS và chứng chỉ cho endpoint public chỉ được nêu ở dạng điều kiện cần.

## Magnum với Cluster API hoạt động thế nào

![Magnum với driver Cluster API: magnum-conductor trong cluster Genestack cài Helm release lên management cluster, CAPI trên management cluster dựng workload cluster bằng API của OpenStack](/assets/img/genestack/12-magnum-capi.svg)

Driver cũ của Magnum dựng cluster bằng một Heat stack: Heat tạo VM, script trong VM tự cài Kubernetes. Driver này đã bị bỏ. Genestack dùng **`magnum-capi-helm`**, và chia việc cho ba tầng:

| Tầng | Chạy ở đâu | Việc |
|---|---|---|
| ① Magnum | `magnum-api` và `magnum-conductor` trong cluster Genestack | Nhận yêu cầu tạo cluster, tạo application credential cho người dùng, rồi `helm install` chart `openstack-cluster` lên management cluster |
| ② Management cluster | Cluster Genestack (phương án A) hoặc 3 VM riêng (phương án B) | Chạy các controller của Cluster API: core CAPI, **CAPO** (provider OpenStack), kubeadm bootstrap/control-plane, ORC, và `cluster-api-addon-provider` |
| ③ Workload cluster | VM trong project của người dùng | Cluster Kubernetes mà người dùng nhận được. Addon (CNI, OpenStack CCM, Cinder CSI) được addon-provider cài sau khi cluster lên |

Một vài điểm quan trọng:

- **Magnum không tự dựng Kubernetes.** Conductor chỉ cài, nâng cấp, xoá một Helm release trên management cluster (mỗi project một namespace `magnum-<project-id>`), rồi đọc trạng thái các object CAPI để cập nhật `status` của cluster. Vì vậy conductor phải có **kubeconfig** của management cluster.
- **Mọi thành phần gọi API của OpenStack qua endpoint public.** CAPO tạo VM, network, LB bằng application credential của người dùng. Trong workload cluster, OpenStack CCM và Cinder CSI cũng gọi API bằng credential đó. Driver ghi `auth_url` **public** của Keystone vào `clouds.yaml` của từng cluster. Vì vậy cả management cluster lẫn VM của workload cluster đều phải phân giải được hostname, đi tới được, và **tin cậy chứng chỉ** của `*.api.openstack.example.com`.
- **Management cluster phải gọi được API của từng workload cluster.** Đó là floating IP của LB API, nằm trên mạng `public`. Điều này đúng với cả hai phương án.

Hai điểm cuối là lý do bảng điều kiện cần bên dưới có các dòng về mạng, DNS và chứng chỉ.

## Điều kiện cần

| Điều kiện | Trong lab | Ghi chú |
|---|---|---|
| Barbican | Đã chạy | Magnum lưu chứng chỉ của cluster trong Barbican (`cert_manager_type: barbican`) |
| Cinder | Phần 6, volume type `lvm-ssd` | Workload cluster dùng Cinder CSI. Phương án B còn boot VM mgmt từ volume |
| Octavia | Phần 10 | LB cho API của workload cluster (và của mgmt cluster ở phương án B) |
| Mạng external flat hoặc VLAN | `public` (flat, `192.168.10.0/24`) | Phương án B: playbook kiểm tra loại mạng này |
| Mạng `public` ra được Internet, và đi tới được hai chiều với mạng của node (`br-host`) | Gateway `192.168.10.1` của hạ tầng | VM cần kéo image và chart, gọi được gateway `10.0.0.200`. Management cluster cần gọi floating IP của từng workload cluster |
| DNS phân giải được `*.api.openstack.example.com`, dùng được từ cả VM lẫn pod | dnsmasq trên deployer, `10.0.0.54` | Ở production dùng DNS thật. Nếu hạ tầng có security group, mở UDP/TCP 53 |
| Chứng chỉ của gateway được ký bởi một CA mà mọi bên tin được | Issuer `public-endpoint-issuer` của Genestack. CA ở `/etc/genestack/public-endpoint-ca.crt` và Secret `openstack/openstack-public-ca` | Chứng chỉ tự ký sẽ làm Magnum lỗi. Ở production nên dùng domain thật + Let's Encrypt DNS-01 (Phần 4a) |
| Image có Kubernetes | Chưa có | Bước 3 |
| Secret `magnum-*`, endpoint `container_infra`, HTTPRoute | Đã có từ Phần 4b và Phần 5 | |

Phần còn lại của bài dùng ba giá trị của lab: DNS `10.0.0.54`, file CA `/etc/genestack/public-endpoint-ca.crt`, và Secret `openstack-public-ca` chứa CA đó. Nếu môi trường của bạn khác, thay các giá trị tương ứng.

Kiểm tra nhanh những gì đã có:

```bash
source /opt/genestack/scripts/genestack.rc
os() { kubectl -n openstack exec -i openstack-admin-client -- openstack "$@"; }

kubectl -n openstack get secret magnum-admin magnum-db-password magnum-rabbitmq-password
kubectl -n openstack get httproute custom-magnum-gateway-route
os volume type list
os network show public -c provider:network_type -c router:external -c shared
os loadbalancer provider list
kubectl -n openstack get secret openstack-public-ca
nslookup keystone.api.openstack.example.com 10.0.0.54
curl --cacert /etc/genestack/public-endpoint-ca.crt \
  --resolve keystone.api.openstack.example.com:443:10.0.0.200 \
  https://keystone.api.openstack.example.com/v3      # phải thành công mà không cần -k
```

**Tài nguyên.** Một workload cluster nhỏ nhất (1 control-plane, 1 worker, flavor `m1.medium`) tốn 2 VM × 4 GB. Phương án B tốn thêm 3 VM × 4 vCPU / 4 GB cho management cluster. Ba compute của lab có tổng 48 GB RAM. Compute chạy QEMU không có KVM (Phần 2), nên mọi VM Kubernetes đều chạy chậm. Hãy chờ lâu hơn các con số trong tài liệu, và đừng ngạc nhiên nếu kubeadm phải thử lại vài lần.

## Bước 1 – Management cluster: chọn chỗ chạy

Driver `magnum-capi-helm` chỉ cần một kubeconfig trỏ tới một cluster Kubernetes có cài Cluster API. Cluster đó có thể là:

- **Phương án A: chính cluster Genestack.** Các controller CAPI chạy chung với control plane của OpenStack.
- **Phương án B: một cluster riêng.** Genestack có sẵn playbook dựng nó trên chính cloud: 3 VM, LB Octavia, Kubespray.

Tài liệu Genestack chỉ hỗ trợ phương án B. Tài liệu ghi rõ management cluster *"is **not** the existing Genestack Kubernetes cluster"*. Phương án A không phải ý tưởng lạ: có những bản phân phối OpenStack chạy CAPI ngay trong cluster control plane. Nhưng chọn A với Genestack nghĩa là bạn tự chịu phần tích hợp.

### So sánh

| | A – Dùng cluster Genestack | B – Cluster riêng |
|---|---|---|
| Tài nguyên thêm | Vài pod controller, cỡ vài trăm MB RAM | 3 VM × 4 vCPU / 4 GB, 50 GB volume, 1 LB Octavia |
| Thêm một cluster Kubernetes phải vận hành | Không: dùng chung quy trình nâng cấp, backup của Phần 3 và Phần 9 | Có: Kubespray riêng, nâng cấp riêng, backup etcd riêng |
| Magnum gọi tới mgmt cluster | Trong cluster, qua `https://kubernetes.default.svc` | Qua VIP trên `public`: thêm một đường mạng và một LB phải luôn chạy |
| Mgmt cluster gọi API workload cluster (floating IP) | Từ node Genestack, qua router của hạ tầng | Qua router của `capi-mgmt-net` |
| DNS cho `*.api.openstack.example.com` | Phải thêm zone cho CoreDNS và nodelocaldns của Genestack | Đặt qua biến của playbook |
| Quyền của Magnum và CAPI | Có quyền rộng **trên cluster đang chạy OpenStack** | Chỉ trong mgmt cluster |
| Phiên bản và thành phần dùng chung | Chung cert-manager, chung nhịp nâng cấp Kubernetes, chung CRD | Độc lập |
| Miền lỗi | Bảo trì cluster Genestack thì CAPI cũng dừng | Mgmt cluster chạy trên Nova/Cinder/Octavia của chính cloud |
| Được Genestack hỗ trợ | Không | Có |

### Phân tích từng điểm

**1. Bảo mật: điểm nặng nhất của phương án A.**
- `clusterctl init` cài controller với ClusterRole rất rộng, gồm cả đọc và ghi **Secret trên toàn cluster**. Trong cluster Genestack, namespace `openstack` chứa mật khẩu admin Keystone, root MariaDB, RabbitMQ… Một lỗ hổng ở CAPI, CAPO hay addon-provider sẽ thành lỗ hổng của cả cloud. Addon-provider còn chạy Helm với chart lấy từ Internet.
- Magnum cần một credential tới cluster này. Driver tự tạo namespace `magnum-<project-id>` khi cần, và RBAC của Kubernetes không giới hạn được quyền theo tiền tố tên namespace. Vì vậy credential của Magnum thực tế phải có quyền gần như cluster-admin. Chiếm được `magnum-conductor` là chiếm được cả cloud.
- Application credential của mọi tenant (trong `clouds.yaml` mà Magnum tạo) nằm trong etcd của cluster Genestack.

**2. Ràng buộc phiên bản.**
- **cert-manager:** Genestack quản lý cert-manager (v1.19.5) bằng Helm, và cert-manager đang cấp chứng chỉ cho gateway của toàn bộ API (Phần 4a). `clusterctl init` thấy cert-manager có sẵn thì dùng lại, không cài đè. Nhưng sau này mỗi lần nâng CAPI phải kiểm tra lại yêu cầu về phiên bản cert-manager. Mỗi lần nâng cert-manager cho Genestack cũng phải nghĩ tới CAPI.
- **Kubernetes:** mỗi lần nâng Kubernetes của Genestack phải kiểm tra thêm ma trận tương thích của CAPI và CAPO, và ngược lại. Hiện tại thì ổn: playbook của chính Genestack dựng mgmt cluster Kubernetes 1.35 với clusterctl v1.11.3, giống cluster Genestack (v1.35.4).

**3. Miền lỗi.**
- Phương án A: khi cluster Genestack gặp sự cố hoặc đang bảo trì (thay controller như Phần 9, etcd chậm), CAPI ngừng reconcile. Auto-healing (MachineHealthCheck), resize và upgrade của mọi workload cluster dừng theo. Chiều ngược lại ít rủi ro hơn: webhook của CAPI chỉ áp lên resource của CAPI. Nguy cơ thực sự nằm ở thành phần dùng chung như cert-manager.
- Cần công bằng: phương án B cũng không độc lập hoàn toàn. Mgmt cluster chạy trên chính Nova, Cinder, Octavia của cloud này. Khi OpenStack hỏng, CAPI dù ở đâu cũng không làm được gì. Trong mọi trường hợp, workload cluster của tenant **vẫn chạy bình thường** khi mgmt cluster ngừng, vì mgmt cluster chỉ lo vòng đời cluster.

**4. Mạng.** Phương án A bỏ được VM mgmt, `capi-mgmt-net`, LB của mgmt, và đường Magnum → VIP của mgmt. Đổi lại, node Genestack vẫn phải tới được `192.168.10.0/24` để gọi API của workload cluster, và phải thêm DNS cho pod.

**5. Quy mô.** Mỗi workload cluster tạo ra vài chục object CAPI trong etcd, và addon-provider chạy Helm cho từng cluster. Ở mức vài chục cluster thì không đáng kể. Ở mức hàng trăm, CAPI cạnh tranh tài nguyên với control plane OpenStack trên cùng các controller.

### Khuyến nghị

- **Lab: phương án A.** Controller của lab chỉ có 16 GB RAM, còn compute chạy QEMU không có KVM. Thêm 3 VM mgmt và một lần Kubespray nữa là phần nặng và dễ hỏng nhất. Phương án A loại bỏ được phần đó.
- **Production: phương án B**, như Genestack khuyến nghị. Lý do chính là bảo mật (mục 1) và tách nhịp nâng cấp (mục 2), không phải tính sẵn sàng.

Làm một trong hai mục 1A hoặc 1B dưới đây. Cả hai đều kết thúc bằng cùng một kết quả: biến `MGMT_KUBECONFIG` trỏ tới kubeconfig của mgmt cluster dùng trên deployer, và file `magnum-helm-overrides-capi.yaml` để Magnum dùng ở bước 2.

## Bước 1A – Lab: dùng cluster Genestack làm management cluster

### DNS cho pod

Kubespray bật **nodelocaldns**. Mọi tên ngoài `cluster.local` được nodelocaldns chuyển thẳng tới DNS upstream (trong lab là `208.67.222.222`, `8.8.8.8`), không đi qua CoreDNS. Vì vậy chỉ sửa ConfigMap của CoreDNS là không đủ. Hãy khai báo zone qua biến của Kubespray, để cấu hình còn nguyên khi chạy lại `cluster.yml` (Phần 8, Phần 9):

```bash
cat > /etc/genestack/inventory/group_vars/k8s_cluster/zz-dns-zones.yml <<'EOF'
coredns_external_zones:
  - zones: [api.openstack.example.com]
    nameservers: [10.0.0.54]
    cache: 30
nodelocaldns_external_zones:
  - zones: [api.openstack.example.com]
    nameservers: [10.0.0.54]
    cache: 30
EOF

cd /opt/genestack/submodules/kubespray
ansible-playbook cluster.yml --become --tags coredns,nodelocaldns

kubectl -n kube-system get cm nodelocaldns -o jsonpath='{.data.Corefile}' | grep -A3 api.openstack
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- \
  nslookup keystone.api.openstack.example.com
```

`10.0.0.54` là DNS của lab (xem phần điều kiện cần).

### Cài các thành phần Cluster API

Cài cùng các phiên bản mà playbook của Genestack dùng, và **ghim phiên bản** để `clusterctl` không tự kéo bản mới nhất:

```bash
curl -L -o /usr/local/bin/clusterctl \
  https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.11.3/clusterctl-linux-amd64
chmod +x /usr/local/bin/clusterctl

export EXP_KUBEADM_BOOTSTRAP_FORMAT_IGNITION=true
export EXP_BOOTSTRAP_FORMAT_IGNITION=true
export CLUSTER_TOPOLOGY=true
clusterctl init \
  --core cluster-api:v1.11.3 \
  --bootstrap kubeadm:v1.11.3 \
  --control-plane kubeadm:v1.11.3 \
  --infrastructure openstack:v0.14.0
```

Output phải có dòng báo **bỏ qua cài cert-manager** vì đã có sẵn. Nếu `clusterctl` định cài một cert-manager khác, dừng lại và kiểm tra: không để nó đè lên cert-manager mà gateway đang dùng.

ORC và addon-provider:

```bash
kubectl apply -f https://github.com/k-orc/openstack-resource-controller/releases/latest/download/install.yaml

helm repo add capi-addons https://azimuth-cloud.github.io/cluster-api-addon-provider
helm upgrade --install cluster-api-addon-provider capi-addons/cluster-api-addon-provider \
  --namespace capi-addon-system --create-namespace
```

Playbook của Genestack cũng dùng `releases/latest` cho ORC. Muốn lặp lại chính xác, thay `latest` bằng một tag cụ thể.

Mặc định các Deployment này có thể chạy trên **compute node**, vì compute cũng là node của cluster. Ghim chúng về controller:

```bash
for ns in capi-system capi-kubeadm-bootstrap-system capi-kubeadm-control-plane-system \
          capo-system orc-system capi-addon-system; do
  for d in $(kubectl -n $ns get deploy -o name); do
    kubectl -n $ns patch $d --type merge -p \
      '{"spec":{"template":{"spec":{"nodeSelector":{"openstack-control-plane":"enabled"}}}}}'
  done
done
kubectl get providers -A
kubectl get pods -o wide -A | grep -E 'capi|capo|orc'
```

`clusterctl upgrade` sẽ ghi lại manifest của provider và làm mất patch này. Hãy chạy lại vòng lặp sau mỗi lần nâng cấp.

### Credential cho Magnum

Không đưa `admin.conf` vào Magnum. Tạo một user riêng, `magnum-capi`, xác thực bằng client certificate. Template kubeconfig của chart Magnum chỉ nhận client certificate, không nhận token, nên cấp cert qua CSR API của Kubernetes:

```bash
mkdir -p /etc/genestack/magnum-capi && cd /etc/genestack/magnum-capi
openssl genrsa -out magnum-capi.key 2048
openssl req -new -key magnum-capi.key -subj "/CN=magnum-capi" -out magnum-capi.csr

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: magnum-capi
spec:
  request: $(base64 -w0 magnum-capi.csr)
  signerName: kubernetes.io/kube-apiserver-client
  usages: [client auth]
EOF
kubectl certificate approve magnum-capi
kubectl get csr magnum-capi -o jsonpath='{.status.certificate}' | base64 -d > magnum-capi.crt
openssl x509 -in magnum-capi.crt -noout -subject -enddate
chmod 600 magnum-capi.key

kubectl create clusterrolebinding magnum-capi --clusterrole=cluster-admin --user=magnum-capi
```

Chứng chỉ có hạn dùng theo `cluster-signing-duration` của kube-controller-manager, mặc định là 1 năm. Hãy ghi nhớ ngày hết hạn và cấp lại trước khi hết.

User này được gán `cluster-admin`. Như đã phân tích ở mục 1, có viết ClusterRole riêng thì vẫn phải cấp quyền tạo namespace và ghi Secret trên toàn cluster. Gán `cluster-admin` cho một user **riêng** ít nhất giúp nhận diện được trong audit log, và thu hồi được ngay bằng cách xoá ClusterRoleBinding, mà không đụng tới admin của cluster.

Tạo kubeconfig cho user này. Magnum chạy trong cluster, nên trỏ thẳng tới `https://kubernetes.default.svc`. Tên này có trong chứng chỉ của API server:

```bash
CA_DATA=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
CRT=$(base64 -w0 magnum-capi.crt); KEY=$(base64 -w0 magnum-capi.key)

cat > /etc/genestack/helm-configs/magnum/magnum-helm-overrides-capi.yaml <<EOF
conf:
  capi:
    enabled: true
    clusterName: genestack
    apiServer: https://kubernetes.default.svc
    certificateAuthorityData: ${CA_DATA}
    contextName: magnum-capi@genestack
    userName: magnum-capi
    clientCertificateData: ${CRT}
    clientKeyData: ${KEY}
EOF
chmod 600 /etc/genestack/helm-configs/magnum/magnum-helm-overrides-capi.yaml

# Trên deployer, mgmt cluster chính là cluster Genestack
export MGMT_KUBECONFIG=/root/.kube/config
kubectl auth can-i create namespaces --as magnum-capi     # yes
```

Phương án A dùng chung etcd của cluster Genestack. Snapshot etcd trong quy trình backup ở Phần 9 đã chứa luôn trạng thái của CAPI, không cần backup riêng.

## Bước 1B – Production: dựng management cluster riêng

Playbook `capi-mgmt-cluster-main.yaml` làm toàn bộ việc này, trong ba play:

1. Bằng tài khoản admin: tạo project `capi-mgmt-cluster-project` và user `capi-mgmt-user` trong domain `service`, gán role `admin`, đặt quota.
2. Bằng user mới (role `capi_cluster`): tạo `capi-mgmt-net` (`172.16.51.0/24`) và router ra mạng external, security group, 3 port, image Ubuntu 24.04, flavor `m1.capi-mgmt-cluster-flavor` (4 vCPU, 4 GB), keypair, volume, LB Octavia cho cổng 6443 với VIP trên mạng external, server group anti-affinity, 3 VM boot từ volume, và một floating IP cho VM đầu tiên.
3. SSH vào VM đầu tiên và chạy script cài: Kubespray (Kubernetes 1.35), Helm, `clusterctl init --infrastructure openstack:v0.14.0`, ORC, `cluster-api-addon-provider`, cron backup etcd 6 giờ một lần. Cuối cùng kéo `admin.conf` về deployer.

### Ghi đè biến

File `/opt/genestack/ansible/playbooks/vars/main.yml` có vài giá trị **viết cứng cho môi trường của Rackspace**, trong đó có `os_auth_url` và cả `OS_AUTH_URL` bên trong dict `capi_mgmt_env` (không lấy từ `os_auth_url`). Đừng sửa file trong `/opt/genestack`. Hãy tạo một file biến riêng trong `/etc/genestack` và truyền bằng `-e @file`: extra vars có độ ưu tiên cao nhất, đè được cả `vars_files` của playbook.

```bash
cat > /etc/genestack/capi-mgmt-vars.yml <<'EOF'
os_auth_url: https://keystone.api.openstack.example.com/v3

capi_mgmt_env:
  OS_ENDPOINT_TYPE: public
  OS_INTERFACE: public
  OS_AUTH_URL: https://keystone.api.openstack.example.com/v3
  OS_USERNAME: capi-mgmt-user
  OS_PASSWORD: "{{ os_user_password }}"
  OS_PROJECT_NAME: capi-mgmt-cluster-project
  OS_USER_DOMAIN_NAME: service
  OS_PROJECT_DOMAIN_NAME: service
  OS_AUTH_TYPE: password
  OS_REGION_NAME: RegionOne
  OS_IDENTITY_API_VERSION: 3
  OS_AUTH_VERSION: 3
  NOVA_ENDPOINT_TYPE: public
  OS_CACERT: /etc/genestack/public-endpoint-ca.crt
  KUBECONFIG: /root/.kube/config

capi_mgmt_cluster_volume_type: lvm-ssd
capi_mgmt_dns_servers: "10.0.0.54"
capi_mgmt_dns_forwarders:
  - 10.0.0.54
EOF
chmod 600 /etc/genestack/capi-mgmt-vars.yml
```

Giải thích các giá trị:

- `capi_mgmt_cluster_volume_type`: mặc định là `lvmdriver-1`, lab chỉ có `lvm-ssd` (Phần 6). Playbook kiểm tra volume type có tồn tại trước khi chạy.
- `capi_mgmt_dns_servers` và `capi_mgmt_dns_forwarders`: mặc định là `10.239.0.55`, DNS nội bộ của Rackspace. Ở đây dùng DNS `10.0.0.54` của lab. Ở production, thay bằng DNS thật của bạn. Giá trị đầu được đặt cho `capi-mgmt-subnet`, giá trị sau làm upstream cho CoreDNS và nodelocaldns của management cluster.
- `OS_CACERT` trong `capi_mgmt_env` giúp các module `openstack.cloud` tin CA của gateway (xem phần điều kiện cần). Play đầu tiên (bằng admin) dùng biến môi trường của shell, nên export `OS_CACERT` trước khi chạy. Nếu dùng chứng chỉ từ CA công cộng thì bỏ biến này.

Deployer cũng phải phân giải được `*.api.openstack.example.com`. Nếu đã thêm `/etc/hosts` ở Phần 10 thì không cần làm gì thêm.

### Chạy playbook

```bash
export OS_CACERT=/etc/genestack/public-endpoint-ca.crt
EXT_NET_ID=$(os network show public -c id -f value)

cd /opt/genestack/ansible/playbooks
ansible-playbook capi-mgmt-cluster-main.yaml \
  -e @/etc/genestack/capi-mgmt-vars.yml \
  -e os_admin_password="$(kubectl -n openstack get secret keystone-admin -o jsonpath='{.data.password}' | base64 -d)" \
  -e os_user_password="$(openssl rand -hex 16 | tee /root/.capi-mgmt-user-password)" \
  -e ext_net_id="$EXT_NET_ID"
```

Play thứ ba chạy script cài đặt ở chế độ `async` trên VM đầu tiên, và chờ tối đa 30 phút (180 lần × 10 giây). Trong lúc chờ, theo dõi log trên VM đó:

```bash
FIP=$(os floating ip list --project capi-mgmt-cluster-project --project-domain service \
  -c "Floating IP Address" -f value | head -1)
ssh -i ~/capi_mgmt_private_key ubuntu@$FIP sudo tail -f /var/log/capi-mgmt-cluster-install.log
```

Vài điều cần biết về script cài đặt:

- **Kubespray được clone từ nhánh `master`**, không gắn tag (dòng `git checkout tags/...` bị comment). Mỗi lần dựng có thể ra một phiên bản Kubespray khác nhau. Nếu cần lặp lại chính xác, clone và checkout tag trên VM trước khi chạy playbook.
- Script **không tự sửa** cluster dở dang: nếu đã cài nhưng chưa đủ 3 node `Ready`, nó dừng và yêu cầu kiểm tra bằng tay.
- `clusterctl init` không ghim phiên bản core CAPI, nên sẽ lấy bản mới nhất tương thích với clusterctl v1.11.3. Feature gate `ClusterTopology` và bootstrap dạng Ignition (cho image Flatcar) được bật.

### Kiểm tra management cluster

Kubeconfig được kéo về `/var/tmp/capi_mgmt_cluster.kubeconfig`. Trường `server` trong file trỏ tới `https://capi-mgmt-lb.cluster.local:6443`, tên mà chỉ các VM của management cluster phân giải được (qua `/etc/hosts` do script thêm vào). Đổi sang IP của VIP. IP này đã nằm trong chứng chỉ của API server nhờ `supplementary_addresses_in_ssl_keys`:

```bash
MGMT_VIP=$(os loadbalancer show capi-mgmt-cluster-lb -c vip_address -f value)
export MGMT_KUBECONFIG=/etc/genestack/capi-mgmt.kubeconfig
sed "s#https://capi-mgmt-lb.cluster.local:6443#https://$MGMT_VIP:6443#" \
  /var/tmp/capi_mgmt_cluster.kubeconfig > $MGMT_KUBECONFIG
chmod 600 $MGMT_KUBECONFIG

kubectl --kubeconfig $MGMT_KUBECONFIG get nodes -o wide
kubectl --kubeconfig $MGMT_KUBECONFIG get providers -A
kubectl --kubeconfig $MGMT_KUBECONFIG get pods -A | grep -E 'capi|capo|orc|addon'
```

Mong đợi: 3 node `Ready`, và các provider `cluster-api`, `bootstrap-kubeadm`, `control-plane-kubeadm`, `infrastructure-openstack`.

Việc đổi sang IP còn có một lý do quan trọng hơn: kubeconfig này sẽ được đưa vào pod `magnum-conductor` trong cluster Genestack. Ở đó, mọi tên kết thúc bằng `cluster.local` được CoreDNS coi là tên **nội bộ của chính cluster Genestack**, nên `capi-mgmt-lb.cluster.local` sẽ không bao giờ phân giải đúng.

Kiểm tra luôn đường đi từ một controller tới VIP:

```bash
ssh openstack-controller-1 curl -sk https://$MGMT_VIP:6443/version
```

Tạo file override cho Magnum từ kubeconfig vừa sửa:

```bash
kc() { kubectl --kubeconfig $MGMT_KUBECONFIG config view --raw -o jsonpath="$1"; }

cat > /etc/genestack/helm-configs/magnum/magnum-helm-overrides-capi.yaml <<EOF
conf:
  capi:
    enabled: true
    clusterName: $(kc '{.clusters[0].name}')
    apiServer: $(kc '{.clusters[0].cluster.server}')
    certificateAuthorityData: $(kc '{.clusters[0].cluster.certificate-authority-data}')
    contextName: $(kc '{.contexts[0].name}')
    userName: $(kc '{.users[0].name}')
    clientCertificateData: $(kc '{.users[0].user.client-certificate-data}')
    clientKeyData: $(kc '{.users[0].user.client-key-data}')
EOF
chmod 600 /etc/genestack/helm-configs/magnum/magnum-helm-overrides-capi.yaml
grep apiServer /etc/genestack/helm-configs/magnum/magnum-helm-overrides-capi.yaml   # phải là IP của VIP
```

File này chứa client key với quyền cluster-admin trên management cluster. Hãy bảo vệ nó như bảo vệ chính kubeconfig.

Management cluster giữ toàn bộ trạng thái của mọi workload cluster. Script đã đặt cron backup etcd vào `/var/backups` trên từng VM, nhưng thư mục đó vẫn nằm trong chính cloud này. Hãy định kỳ chép bản backup ra ngoài.

## Bước 2 – Cài Magnum

### Override cho Magnum

File `magnum-helm-overrides-capi.yaml` đã được tạo ở bước 1A hoặc 1B. Chart Magnum dựng kubeconfig từ các trường `conf.capi`, lưu vào Secret `magnum-etc`, và mount vào cả `magnum-api` lẫn `magnum-conductor` tại `/etc/magnum/kubeconfig.conf`.

Thêm một file override thứ hai cho CA và endpoint. File này giống nhau cho cả hai phương án:

```bash
cat > /etc/genestack/helm-configs/magnum/magnum-lab-overrides.yaml <<'EOF'
conf:
  magnum:
    drivers:
      verify_ca: true
      openstack_ca_file: /etc/magnum/openstack-ca/ca.crt
    keystone_authtoken:
      interface: internal
      cafile: /etc/magnum/openstack-ca/ca.crt
    keystone_auth:
      cafile: /etc/magnum/openstack-ca/ca.crt
    barbican_client: {endpoint_type: internalURL}
    cinder_client: {endpoint_type: internalURL}
    glance_client: {endpoint_type: internalURL}
    heat_client: {endpoint_type: internalURL}
    magnum_client: {endpoint_type: internalURL}
    neutron_client: {endpoint_type: internalURL}
    nova_client: {endpoint_type: internalURL}
    octavia_client: {endpoint_type: internalURL}
pod:
  mounts:
    magnum_api:
      magnum_api:
        volumeMounts: &ca_mounts
          - name: openstack-ca
            mountPath: /etc/magnum/openstack-ca
            readOnly: true
        volumes: &ca_volumes
          - name: openstack-ca
            secret:
              secretName: openstack-public-ca
    magnum_conductor:
      magnum_conductor:
        volumeMounts: *ca_mounts
        volumes: *ca_volumes
EOF
```

Mỗi phần có một lý do:

- **`[drivers] openstack_ca_file` và `verify_ca`.** Khi tạo cluster, driver đọc CA từ `openstack_ca_file` và ghi vào Secret chứa `clouds.yaml` cho cluster đó (khoá `cacert`), cùng với `verify: true`. CAPO trên management cluster, CCM và Cinder CSI trong workload cluster đều dùng Secret này. Không đặt `openstack_ca_file` thì driver dùng bộ CA mặc định của Python (`certifi`), và mọi lời gọi tới endpoint dùng CA nội bộ đều lỗi. Nếu gateway dùng chứng chỉ từ CA công cộng, bỏ phần này và phần `pod.mounts`.
- **`cafile` trong `[keystone_authtoken]` và `[keystone_auth]`.** Magnum dùng `keystoneclient` để tạo trustee user và trust cho mỗi cluster (`cluster_user_trust: true`). Client này được tạo chỉ với session, không truyền `interface`, nên mặc định lấy endpoint Keystone **public** từ catalog, bất kể `auth_url` là internal. Session của domain admin (tạo trustee) lấy CA từ `[keystone_authtoken] cafile`, còn session tạo trust lấy từ `[keystone_auth] cafile`. Thiếu hai dòng này, cluster lỗi ngay với `Failed to create trustee or trust for Cluster`, và log conductor báo `CERTIFICATE_VERIFY_FAILED` khi gọi `https://keystone.api.../v3/users`. CA được mount vào cả `magnum-api` lẫn `magnum-conductor`, vì cả hai đọc cùng `[keystone_authtoken]`.
- **Endpoint `internalURL`.** Base override của Genestack cho Magnum gọi các service khác qua `publicURL`, và xác thực token qua `interface: public`. Trong lab, gọi qua endpoint public nghĩa là pod phải tin CA ở mọi client. Chuyển sang endpoint internal (HTTP, qua Service trong cluster) giúp tránh việc đó, và cũng là cách các service khác của Genestack vẫn làm. `auth_url` mà driver ghi vào `clouds.yaml` cho cluster vẫn luôn là endpoint **public**: driver lấy nó từ catalog với `interface="public"`.

**Chú ý tên section.** `values.yaml` của chart có sẵn một section `capi-helm` (gạch ngang), còn driver đọc section `capi_helm` (gạch dưới). Base override của Genestack đặt đúng `conf.magnum.capi_helm`: `kubeconfig_file`, `helm_chart_repo: https://rackerlabs.github.io/genestack-capi-helm-charts`, `default_helm_chart_version: 0.1.1`. Nếu cần ghi đè, hãy dùng `capi_helm`.

### Chạy cài đặt

```bash
/opt/genestack/bin/install-magnum.sh
```

Script theo khuôn chung (Phần 5): chart `2025.1.4+4d4d4e25c` theo `helm-chart-versions.yaml`, mật khẩu lấy từ Secret qua `--set`. Post-renderer kustomize thêm các CR cho database và RabbitMQ, cùng HPA cho `magnum-api`.

```bash
kubectl -n openstack get pods | grep magnum      # magnum-api-*, magnum-conductor-*, các job Completed
os coe service list                               # magnum-conductor: up
kubectl -n openstack exec magnum-conductor-0 -- ls /etc/magnum/kubeconfig.conf /etc/magnum/openstack-ca/ca.crt
kubectl -n openstack exec magnum-conductor-0 -- grep -A4 '^\[capi_helm\]' /etc/magnum/magnum.conf
```

Job `magnum-domain-ks-user` tạo domain `magnum` và user quản trị của domain đó. Magnum dùng chúng để tạo trust (`cluster_user_trust: true`).

## Bước 3 – Image cho node Kubernetes

Driver chỉ nhận cluster template khi image có hai thuộc tính:

| Thuộc tính | Giá trị | Vai trò |
|---|---|---|
| `os_distro` | `ubuntu` hoặc `flatcar` | Chọn driver: `magnum-capi-helm` khai báo hỗ trợ `(vm, ubuntu, kubernetes)` và `(vm, flatcar, kubernetes)` |
| `kube_version` | Ví dụ `v1.34.12` | Phiên bản Kubernetes trong image, truyền vào chart |

Image phải có sẵn `kubeadm`, `kubelet`, `containerd`: CAPI không cài Kubernetes lúc boot. Có hai cách lấy image:

- **Build bằng [image-builder](https://github.com/kubernetes-sigs/image-builder)** của Kubernetes SIG, như tài liệu Genestack hướng dẫn (`make build-qemu-flatcar` với `OEM_ID=openstack`, hoặc target Ubuntu tương ứng). Việc build cần một máy có KVM. Compute của lab không có KVM, nên hãy build trên một máy khác rồi upload lên.
- **Dùng image dựng sẵn** của dự án [azimuth-images](https://github.com/azimuth-cloud/azimuth-images/releases), cùng nhóm phát triển chart `openstack-cluster` mà driver dùng. Lab dùng cách này.

Mỗi release của azimuth-images có file `manifest.json` liệt kê image và phiên bản Kubernetes. Ví dụ với release `0.38.0`:

```bash
curl -sL https://github.com/azimuth-cloud/azimuth-images/releases/download/0.38.0/manifest.json \
  | jq -r 'to_entries[] | select(.key | startswith("kubernetes")) | "\(.value.kubernetes_version)  \(.value.url)"'
```

| Kubernetes | Image (Ubuntu 24.04) |
|---|---|
| v1.34.12 | `https://azimuth-images.stackhpc.cloud/ubuntu-noble-kube-v1.34.12-261022-1527.qcow2` |
| v1.35.9 | `https://azimuth-images.stackhpc.cloud/ubuntu-noble-kube-v1.35.9-261022-1527.qcow2` |
| v1.36.5 | `https://azimuth-images.stackhpc.cloud/ubuntu-noble-kube-v1.36.5-261022-1528.qcow2` |
| v1.37.1 | `https://azimuth-images.stackhpc.cloud/ubuntu-noble-kube-v1.37.1-261022-1526.qcow2` |

Dù chọn cách nào, phiên bản Kubernetes phải nằm trong dải mà chart `openstack-cluster` đang dùng hỗ trợ. `default_helm_chart_version` của Genestack là 0.1.1, phát hành từ 11/2025, và repo đã có 0.1.2. Lab bắt đầu với **v1.34**, bản cũ nhất trong release này. Khi cluster đầu tiên đã chạy, hãy thử bản mới hơn cùng chart 0.1.2.

Image cần ở chế độ `--public` để người dùng ở mọi project tạo được cluster. Policy Glance của Genestack chỉ cho role **`glance_admin`** publicize image (`publicize_image: role:glance_admin`), và role `admin` **không đủ**. Lab chưa có role này, nên tạo role rồi gán cho admin trong project admin:

```bash
os role create glance_admin
os role add --user admin --project admin glance_admin
```

Nếu chỉ test trong project admin, có thể bỏ `--public` ở lệnh dưới. Image private vẫn dùng được cho cluster trong cùng project.

Tải image về deployer và upload. Mỗi file khoảng 4 GB, nên kiểm tra pod admin client còn đủ chỗ trong `/tmp` trước khi chép:

```bash
cd /var/tmp
IMG=ubuntu-noble-kube-v1.34.12-261022-1527
curl -fL -C - --retry 5 -O https://azimuth-images.stackhpc.cloud/$IMG.qcow2
# Checksum lấy từ manifest.json của release
SUM=$(curl -sL https://github.com/azimuth-cloud/azimuth-images/releases/download/0.38.0/manifest.json \
  | jq -r '.[] | select(.name=="'$IMG'") | .checksum' | cut -d: -f2)
echo "$SUM  $IMG.qcow2" | sha256sum -c -        # phải in: OK

kubectl -n openstack cp $IMG.qcow2 openstack-admin-client:/tmp/k8s.qcow2
os image create ubuntu-noble-kube-v1.34 \
  --disk-format qcow2 --container-format bare --public \
  --file /tmp/k8s.qcow2 \
  --property os_distro=ubuntu --property os_version=24.04 \
  --property kube_version=v1.34.12
kubectl -n openstack exec openstack-admin-client -- rm -f /tmp/k8s.qcow2
```

**Đừng bỏ qua bước kiểm tra checksum.** Nếu tải bị đứt giữa chừng, Glance vẫn nhận file qcow2 cụt mà không báo lỗi. Lỗi chỉ lộ ra khi Cinder tạo boot volume: `qemu-img: error while reading at byte ...: Input/output error` trong log `cinder-volume`, volume chuyển `error`, và `OpenStackServer` báo `volume ... is in error state`. So sánh `size` của image trong Glance với dung lượng thật của file (khoảng 4 GB) cũng là một cách kiểm tra nhanh.

Giá trị `kube_version` phải khớp với binary trong image (ở đây là `v1.34.12`, lấy từ tên file hoặc `manifest.json`), nếu không kubeadm sẽ lỗi khi join node.

## Bước 4 – Cluster template và cluster đầu tiên

```bash
os coe cluster template create k8s-v1-34 \
  --coe kubernetes \
  --image ubuntu-noble-kube-v1.34 \
  --external-network public \
  --dns-nameserver 10.0.0.54 \
  --master-flavor m1.medium --flavor m1.medium \
  --network-driver calico \
  --master-lb-enabled --floating-ip-disabled \
  --labels fixed_subnet_cidr=10.20.0.0/24,octavia_provider=ovn,boot_volume_type=lvm-ssd,boot_volume_size=20,kube_dashboard_enabled=false,monitoring_enabled=false
```

Các lựa chọn cho lab:

| Tham số | Giá trị | Lý do |
|---|---|---|
| `fixed_subnet_cidr` | `10.20.0.0/24` | **Bắt buộc phải đổi trong lab.** Mặc định của driver là `10.0.0.0/24`, trùng với `br-host`. VM của cluster sẽ coi `10.0.0.200` là địa chỉ cùng subnet, gửi ARP thay vì đi qua router, và không bao giờ tới được endpoint public |
| `octavia_provider` | `ovn` | LB cho API (TCP 6443) không cần L7. Provider `ovn` không tốn VM amphora và tạo xong trong vài giây (Phần 10) |
| `--dns-nameserver` | `10.0.0.54` | DNS của lab, phân giải được `*.api.openstack.example.com` |
| `--master-lb-enabled` | | API của cluster đứng sau LB có floating IP (`master_lb_floating_ip_enabled` mặc định bật). Management cluster gọi API của workload cluster qua địa chỉ này |
| `--floating-ip-disabled` | | Node không cần floating IP riêng: ra ngoài qua SNAT của router |
| `boot_volume_*` | `lvm-ssd`, 20 GB | Node boot từ volume Cinder |
| `kube_dashboard_enabled`, `monitoring_enabled` | `false` | Bớt addon cho lab nhỏ |

Driver yêu cầu flavor tối thiểu 2 vCPU và 2048 MB RAM (`minimum_flavor_vcpus`, `minimum_flavor_ram`). `m1.medium` (2 vCPU, 4 GB) vừa đủ. `m1.small` (1 vCPU) sẽ bị từ chối.

Tạo cluster nhỏ nhất (1 control-plane, 1 worker) trong project của người dùng:

```bash
os coe cluster create k8s-lab --cluster-template k8s-v1-34 \
  --master-count 1 --node-count 1 --keypair <keypair>
os coe cluster show k8s-lab -c status -c status_reason -c health_status
```

### Theo dõi trên management cluster

Trạng thái trong Magnum chỉ là bản tóm tắt. Muốn biết cluster đang kẹt ở đâu, xem trên management cluster. `MGMT_KUBECONFIG` đã được đặt ở bước 1A hoặc 1B:

```bash
mk() { kubectl --kubeconfig "$MGMT_KUBECONFIG" "$@"; }
PROJECT_ID=$(os coe cluster show k8s-lab -c project_id -f value)
NS=magnum-$PROJECT_ID

helm --kubeconfig "$MGMT_KUBECONFIG" -n $NS list   # release của cluster
mk -n $NS get cluster,openstackcluster               # network, router, LB API
mk -n $NS get kubeadmcontrolplane,machinedeployment
mk -n $NS get machines,openstackmachines -o wide
mk -n $NS get helmreleases,manifests                 # addon: CNI, CCM, CSI
mk -n capo-system logs deploy/capo-controller-manager --tail=50
```

Hàm `mk` dùng `--kubeconfig` thay vì `export KUBECONFIG`. Ở phương án B, nếu export thì hàm `os` (gọi `kubectl` tới cluster Genestack) sẽ trỏ nhầm sang mgmt cluster.

Gõ từng lệnh như trên khá mất công. Script dưới đây gom chúng lại và làm mới sau mỗi chu kỳ, tới khi cluster `*_COMPLETE` hoặc `*_FAILED`. Mỗi lượt script hiển thị trạng thái Magnum, Helm release, các object CAPI, `OpenStackServer` (VM và volume), addon, các condition đang `False` kèm thông báo, VM và LB trong project, event `Warning`, và lỗi gần nhất của `magnum-conductor`.

```bash
cat > /root/magnum-watch.sh <<'SCRIPT'
#!/usr/bin/env bash
# Theo dõi quá trình tạo/resize/xoá một cluster Magnum (driver magnum-capi-helm).
# Dùng: magnum-watch.sh <tên-cluster> [chu-kỳ-giây]     (--once để in một lần rồi thoát)
# Biến môi trường: MGMT_KUBECONFIG (mặc định /root/.kube/config)
set -uo pipefail

ONCE=0
[[ "${1:-}" == "--once" ]] && { ONCE=1; shift; }
CLUSTER=${1:?Dùng: $0 [--once] <tên-cluster> [chu-kỳ-giây]}
INTERVAL=${2:-20}
MGMT_KUBECONFIG=${MGMT_KUBECONFIG:-/root/.kube/config}

os() { kubectl -n openstack exec -i openstack-admin-client -- openstack "$@" </dev/null 2>/dev/null; }
mk() { kubectl --kubeconfig "$MGMT_KUBECONFIG" "$@" </dev/null 2>/dev/null; }
hdr() { printf '\n\033[1;34m== %s\033[0m\n' "$*"; }
field() { python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('$1') or '')"; }

START=$(date +%s)
PROJECT_ID=$(os coe cluster show "$CLUSTER" -f json | field project_id)
[[ -z "$PROJECT_ID" ]] && { echo "Không tìm thấy cluster $CLUSTER"; exit 1; }
NS=magnum-$PROJECT_ID

while :; do
  JSON=$(os coe cluster show "$CLUSTER" -f json)
  if [[ -z "$JSON" ]]; then
    echo "$(date +%T) Cluster $CLUSTER không còn trong Magnum (đã xoá)."; exit 0
  fi
  STATUS=$(field status <<<"$JSON")

  # Release Helm: <tên-cluster>-<12 ký tự ngẫu nhiên>
  RELEASE=$(helm --kubeconfig "$MGMT_KUBECONFIG" -n "$NS" list -q --filter "^${CLUSTER}-[a-z0-9]{12}\$" 2>/dev/null | head -1)
  SEL="capi.stackhpc.com/cluster=${RELEASE:-none}"          # Cluster, KCP, MD, Machine, addon
  SEL_SRV="cluster.x-k8s.io/cluster-name=${RELEASE:-none}"   # OpenStackServer chỉ có nhãn này

  [[ $ONCE -eq 0 ]] && clear
  ELAPSED=$(( $(date +%s) - START ))
  printf '\033[1m%s  |  cluster %s  |  %s  |  đã theo dõi %dm%02ds\033[0m\n' \
    "$(date +%T)" "$CLUSTER" "$STATUS" $((ELAPSED/60)) $((ELAPSED%60))

  hdr "Magnum"
  python3 -c '
import json,sys; d=json.load(sys.stdin)
for k in ["uuid","status","status_reason","health_status","api_address","master_count","node_count","created_at","updated_at"]:
    print(f"  {k:<15} {d.get(k)}")' <<<"$JSON"

  hdr "Helm release trên mgmt cluster (ns $NS)"
  helm --kubeconfig "$MGMT_KUBECONFIG" -n "$NS" list --filter "^${CLUSTER}-" 2>/dev/null | sed 's/^/  /'

  if [[ -n "$RELEASE" ]]; then
    hdr "Cluster API"
    mk -n "$NS" get cluster,openstackcluster,kubeadmcontrolplane,machinedeployment -l "$SEL" | sed 's/^/  /'

    hdr "Machine"
    mk -n "$NS" get machines -l "$SEL" \
      -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,NODE:.status.nodeRef.name,VERSION:.spec.version' | sed 's/^/  /'
    mk -n "$NS" get openstackservers -l "$SEL_SRV" \
      -o custom-columns='OPENSTACKSERVER:.metadata.name,STATE:.status.instanceState,READY:.status.ready,INSTANCE:.status.instanceID' | sed 's/^/  /'

    hdr "Addon (HelmRelease / Manifests)"
    mk -n "$NS" get helmreleases,manifests -l "$SEL" \
      -o custom-columns='KIND:.kind,NAME:.metadata.labels.capi\.stackhpc\.com/component,PHASE:.status.phase,FAILURE:.status.failureMessage' \
      | sed 's/^/  /'

    hdr "Điều kiện chưa đạt (Cluster / Machine / OpenStackServer)"
    { mk -n "$NS" get cluster,machines -l "$SEL" -o json
      mk -n "$NS" get openstackservers -l "$SEL_SRV" -o json; } | python3 -c '
import json, sys
dec = json.JSONDecoder(); buf = sys.stdin.read(); i = 0
while i < len(buf):
    while i < len(buf) and buf[i].isspace(): i += 1
    if i >= len(buf): break
    doc, i = dec.raw_decode(buf, i)
    for o in doc.get("items", []):
        name = o["kind"] + "/" + o["metadata"]["name"][-34:]
        for c in o.get("status", {}).get("conditions", []):
            if c.get("status") == "False" and c.get("message") and c["type"] not in ("Available", "Deleting", "Paused") and not c["type"].startswith(("Scaling", "Rolling", "Remediat")):
                print("  %s: %s: %s" % (name, c["type"], " ".join(c["message"].split())[:170]))
'
  fi

  hdr "Tài nguyên OpenStack của project"
  os server list --project "$PROJECT_ID" -c Name -c Status -c Networks | grep -E "$CLUSTER|Name" | sed 's/^/  /'
  os loadbalancer list --project "$PROJECT_ID" -c name -c vip_address -c provisioning_status -c operating_status \
    | grep -E "$CLUSTER|name" | sed 's/^/  /'

  hdr "Sự kiện Warning gần nhất (ns $NS)"
  mk -n "$NS" get events --field-selector type=Warning --sort-by=.lastTimestamp \
    -o custom-columns='LAST:.lastTimestamp,OBJECT:.involvedObject.name,REASON:.reason,MESSAGE:.message' \
    | tail -6 | cut -c1-200 | sed 's/^/  /'

  hdr "Lỗi gần nhất của magnum-conductor"
  kubectl -n openstack logs -l application=magnum,component=conductor -c magnum-conductor --since=10m --tail=2000 2>/dev/null \
    | grep -v sqlalchemy | grep -E ' (ERROR|WARNING) ' | grep -vE '^\s*$' | tail -3 | cut -c1-220 | sed 's/^/  /'

  case "$STATUS" in
    *_COMPLETE|*_FAILED)
      printf '\n\033[1mKết thúc: %s sau %dm%02ds\033[0m\n' "$STATUS" $((ELAPSED/60)) $((ELAPSED%60)); exit 0 ;;
  esac
  [[ $ONCE -eq 1 ]] && exit 0
  sleep "$INTERVAL"
done
SCRIPT
chmod +x /root/magnum-watch.sh

/root/magnum-watch.sh k8s-lab            # làm mới mỗi 20 giây
/root/magnum-watch.sh --once k8s-lab     # in một lần
```

Phần "Điều kiện chưa đạt" thường chỉ thẳng ra nguyên nhân. Ví dụ, khi image bị tải cụt, dòng `OpenStackServer/...: InstanceReady: volume ... is in error state` xuất hiện ngay từ phút thứ hai. Nếu chỉ nhìn Magnum, bạn sẽ thấy `CREATE_IN_PROGRESS` cho tới khi hết `create_timeout` (60 phút).

Thứ tự thường thấy: `OpenStackCluster` tạo network, router, LB → VM control-plane boot, kubeadm init → LB API có member → VM worker join → addon được cài → cluster `Ready` → Magnum chuyển sang `CREATE_COMPLETE`. Trên lab nested, toàn bộ quá trình mất lâu hơn đáng kể so với con số 15–20 phút trong tài liệu.

### Dùng cluster

Lệnh `coe cluster config` ghi kubeconfig ra thư mục trong pod admin client. Chép nó về deployer:

```bash
kubectl -n openstack exec openstack-admin-client -- mkdir -p /tmp/k8s-lab
os coe cluster config k8s-lab --dir /tmp/k8s-lab --force
kubectl -n openstack cp openstack-admin-client:/tmp/k8s-lab/config ./k8s-lab.kubeconfig

kubectl --kubeconfig ./k8s-lab.kubeconfig get nodes -o wide
kubectl --kubeconfig ./k8s-lab.kubeconfig get pods -A
kubectl --kubeconfig ./k8s-lab.kubeconfig get storageclass
```

Kiểm tra hai tích hợp với OpenStack: một Service `type: LoadBalancer` phải tạo ra LB Octavia, và một PVC phải tạo ra volume Cinder:

```bash
kubectl --kubeconfig ./k8s-lab.kubeconfig create deployment web --image=nginx
kubectl --kubeconfig ./k8s-lab.kubeconfig expose deployment web --port 80 --type LoadBalancer
kubectl --kubeconfig ./k8s-lab.kubeconfig get svc web -w      # EXTERNAL-IP từ Octavia
os loadbalancer list
```

Các thao tác khác:

```bash
os coe cluster resize k8s-lab 2                                  # 2 worker
os coe cluster upgrade k8s-lab <template-có-image-mới>           # mỗi lần chỉ lên 1 minor version
os coe cluster delete k8s-lab
```

Khi xoá, chart được gỡ và CAPI dọn VM, LB, network, router của cluster. Volume do PVC tạo ra **không** bị xoá theo, vì `csi_cinder_reclaim_policy` mặc định là `Retain`.

## Tóm tắt

- Genestack dùng driver `magnum-capi-helm`: Magnum cài chart `openstack-cluster` lên một **management cluster**, và Cluster API dựng workload cluster bằng app credential của người dùng.
- Management cluster có thể là **chính cluster Genestack** (A) hoặc **một cluster riêng** (B):
  - A rẻ và đơn giản hơn nhiều, hợp với lab. Đổi lại, controller CAPI và credential của Magnum có quyền rộng trên cluster đang chạy OpenStack, dùng chung cert-manager và nhịp nâng cấp, và Genestack không hỗ trợ cách này.
  - B là cách Genestack hỗ trợ và nên dùng cho production. Dựng bằng `capi-mgmt-cluster-main.yaml`. File `vars/main.yml` có giá trị viết cứng của Rackspace, nên phải ghi đè bằng `-e @file`, và `apiServer` phải là **IP** của VIP, không phải tên `*.cluster.local`.
- Mọi bên gọi API của OpenStack qua endpoint **public**, và mgmt cluster phải tới được floating IP của từng workload cluster. Vì vậy cần DNS cho `*.api.openstack.example.com`, và chứng chỉ ký bởi một CA mà mọi bên tin được.
- `[drivers] openstack_ca_file` phân phối CA cho từng cluster. Các client của Magnum dùng endpoint internal.
- Image node phải có Kubernetes sẵn và hai thuộc tính `os_distro`, `kube_version`.
- Trong lab: `fixed_subnet_cidr` không được trùng `10.0.0.0/24`, dùng `octavia_provider=ovn` cho LB của API, flavor tối thiểu 2 vCPU / 2 GB.
