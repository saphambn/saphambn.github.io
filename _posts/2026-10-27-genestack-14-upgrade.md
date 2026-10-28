---
title: "OpenStack trên Kubernetes với Genestack – Phần 13: Nâng cấp OpenStack 2025.1 lên 2026.1"
date: 2026-10-27 10:30:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, upgrade, helm, slurp, operations]
render_with_liquid: false
---

Suốt series, lab chạy Genestack `release-2026.2.0.2` với OpenStack **2025.1 (Epoxy)**. Các bản `release-2026.3.x` chuyển sang OpenStack **2026.1 (Gazpacho)**. Bài này nâng cấp lab lên `release-2026.3.1`, làm bằng tay từng service, để thấy rõ mỗi bước thay đổi gì trong cluster.

Tài liệu nâng cấp của Genestack (`docs/genestack-upgrade.md`) rất ngắn: cập nhật git, rồi "chạy lại các chart như lúc cài". Điều đó đúng về nguyên tắc. Nhưng khi đọc diff giữa hai tag, có vài chỗ mà làm đúng nguyên văn tài liệu vẫn có thể gây sự cố trên lab này: chart không được nâng dù image đã đổi, toàn bộ amphora của Octavia bị failover, Barbican không khởi động được. Bài này đi qua từng điểm đó trước khi bắt tay vào làm.

## Nâng cấp lần này đổi những gì

### OpenStack: nhảy thẳng từ 2025.1 lên 2026.1

2025.1 và 2026.1 đều là bản **SLURP** (Skip-Level Upgrade Release Process). OpenStack hỗ trợ nâng trực tiếp giữa hai bản SLURP liên tiếp, bỏ qua bản ở giữa là 2025.2 (Flamingo). Điều kiện kèm theo: mọi **online data migration** của bản cũ phải chạy xong *trước* khi nâng. Bước 0 sẽ kiểm tra việc này.

### Chart nào đổi version

So sánh `helm-chart-versions.yaml` giữa hai tag, chỉ giữ các chart lab đang dùng:

| Chart | 2026.2.0.2 | 2026.3.1 |
|---|---|---|
| keystone | `2025.1.5+ed289c1cd` | `2026.1.8+db238e7c3` |
| glance | `2025.1.4+ed289c1cd` | `2026.1.9+7cce5ac45` |
| placement | `2025.1.2+0cd784591` | `2026.1.8+50c697fc3` |
| nova | `2025.1.19+ed289c1cd` | `2026.1.8+a2a343968` |
| libvirt | `2025.1.4+ed289c1cd` | `2026.1.29+a34a2f9b9` |
| neutron | `2025.1.17+95bf0bf6e` | `2026.1.9+a2a343968` |
| cinder | `2025.1.13+4d4d4e25c` | `2026.1.9+a2a343968` |
| barbican | `2025.1.5+4d4d4e25c` | `2026.1.8+a659ab8a2` |
| octavia | `2025.1.18+12458c92d` | `2026.1.8+d03dd061a` |
| magnum | `2025.1.4+4d4d4e25c` | `2026.1.5+a659ab8a2` |
| masakari | `2025.2.0+eaefa0173` | `2026.1.4+c0d71bdbf` |
| memcached | `2025.2.2+c31cd6c96` | `2026.1.23+0ee5adeff` |
| skyline | `2025.2.9+c665eedfa` | không đổi |

Các chart hạ tầng **không đổi**: kube-ovn `v1.15.4`, MetalLB, Envoy Gateway, Longhorn, mariadb-operator, kube-prometheus-stack, Grafana, opentelemetry-kube-stack. Vì vậy bài này chỉ nâng lớp OpenStack. Nâng Kubernetes bằng Kubespray là một quy trình riêng (`docs/k8s-kubespray-upgrade.md`).

Danh sách đầy đủ nằm trong `docs/release-2026.3.0.md`, `docs/release-2026.3.1.md` và `docs/product-matrix-2026.3.1.md` của repo. Nên đọc lướt chúng trước khi nâng cấp môi trường thật.

## Ba nơi quyết định version

![Ba nơi quyết định version khi nâng cấp Genestack, và các giai đoạn nâng cấp](/assets/img/genestack/14-upgrade-flow.svg)

Mỗi `install-<service>.sh` lấy thông tin từ ba nơi. Mỗi nơi được cập nhật theo một cách khác nhau:

1. **`/opt/genestack`**: script cài đặt và `base-helm-configs/`. Image tag nằm trong `base-helm-configs`, ví dụ `keystone:2025.1-latest` thành `keystone:2026.1-latest`. Mọi thứ ở đây đổi ngay khi `git checkout` sang tag mới.
2. **`/etc/genestack/helm-chart-versions.yaml`**: version chart mà script truyền cho `helm --version`. `bootstrap.sh` chỉ copy file này **một lần**, khi nó chưa tồn tại. `git checkout` không động tới nó.
3. **Secret trong namespace `openstack`**: mật khẩu, khoá. Script `--set` chúng vào chart.

Điểm thứ hai là cái bẫy lớn nhất. Nếu chỉ làm theo tài liệu (checkout rồi chạy lại script), Helm vẫn dùng **chart 2025.1** nhưng với **image 2026.1** từ `base-helm-configs`. Template của chart cũ chạy code mới: lúc thì lỗi ngay, lúc thì chạy được nhưng thiếu cấu hình mà code mới cần.

## Những thay đổi cần xử lý trên lab

| Thay đổi | Ảnh hưởng tới lab | Xử lý ở |
|---|---|---|
| `helm-chart-versions.yaml` trong `/etc/genestack` không tự cập nhật | Chart cũ, image mới | Giai đoạn 2 |
| `bin/create-secrets.sh` bị bỏ. Mỗi service khai báo secret trong `bin/services/<svc>.yaml`, script tự tạo secret còn thiếu và **không ghi đè** secret đã có | Có thêm vài secret mới, ví dụ `keystone-test-password`. Phần lớn không cần làm gì | – |
| Octavia có secret mới `octavia-heartbeat-key`. Nếu chưa có, script sinh khoá **ngẫu nhiên** | Lab dùng provider amphora. Amphora đang chạy vẫn ký heartbeat bằng khoá cũ (`insecure`, giá trị mặc định của chart), health manager sẽ coi mọi amphora là chết và failover **hàng loạt** | Giai đoạn 3 |
| Barbican 2026.1 bỏ KEK mặc định của `simple_crypto`. Không có KEK thì `barbican-api` không khởi động | Lab chưa từng đặt KEK, tức là đang chạy trên KEK mặc định công khai. Chứng chỉ cluster của Magnum nằm trong Barbican | Giai đoạn 3 |
| Barbican dùng thêm SoftHSM2 (`p11_crypto`) làm backend chính, có PVC RWX `barbican-softhsm-tokens` 100 MiB | PVC dùng StorageClass mặc định (`general`). Longhorn dựng thêm một pod share-manager | Bước Barbican |
| Role `cinder_volumes` được viết lại, nhánh mặc định là `stable/2026.1` | `cinder-volume` trên host vẫn chạy 2025.1 cho tới khi chạy lại playbook | Bước Cinder |
| libvirt-exporter chuyển sang `extraContainers` | DaemonSet libvirt cuộn lại trên mọi compute | Bước libvirt |
| Dọn option cũ: bỏ `idle_timeout`, `heartbeat_in_pthread`, `ovn_l3_mode`. `tenant_network_types` thành `project_network_types`. `cross_az_attach` của Nova chuyển từ `[DEFAULT]` sang `[cinder]` | Override của lab không dùng các option này | Giai đoạn 0 (kiểm tra) |
| Nova có thêm `BackendTrafficPolicy` `nova-volume-attach`, nâng timeout của route `custom-nova-gateway-route` lên 120 giây | Gắn nhiều volume liên tiếp vào một VM không còn bị 504 ở gateway | Tự áp dụng |
| Neutron và Octavia chỉ mount chứng chỉ OVN khi kube-ovn bật `ENABLE_SSL` | Lab dùng `tcp:`, mount thừa được gỡ | Tự áp dụng |

## Giai đoạn 0 – Kiểm tra sức khoẻ, chuẩn bị và backup

Chỉ nâng cấp khi cluster đang khoẻ. Dùng lại bộ lệnh kiểm tra ở [Phần 9](/posts/genestack-10-replace-controller/):

```bash
os() { kubectl -n openstack exec -i openstack-admin-client -- openstack "$@"; }

kubectl get nodes
kubectl -n openstack get mariadb                         # READY=True
kubectl -n openstack exec rabbitmq-server-0 -- rabbitmqctl cluster_status
kubectl -n longhorn-system get volumes.longhorn.io       # mọi volume healthy
kubectl -n openstack get pods | grep -v -E 'Running|Completed'
os compute service list ; os network agent list ; os volume service list
```

### Ghi lại trạng thái trước nâng cấp

```bash
mkdir -p ~/upgrade-2026.3 && cd ~/upgrade-2026.3
helm -n openstack list -o yaml > helm-before.yaml
cp /etc/genestack/helm-chart-versions.yaml helm-chart-versions.before.yaml
os loadbalancer amphora list -c id -c loadbalancer_id -c status -f value > amphora-before.txt
os server list --all-projects -c ID -c Name -c Status -c Host -f value > servers-before.txt
```

`amphora-before.txt` dùng để xác nhận sau này rằng không amphora nào bị thay thế. `helm-before.yaml` cho biết revision của từng release, cần khi phải rollback.

### Online data migration trên bản cũ

Đây là điều kiện của SLURP. Chạy cho Nova và Cinder, lặp lại cho tới khi cột `Total Needed` của mọi dòng bằng 0:

```bash
kubectl -n openstack exec deploy/nova-api-osapi -c nova-osapi -- nova-manage db online_data_migrations
kubectl -n openstack exec deploy/cinder-api -c cinder-api -- cinder-manage db online_data_migrations
```

Job `nova-db-sync` của chart cũng chạy lệnh này ở mỗi lần cài, nên Nova thường đã sạch. Chart Cinder thì chỉ chạy `db sync`, vì vậy Cinder là cái cần kiểm tra thật sự.

### Kiểm tra override của mình

Tìm các option đã bị bỏ hoặc đổi tên trong override của bạn:

```bash
grep -rnE 'idle_timeout|heartbeat_in_pthread|tenant_network_types|ovn_l3_mode|cross_az_attach' \
  /etc/genestack/helm-configs/
```

Lab không ra kết quả nào. Nếu có, hãy sửa trước khi nâng cấp, theo cách base config của bản mới đã làm.

### Backup

```bash
# MariaDB: dump toàn bộ
kubectl -n openstack exec mariadb-cluster-0 -c mariadb -- sh -c \
  'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --all-databases --single-transaction --routines --events' \
  | gzip > mariadb-$(date +%F-%H%M).sql.gz
# Toàn bộ Secret của namespace openstack (chứa mật khẩu: giữ kín)
kubectl -n openstack get secrets -o yaml > secrets-openstack.yaml && chmod 600 secrets-openstack.yaml
# Cấu hình Genestack
tar czf etc-genestack-$(date +%F).tgz /etc/genestack
```

Mở file dump ra xem để chắc nó có dữ liệu (`zcat … | head`, có các `CREATE DATABASE` của `nova`, `barbican`…). Một bản backup chưa kiểm tra coi như chưa có.

### Tài nguyên để quan sát data plane

Chuẩn bị một VM có floating IP và gắn một volume, cùng load balancer `lb-amp` ở [Phần 10](/posts/genestack-11-octavia/). Mở một terminal riêng, ping liên tục floating IP từ deployer (hoặc từ máy nào vào được mạng external) trong suốt quá trình:

```bash
ping -D <floating-ip> | tee ~/upgrade-2026.3/ping.log
```

API sẽ có những lúc gián đoạn ngắn khi pod cuộn lại. Còn ping tới VM thì **không được mất**. Nếu mất, dừng lại và tìm nguyên nhân trước khi đi tiếp.

## Giai đoạn 1 – Cập nhật mã nguồn Genestack

```bash
cd /opt/genestack
git status --short          # phải trống
git fetch --tags
git checkout release-2026.3.1
git submodule update --init --recursive
git describe --tags         # release-2026.3.1
git submodule status        # không có dấu '+' ở đầu dòng
```

`git status` phải trống. Nếu có file bị sửa, nghĩa là ai đó đã sửa thẳng vào mã nguồn upstream ([Phần 1](/posts/genestack-01-architecture/)). Hãy chuyển thay đổi đó sang `/etc/genestack` trước khi checkout, nếu không nó sẽ bị mất hoặc gây conflict.

Xoá cache chart mà Kustomize có thể đã tải về trong các overlay, theo khuyến nghị của tài liệu upstream:

```bash
find /etc/genestack/kustomize/ -name charts -type d -prune -exec rm -rf {} +
```

Chạy lại `bootstrap.sh`, như ở [Phần 3](/posts/genestack-03-kubernetes-kubespray/): luôn chạy **sau khi** đã checkout đúng release:

```bash
/opt/genestack/bootstrap.sh --no-wait-cloud-init
```

Script chỉ tạo những gì còn thiếu, nên chạy lại trên lab đang chạy là an toàn:

- symlink cho các mục **mới** trong `base-kustomize` (ví dụ `kustomize/octavia/post-renderer.sh`, `kustomize/qonos`) và thư mục `helm-configs/<svc>` cho service mới. Symlink, overlay và override đã có thì giữ nguyên;
- venv `~/.venvs/genestack` được cài lại theo `requirements.txt` của release, và các Ansible collection được tải lại, nên deployer cần ra được Internet;
- `inventory/`, `gateway-api/`, `manifests/` đã có thì bỏ qua. `helm`, `yq`, `cmctl` chỉ được cài khi chưa có, và không bị nâng version.

Giữa hai tag này, `requirements.txt` không đổi, và lab không dùng service mới nào. Vì vậy lần này bỏ qua bootstrap cũng không sao. Nhưng nên giữ bước này thành thói quen cho mọi lần nâng cấp.

Điều quan trọng: **bootstrap không cập nhật `helm-chart-versions.yaml`.** File đã tồn tại thì script bỏ qua, nên giai đoạn 2 vẫn phải làm tay.

> **Từ lúc này tới hết giai đoạn 2, không chạy bất kỳ `install-*.sh` nào.** Script và image đã là bản mới, nhưng chart vẫn là bản cũ.

## Giai đoạn 2 – Cập nhật version chart

So sánh file của bạn với file của release mới:

```bash
diff -u /etc/genestack/helm-chart-versions.yaml /opt/genestack/helm-chart-versions.yaml
```

Lab không tự ghim chart nào, nên diff chỉ gồm các dòng của bảng ở đầu bài (cộng các chart lab không cài như trove, zaqar, ironic…). Chép đè file mới:

```bash
cp /opt/genestack/helm-chart-versions.yaml /etc/genestack/helm-chart-versions.yaml
grep -E '^\s*(keystone|nova|barbican|octavia):' /etc/genestack/helm-chart-versions.yaml
helm repo update
```

Nếu bạn đã ghim version riêng cho chart nào đó, hãy sửa từng dòng theo diff thay vì chép đè cả file.

## Giai đoạn 3 – Giữ nguyên các khoá đang dùng

Script mới tạo secret còn thiếu bằng giá trị ngẫu nhiên. Với mật khẩu của một service mới, đó là điều tốt. Nhưng hai secret dưới đây gắn với **dữ liệu đã có**, nên phải mang đúng giá trị đang chạy.

### Octavia: heartbeat key

Mỗi amphora nhận `heartbeat_key` lúc được tạo, và dùng nó để ký (HMAC) gói heartbeat UDP gửi về health manager. Lấy khoá đang chạy từ cấu hình hiện tại của Octavia rồi tạo secret với đúng giá trị đó:

```bash
KEY=$(kubectl -n openstack get secret octavia-etc -o jsonpath='{.data.octavia\.conf}' \
      | base64 -d | awk -F' *= *' '/^heartbeat_key/ {print $2}')
[ -n "$KEY" ] && echo "đã lấy được khoá (${#KEY} ký tự)"
kubectl -n openstack create secret generic octavia-heartbeat-key \
  --from-literal=heartbeat_key="$KEY"
unset KEY
```

Lab chưa từng đổi khoá này, nên giá trị là `insecure`, mặc định của chart. `install-octavia.sh` sẽ thấy secret đã tồn tại và giữ nguyên. Dùng khoá mặc định là không an toàn, nhưng xử lý nó là việc **sau** nâng cấp (xem cuối bài). Đổi khoá ngay trong lúc nâng cấp sẽ làm health manager bỏ qua heartbeat của mọi amphora đang chạy.

Lab nào chỉ dùng provider `ovn` thì không có amphora, và không cần bước này.

### Barbican: đưa KEK đang chạy vào Secret

Barbican mã hoá khoá của từng project bằng một master key (KEK). Bản 2025.1 tự dùng một KEK mặc định công khai nếu cấu hình không đặt. Bản 2026.1 bỏ giá trị mặc định đó. Genestack mới quản lý KEK trong Secret `barbican-simple-crypto-kek`, kèm script `rotate-barbican-kek.py`.

Chạy script khi `barbican-api` bản **cũ** vẫn đang chạy, vì script cần exec vào pod đó để kiểm tra KEK với database:

```bash
python3 /opt/genestack/scripts/rotate-barbican-kek.py           # chỉ đọc: KEK nào đang dùng, có giải mã được DB không
python3 /opt/genestack/scripts/rotate-barbican-kek.py --adopt   # ghi KEK đang chạy vào Secret, không xoay khoá
kubectl -n openstack get secret barbican-simple-crypto-kek
```

Ở lần chạy đầu, script báo `no kek in the rendered barbican.conf: the upstream default is in effect`. `--adopt` ghi đúng KEK đó vào Secret. Script không bao giờ in khoá, chỉ in fingerprint 8 ký tự.

`install-barbican.sh` cũng có thể tự lấy KEK từ cấu hình đang chạy nếu chưa có Secret. Nhưng `--adopt` kiểm tra trước rằng khoá giải mã được database, nên làm tay bước này an toàn hơn.

## Giai đoạn 4 – Nâng từng service

### Khuôn mẫu chung

Mỗi service đi qua ba bước giống nhau: xoá hook job cũ, chạy script cài đặt, kiểm tra. Định nghĩa hai hàm tiện ích trên deployer:

```bash
# Xoá các Job Helm sẽ tạo lại: hook của service, và Job thường thuộc release
# (không đụng tới Job do CronJob sinh ra)
cleanjobs() {
  kubectl -n openstack get jobs -o json \
    | jq -r --arg p "$1-" --arg r "$1" '.items[]
        | select(.metadata.name | startswith($p))
        | select(all(.metadata.ownerReferences[]?; .kind != "CronJob"))
        | select(.metadata.annotations["helm.sh/hook"] != null
                 or .metadata.annotations["meta.helm.sh/release-name"] == $r)
        | .metadata.name' \
    | xargs -r kubectl -n openstack delete job
}

# Version chart, pod chưa chạy, image đang dùng
chk() {
  helm -n openstack list -f "^$1\$" -o json | jq -r '.[] | "\(.name)  \(.chart)  \(.status)"'
  kubectl -n openstack get pods -l application=$1 --no-headers | grep -v -E 'Running|Completed' || echo "pod: OK"
  kubectl -n openstack get pods -l application=$1 \
    -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort | uniq -c
}
```

**Vì sao xoá job trước khi cài?** Trong chart OpenStack-Helm, các job `db-sync`, `ks-user`, `ks-endpoints`… là Helm hook `post-install,post-upgrade`. Helm chỉ tạo lại chúng **sau khi** đã apply Deployment mới. Trong khoảng thời gian đó, job cũ (đã `Completed`) vẫn còn, nên init container `kubernetes-entrypoint` của pod mới thấy điều kiện "job db-sync đã xong" là thoả mãn. Pod chạy code 2026.1 trên schema database của 2025.1. Xoá job cũ trước thì pod mới đứng ở `Init` cho tới khi `db-sync` mới chạy xong, còn pod cũ tiếp tục phục vụ. Tài liệu Genestack nêu bước này cho Nova. Cùng cơ chế đó áp dụng cho mọi chart.

Hàm `cleanjobs` xoá hai loại job, đều là job Helm chắc chắn sẽ tạo lại, nên xoá chúng không làm mất bước nào:

- **Job có annotation `helm.sh/hook`**: `db-sync`, `ks-*`… như giải thích ở trên.
- **Job thường thuộc release** (annotation `meta.helm.sh/release-name`): ở chart Nova là `nova-bootstrap` và `nova-cell-setup`. Đây là resource bình thường của release, không phải hook. `spec.template` của Job không sửa được sau khi tạo, nên khi chart mới đổi template, Helm báo `cannot patch "nova-bootstrap" with kind Job ... spec.template: Invalid value` và release chuyển sang `failed`. Lúc đó Deployment và DaemonSet đã được apply, còn hook `db-sync` thì không bao giờ được tạo. Toàn bộ pod Nova mới, kể cả `nova-compute` trên compute đang được rolling, đứng mãi ở `Init` với log `jobs.batch "nova-db-sync" not found`.

Job do CronJob sinh ra (`nova-cell-setup-2988…`, `nova-service-cleaner-…`) có `ownerReferences` là CronJob và được bỏ qua.

Script cài đặt chạy `helm upgrade --install` với timeout 120 phút, và Helm **chờ hook job chạy xong** mới trả về. Một script đứng lâu thường là đang chờ `db-sync`. Mở terminal khác để xem:

```bash
kubectl -n openstack get jobs -w
kubectl -n openstack logs -f job/<service>-db-sync
```

### Thứ tự

| # | Service | Lý do đứng ở vị trí này |
|---|---|---|
| 1 | memcached | Hạ tầng dùng chung. Chỉ làm mất cache, không mất dữ liệu |
| 2 | Keystone | Mọi service khác xác thực qua nó |
| 3 | Barbican | Cinder, Octavia, Magnum lấy secret từ đây |
| 4 | Glance, Placement | Nova cần cả hai |
| 5 | libvirt, Nova | Control plane Nova trước, Neutron sau, theo thứ tự upstream khuyến nghị |
| 6 | Neutron | |
| 7 | Cinder (+ playbook trên storage node) | |
| 8 | Octavia, Magnum | Dựa vào Nova, Neutron, Barbican |
| 9 | Masakari, Skyline | Masakari theo dõi Nova. Skyline gọi API của mọi service |

Sau mỗi service, đọc kết quả `chk`. Chỉ đi tiếp khi chart đã là `2026.1.x`, không có pod lỗi, và image không còn `2025.1`.

### 1. memcached

```bash
/opt/genestack/bin/install-memcached.sh
chk memcached
```

Token và cache của các service bị xoá khi memcached khởi động lại. Vài phút đầu, Keystone phải xác thực lại token nên API chậm hơn một chút.

### 2. Keystone

```bash
cleanjobs keystone
/opt/genestack/bin/install-keystone.sh
chk keystone
os token issue -c expires
os endpoint list --interface public -c "Service Name" -c URL
```

Trong log của script sẽ có dòng `Secret not found in k8s, generating and creating: openstack/keystone-test-password`. Đây là secret mới mà `bin/services/keystone.yaml` khai báo (`create-secrets.sh` cũ không tạo nó), không ảnh hưởng gì. Gặp dòng tương tự ở các service khác thì đọc tên secret: chỉ cần lo nếu đó là secret gắn với dữ liệu đã có, như hai secret ở giai đoạn 3.

Keystone 2026.1 trong Genestack thêm policy cho `identity:create_trust` và `identity:list_trusts`. Magnum dùng trust khi tạo cluster. Cluster mới vẫn tạo được, vì người tạo trust chính là người dùng đó.

### 3. Barbican

```bash
cleanjobs barbican
/opt/genestack/bin/install-barbican.sh
chk barbican
kubectl -n openstack get pvc barbican-softhsm-tokens        # Bound
python3 /opt/genestack/scripts/rotate-barbican-kek.py --validate deployed
os secret list -c "Secret href" -c Name -f value | head
```

`--validate deployed` phải thành công: KEK mà bản mới đang dùng giải mã được mọi khoá trong database. Đọc thử một secret cũ (ví dụ chứng chỉ của cluster Magnum ở Phần 11) bằng `os secret get --payload <href>` để chắc chắn.

Từ đây, secret **mới** được mã hoá bằng SoftHSM2 (`p11_crypto`). Secret **cũ** vẫn đọc qua `simple_crypto` với KEK cũ.

### 4. Glance và Placement

```bash
cleanjobs glance ;    /opt/genestack/bin/install-glance.sh ;    chk glance
cleanjobs placement ; /opt/genestack/bin/install-placement.sh ; chk placement
os image list
os resource provider list
```

### 5. libvirt và Nova

Trước khi đụng tới compute, chuyển các host trong segment của Masakari ([Phần 7](/posts/genestack-08-masakari-instance-ha/)) sang chế độ bảo trì. libvirt và `nova-compute` sẽ lần lượt khởi động lại trên từng compute. Monitor của Masakari có thể coi đó là sự cố và hard reboot VM:

```bash
for h in $(os segment host list compute-ha -c name -f value); do
  os segment host update compute-ha "$h" --on_maintenance True
done
os segment host list compute-ha -c name -c on_maintenance
```

**libvirt:**

```bash
/opt/genestack/bin/install-libvirt.sh
kubectl -n openstack rollout status ds/libvirt-libvirt-default
chk libvirt
kubectl -n openstack get ds libvirt-libvirt-default \
  -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'      # libvirt libvirt-exporter
```

DaemonSet cuộn lần lượt từng node (`max_unavailable: 20%`, làm tròn lên thành 1). VM **không** tắt khi pod libvirt khởi động lại: script của chart chạy `libvirtd` bằng `systemd-run --scope` trên host, nên tiến trình QEMU của VM nằm trong cgroup của host, không thuộc pod. Ping trong terminal theo dõi phải không mất gói nào. Xác nhận thêm từ phía libvirt:

```bash
for p in $(kubectl -n openstack get pods -l application=libvirt -o name); do
  kubectl -n openstack exec $p -c libvirt -- virsh list --all
done
```

VM đang chạy vẫn dùng binary QEMU cũ cho tới khi được hard reboot hoặc live migrate. Không cần làm việc đó ngay.

**Nova:**

```bash
cleanjobs nova
/opt/genestack/bin/install-nova.sh
chk nova
kubectl -n openstack exec deploy/nova-api-osapi -c nova-osapi -- nova-status upgrade check
os compute service list
kubectl -n openstack get backendtrafficpolicy nova-volume-attach \
  -o jsonpath='{range .status.ancestors[*]}{range .conditions[*]}{.type}={.status} {end}{"\n"}{end}'
```

`nova-status upgrade check` phải ra `Success` ở mọi mục (`Warning` thì đọc kỹ nội dung). Mọi `nova-compute` phải `up`. Policy `nova-volume-attach` phải `Accepted=True`. Nếu không, xem lại tên route của bạn có đúng là `custom-nova-gateway-route` không.

**Tắt chế độ bảo trì của Masakari:**

Khi libvirt và `nova-compute` trên mọi compute đã chạy ổn định, đưa các host ra khỏi chế độ bảo trì. Đừng để tới cuối đợt nâng cấp: suốt thời gian đó, nếu một compute hỏng thật thì Masakari sẽ không evacuate VM. Các bước sau không khởi động lại gì trên compute nữa. Bước 9 chỉ khởi động lại pod của Masakari, việc đó không làm monitor báo lỗi host.

Trước khi tắt, kiểm tra xem monitor có kịp gửi notification nào trong lúc nâng không. Nếu có, Masakari có thể đã disable `nova-compute` của host đó (Phần 7):

```bash
os notification list --sort-key created_at --sort-dir desc --limit 10
os compute service list --service nova-compute -c Host -c Status -c State
```

Mọi `nova-compute` phải `enabled` và `up`. Host nào bị `disabled` vì notification thì enable lại (`os compute service set --enable <host> nova-compute`) sau khi đã chắc VM trên host đó vẫn chạy bình thường. Sau đó tắt bảo trì và kiểm tra:

```bash
for h in $(os segment host list compute-ha -c name -f value); do
  os segment host update compute-ha "$h" --on_maintenance False
done
os segment host list compute-ha -c name -c on_maintenance      # tất cả False
```

### 6. Neutron

Trước khi nâng, dọn các agent mồ côi. Đó là những dòng `os network agent list` hiện host `("Chassis" register deleted)`, thường còn lại từ node đã gỡ (như controller-2 ở Phần 9). Gốc của chúng là bản ghi `Chassis_Private` trong OVN SB còn tồn tại trong khi `Chassis` tương ứng đã bị xoá. Bản 2025.1 chỉ hiện chúng với `Alive=XXX`. Bản 2026.1 thì đọc `other_config` của chassis để tính availability zone, nên **mọi lệnh `os network agent list` trả về lỗi 500**. Log `neutron-server` có dòng `AttributeError: type object 'DeletedChassis' has no attribute 'other_config'`.

```bash
# Chassis_Private không còn Chassis đi kèm
kubectl ko sbctl --columns=_uuid,name,chassis find Chassis_Private chassis=[]
kubectl ko sbctl destroy Chassis_Private <_uuid>
os network agent list | grep -c deleted        # phải là 0
```

Không dùng `os network agent delete` cho các agent này: lệnh đó đi qua đúng đoạn code bị lỗi ở bản mới.

```bash
cleanjobs neutron
/opt/genestack/bin/install-neutron.sh
chk neutron
os network agent list
```

Trong output của script có dòng `Using tcp connections for the OVN northbound and southbound databases.`, vì lab không bật SSL cho OVN. Data plane do OVN đảm nhiệm, nên ping không bị ảnh hưởng khi `neutron-server` khởi động lại. Metadata (`169.254.169.254`) có thể gián đoạn vài giây khi DaemonSet `neutron-ovn-metadata-agent` cuộn. Chỉ VM đang boot đúng lúc đó mới bị ảnh hưởng.

### 7. Cinder

Phần trong cluster (`cinder-api`, `cinder-scheduler`):

```bash
cleanjobs cinder
/opt/genestack/bin/install-cinder.sh
chk cinder
```

Bản mới chạy `cinder-api` không cần root, với root filesystem chỉ đọc. Nếu bạn có mount hay script tuỳ biến ghi vào container, đây là chỗ có thể lỗi.

Phần trên storage node: `cinder-volume` cài bằng playbook ([Phần 6](/posts/genestack-06-cinder-lvm/)) vẫn đang chạy 2025.1 trong virtualenv `/opt/cinder`. Playbook cài Cinder bằng `pip` với `state: present` vào đúng virtualenv đó. Nâng cấp tại chỗ dễ để lại thư viện của bản cũ. Cách gọn hơn là cất virtualenv cũ đi rồi để playbook dựng virtualenv mới, đồng thời có sẵn đường lui:

```bash
source /opt/genestack/scripts/genestack.rc
INV=/etc/genestack/inventory/inventory.yaml

ansible -i $INV cinder_storage_nodes -b -m shell \
  -a 'systemctl stop cinder-volume && mv /opt/cinder /opt/cinder-2025.1'

ansible-playbook -i $INV /opt/genestack/ansible/playbooks/deploy-cinder-volume.yaml \
  -e cinder_backend_name=lvm-ssd-1 \
  -e cinder_worker_name=lvm \
  -e cinder_release_branch=stable/2026.1
```

Giữa hai lệnh, các thao tác với volume (tạo, xoá, gắn) sẽ chờ. Volume **đang gắn** vào VM không bị ảnh hưởng: I/O đi qua `tgt`, và playbook chỉ reload `tgt` chứ không restart. Handler của playbook có restart `iscsid` trên storage node, nhưng controller không phải là initiator của VM nào.

Bản mới cũng đổi cách dựng cấu hình trên host: `cinder.conf` gốc lấy từ Secret `cinder-etc`, các đoạn cấu hình riêng nằm trong `/etc/cinder/cinder.conf.d/`, và systemd chạy với `--config-dir`. Các file `*.conf.stage` cũ được playbook tự xoá.

Kiểm tra:

```bash
ansible -i $INV cinder_storage_nodes -b -m shell \
  -a '/opt/cinder/bin/pip show cinder | grep ^Version; systemctl is-active cinder-volume tgt'
kubectl -n openstack exec deploy/cinder-api -c cinder-api -- cinder-manage db online_data_migrations
os volume service list
```

Mọi `cinder-volume` (`<node>@lvm-ssd-1`) phải `up`. Tạo một volume mới và gắn vào VM thử để kiểm tra đủ đường đi: LV, target `tgt`, session iSCSI.

### 8. Octavia và Magnum

```bash
cleanjobs octavia
/opt/genestack/bin/install-octavia.sh
chk octavia
```

Đợi vài phút: health manager cần nhận được vài chu kỳ heartbeat. Sau đó kiểm tra:

```bash
os loadbalancer list -c name -c provisioning_status -c operating_status
os loadbalancer amphora list -c id -c loadbalancer_id -c status -f value \
  | diff ~/upgrade-2026.3/amphora-before.txt - && echo "amphora không đổi"
for p in $(kubectl -n openstack get pods -l application=octavia,component=health_manager -o name); do
  kubectl -n openstack logs $p --since=10m | grep -iE 'hmac|failover' | tail -3
done
```

Danh sách amphora phải **giống hệt** trước nâng cấp: không có ID mới, không có `ERROR`. Log health manager không được có lỗi HMAC hay lệnh failover. Nếu label `component` trong bản chart của bạn khác, lấy tên pod bằng `kubectl -n openstack get pods | grep health`.

File `octavia_amphora_provider.yaml` mà lab sinh ra ở Phần 10 vẫn khai báo mount chứng chỉ OVN (`ovn-tls`) cho worker và health manager. Bản mới có post-renderer gỡ các mount này khi kube-ovn không bật SSL, nên không cần sinh lại file.

```bash
cleanjobs magnum
/opt/genestack/bin/install-magnum.sh
chk magnum
os coe cluster template list
os coe cluster list
```

Magnum 2026.1 không còn script `magnum-api-wsgi`. Base config mới đã chuyển sang `module: magnum.wsgi.api:application` và dựng lại pipeline PasteDeploy. Nếu `magnum-api` crash, hãy kiểm tra override của bạn có ghi đè `conf.magnum_api_uwsgi` hay `conf.paste` không. Cluster đang chạy (Phần 11) không bị ảnh hưởng: chúng là tài nguyên trên management cluster và trong project của người dùng.

### 9. Masakari và Skyline

```bash
cleanjobs masakari
/opt/genestack/bin/install-masakari.sh
chk masakari
os segment list
```

Chờ `masakari-api`, `masakari-engine` và ba DaemonSet monitor đều `Running`. Segment và host phải còn nguyên, với `on_maintenance = False` như lúc tắt ở bước 5:

```bash
os segment host list compute-ha -c name -c on_maintenance
os notification list --sort-key created_at --sort-dir desc --limit 5   # không có notification mới
```

```bash
cleanjobs skyline
/opt/genestack/bin/install-skyline.sh
chk skyline
```

Chart Skyline không đổi version, nhưng image của các job đổi sang `openstack-client:2026.1-latest`. Đăng nhập Skyline, mở danh sách instance, volume, load balancer để chắc mọi API đều trả lời.

Lab nào có cài thêm Heat thì nâng Heat theo cùng khuôn mẫu, ở bất kỳ lúc nào sau Keystone.

## Giai đoạn 5 – Kiểm tra sau nâng cấp

Tìm pod còn chạy image 2025.1. Kết quả phải rỗng:

```bash
kubectl -n openstack get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}' \
  | grep 2025.1
```

So version chart với lúc trước:

```bash
helm -n openstack list -o json | jq -r '.[] | "\(.name)\t\(.chart)"' | sort
```

Sau đó kiểm tra theo chức năng:

| Kiểm tra | Lệnh | Mong đợi |
|---|---|---|
| Ping VM thử | Terminal theo dõi | Không mất gói trong suốt quá trình |
| VM cũ | `os server list --all-projects` | Giống `servers-before.txt`, đều `ACTIVE` |
| Boot VM mới từ volume | `os server create --boot-from-volume …` | `ACTIVE`, ping được, SSH được |
| Gắn volume | `os server add volume …` | Thấy disk trong VM |
| Load balancer | `curl` tới VIP của `lb-amp` | Trả lời, amphora không đổi |
| Barbican | `os secret store` rồi `os secret get --payload` | Đọc lại đúng nội dung |
| Masakari | `os segment host list compute-ha` | `on_maintenance = False` |
| Grafana | Dashboard *OpenStack Libvirt Advanced* (Phần 12) | Vẫn có dữ liệu từ `libvirt-exporter` |

Giữ `/opt/cinder-2025.1` trên storage node và các file trong `~/upgrade-2026.3` thêm vài ngày. Xoá sau khi chắc chắn không cần quay lại.

## Nếu phải quay lại

`helm rollback <release> <revision>` đưa manifest về bản cũ, nhưng **không** đưa database về. Job `db-sync` của bản mới đã đổi schema, và code 2025.1 không chạy được trên schema 2026.1. Vì vậy:

- **Lỗi xảy ra trong lúc cài một service** (job `db-sync` fail, pod mới không lên): pod cũ thường vẫn đang phục vụ, vì pod mới đứng ở `Init`. Đọc log của job, sửa nguyên nhân rồi chạy lại script cài. Đó là đường đi tiến, thường nhanh hơn quay lui.
- **Phải quay lại thật sự**: restore database của service đó từ file dump, trả `/etc/genestack/helm-chart-versions.yaml` và `/opt/genestack` về bản cũ (`git checkout release-2026.2.0.2`), rồi `helm rollback` hoặc chạy lại script cài cũ. Mọi thay đổi trên cloud sau thời điểm dump đều mất.
- **Barbican** là trường hợp đặc biệt: secret tạo sau khi nâng cấp được mã hoá bằng SoftHSM2, bản 2025.1 không đọc được chúng.

Vì vậy nên dừng ở từng bước kiểm tra. Phát hiện vấn đề ngay sau service vừa nâng thì xử lý dễ hơn nhiều so với lúc đã đi hết danh sách.

## Gặp lỗi thì xem ở đâu

| Triệu chứng | Nguyên nhân hay gặp | Xử lý |
|---|---|---|
| `chk` cho thấy chart vẫn `2025.1.x` nhưng image là `2026.1` | Quên giai đoạn 2 | Cập nhật `helm-chart-versions.yaml`, chạy lại script |
| Script dừng với `Failed to prepare secret-backed Helm arguments` | Secret đã có nhưng thiếu key mà bản mới cần | Đọc dòng `[ERROR]` phía trên để biết secret và key, so với `bin/services/<svc>.yaml` |
| Pod mới đứng ở `Init:0/1` rất lâu | Job hook đang chạy hoặc fail | `kubectl -n openstack get jobs`, log của `<svc>-db-sync` |
| Pod mới đứng ở `Init`, log `jobs.batch "<svc>-db-sync" not found`; `helm history` báo `cannot patch "<job>" with kind Job` | Job thường của release (ví dụ `nova-bootstrap`) không patch được, Helm dừng trước khi tạo hook | `kubectl -n openstack delete job <job>` rồi chạy lại script cài đặt |
| `os network agent list` lỗi 500, log `'DeletedChassis' has no attribute 'other_config'` | Còn `Chassis_Private` mồ côi trong OVN SB | `kubectl ko sbctl find Chassis_Private chassis=[]`, `destroy` các bản ghi đó (xem mục Neutron) |
| `barbican-api` CrashLoop, log `SimpleCrypto KEK is undefined` | Không có KEK nào được render | `helm rollback barbican`, chạy `rotate-barbican-kek.py --adopt`, cài lại |
| Amphora chuyển `ERROR` và bị thay hàng loạt sau khi nâng Octavia | Heartbeat key mới khác khoá trong amphora | So Secret `octavia-heartbeat-key` với khoá cũ, sửa Secret và chạy lại `install-octavia.sh` |
| `os volume service list` báo `cinder-volume` down | Virtualenv mới lỗi hoặc cấu hình trên host thiếu | `journalctl -u cinder-volume` trên storage node. Đường lui: dừng service, đổi tên `/opt/cinder-2025.1` về `/opt/cinder`, start lại |
| VM bị hard reboot trong lúc nâng compute | Host chưa ở chế độ bảo trì của Masakari | `os notification list`, bật `on_maintenance` trước khi làm tiếp |

## Việc nên làm sau nâng cấp

Các việc dưới đây không bắt buộc cho lần nâng cấp này. Mỗi việc nên làm riêng, khi cloud đã ổn định:

- **Xoay KEK của Barbican** khỏi khoá mặc định công khai: `rotate-barbican-kek.py` (xem kế hoạch), `--stage`, backup database `barbican`, chạy `install-barbican.sh`, rồi `--validate deployed`. Đây là thao tác một chiều. Chứng chỉ của Octavia và Magnum phụ thuộc vào nó.
- **Đổi heartbeat key của Octavia**: amphora đang chạy phải nhận khoá mới, nếu không sẽ bị failover. Cần cửa sổ bảo trì riêng.
- **Monitoring**: chart `opentelemetry-kube-stack` giữ nguyên version, nhưng base config đổi: receiver `httpcheck` mặc định trỏ vào endpoint trong cluster, nên vấn đề domain mẫu ở Phần 12 không còn. Chạy lại `install-opentelemetry-kube-stack.sh`, rồi import thêm các dashboard mới (cert-manager, MetalLB, CoreDNS, Kubernetes control plane, OpenStack API URLs) bằng `scripts/import-grafana-dashboard.py`, nay đã có sẵn trong release.
- **Gia cố bảo mật**: bản mới tắt việc tự mount service account token cho RabbitMQ, Grafana, longhorn-ui, backup của MariaDB… Cluster đang chạy chỉ nhận thay đổi này khi apply lại các kustomize base tương ứng và chạy lại script của Grafana, Longhorn (xem *Upgrade Notes* trong `docs/release-2026.3.0.md`).

## Tóm tắt

- `release-2026.3.1` nâng OpenStack từ 2025.1 lên 2026.1. Cả hai là bản SLURP nên nâng thẳng được, với điều kiện online data migration của bản cũ đã xong.
- Lần này chỉ lớp OpenStack đổi chart. Hạ tầng và Kubernetes giữ nguyên.
- Version do ba nơi quyết định: `/opt/genestack` (đổi theo git), `/etc/genestack/helm-chart-versions.yaml` (**không** tự đổi), và Secret. Phải tự cập nhật file version chart trước khi chạy bất kỳ script nào.
- Script mới tạo secret còn thiếu bằng giá trị ngẫu nhiên. Với Octavia (heartbeat key) và Barbican (KEK), phải đưa giá trị đang dùng vào Secret trước, nếu không sẽ failover toàn bộ amphora hoặc Barbican không khởi động.
- Nâng từng service theo khuôn mẫu: xoá hook job, chạy `install-<svc>.sh`, kiểm tra. Đặt Masakari ở chế độ bảo trì khi nâng libvirt và Nova, và tắt ngay khi Nova đã ổn định. Chạy lại playbook `cinder-volume` với `stable/2026.1` trên một virtualenv mới.
- Theo dõi data plane suốt quá trình: VM, volume đang gắn và amphora phải không bị ảnh hưởng.
- Rollback không chỉ là `helm rollback`: schema database chỉ đi một chiều. Hãy backup, và dừng lại kiểm tra sau mỗi service.
