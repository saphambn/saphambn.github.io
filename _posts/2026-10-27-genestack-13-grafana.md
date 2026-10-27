---
title: "OpenStack trên Kubernetes với Genestack – Phần 12: Giám sát hệ thống với Grafana"
date: 2026-10-27 10:15:00 +0700
categories: [OpenStack, Genestack]
tags: [openstack, kubernetes, genestack, grafana, prometheus, opentelemetry, monitoring]
render_with_liquid: false
---

[Phần 11](/posts/genestack-12-magnum/) đã đưa vào cloud thêm một tầng phức tạp: Kubernetes trong Kubernetes. Tới đây, cụm có hơn chục service OpenStack, một Galera, một RabbitMQ, kube-ovn, Longhorn, Envoy Gateway, cộng thêm các VM của người dùng. Khi có sự cố, `kubectl get pods` chỉ cho biết pod còn sống hay không. Nó không cho biết Galera đang chậm, RabbitMQ đang dồn message, hay một compute đã gần hết RAM.

Bài này cài **Grafana** theo cách của Genestack, và import bộ dashboard có sẵn trong repo để có cái nhìn tổng thể về hệ thống. Trước khi cài, cần hiểu metric đi vào Prometheus bằng đường nào. Ở bản 2026.2, phần lớn metric **không** do kube-prometheus-stack thu thập. Nếu bỏ qua điểm này, bạn sẽ import xong và thấy một nửa số dashboard trống trơn.

## Metric đi vào Prometheus bằng đường nào

![Luồng metric trong Genestack: kube-prometheus-stack scrape kubelet qua ServiceMonitor, OpenTelemetry collector thu thập phần còn lại và remote write vào Prometheus, Grafana đọc từ Prometheus và lưu cấu hình trong MariaDB riêng](/assets/img/genestack/13-grafana-monitoring.svg)

Stack monitoring của Genestack gồm năm thành phần, cài theo thứ tự sau:

| # | Thành phần | Script | Vai trò |
|---|---|---|---|
| 1 | kube-prometheus-stack | `install-kube-prometheus-stack.sh` | Prometheus, Alertmanager, Prometheus Operator |
| 2 | Loki | `install-loki.sh` | Lưu log |
| 3 | Tempo | `install-tempo.sh` | Lưu trace |
| 4 | Grafana | `install-grafana.sh` | Dashboard, datasource |
| 5 | opentelemetry-kube-stack | `install-opentelemetry-kube-stack.sh` | Thu thập metric, log, trace rồi đẩy về 1–3 |

Lab đã có (1) từ [Phần 4a](/posts/genestack-04a-infrastructure-network/): `setup-infrastructure.sh` cài nó trước MetalLB vì nhiều chart cần CRD `ServiceMonitor`. Bài này cài (4) và (5). Loki và Tempo để dành cho một bài riêng.

Mở file override gốc của kube-prometheus-stack sẽ thấy điều đáng chú ý:

```bash
grep -E -A1 '^(grafana|nodeExporter|kubeStateMetrics|kubeProxy|coreDns):' \
  /opt/genestack/base-helm-configs/kube-prometheus-stack/kube-prometheus-stack-helm-overrides.yaml
grep enableRemoteWriteReceiver \
  /opt/genestack/base-helm-configs/kube-prometheus-stack/kube-prometheus-stack-helm-overrides.yaml
```

Grafana, node-exporter, kube-state-metrics, kube-proxy, CoreDNS… đều `enabled: false`. Thứ duy nhất kube-prometheus-stack tự scrape là kubelet/cAdvisor. Prometheus được bật `enableRemoteWriteReceiver: true` để nhận metric từ **OpenTelemetry collector**:

- **Collector daemon** (DaemonSet, mỗi node một pod): scrape node-exporter, etcd, libvirt exporter trên compute, các pod có annotation `prometheus.io/scrape`. Nó còn đọc log container và log OpenStack để gửi sang Loki.
- **Collector deployment** (một pod): dùng receiver `mysql` gọi vào Galera của OpenStack, `rabbitmq` gọi management API, `memcached`, và một receiver `prometheus/infra` scrape kube-state-metrics, kube-ovn, Envoy Gateway, MetalLB, cert-manager, kube-proxy, CoreDNS.

node-exporter và kube-state-metrics cũng được cài như chart con của `opentelemetry-kube-stack`. Nói cách khác: **không có bước (5) thì gần như không có dashboard nào có dữ liệu**, kể cả dashboard node.

## Điều kiện cần

| Điều kiện | Trong lab | Ghi chú |
|---|---|---|
| kube-prometheus-stack | Phần 4a | Pod `prometheus-kube-prometheus-stack-prometheus-0` chạy trong namespace `monitoring` |
| mariadb-operator | Phần 4b | Grafana dùng **một MariaDB riêng** trong namespace `monitoring`, không dùng chung Galera của OpenStack |
| StorageClass `general` | Phần 4b | PVC của Grafana và của MariaDB nói trên |
| Listener `grafana-https` và HTTPRoute | Phần 4a | `setup-envoy-gateway.sh` đã tạo sẵn cho mọi template trong `etc/gateway-api/` |
| DNS và CA cho `*.api.openstack.example.com` | Phần 11 | dnsmasq trên deployer, CA ở `/etc/genestack/public-endpoint-ca.crt` |
| Secret `mariadb-monitoring`, `rabbitmq-monitoring-user` | Chưa có | Script ở bước 4 tự tạo |

Kiểm tra nhanh:

```bash
kubectl -n monitoring get pods
kubectl -n monitoring get httproute grafana-gateway-route -o jsonpath='{.spec.hostnames[0]}{"\n"}'
kubectl -n envoy-gateway get gateway flex-gateway \
  -o jsonpath='{range .spec.listeners[?(@.name=="grafana-https")]}{.name} {.hostname}{"\n"}{end}'
grep -E '^\s*(grafana|opentelemetry-kube-stack|kube-prometheus-stack):' /etc/genestack/helm-chart-versions.yaml
```

Hostname của route và listener phải là `grafana.api.openstack.example.com`. Với Genestack 2026.2.0.2, chart Grafana là `10.1.0` (image Grafana `12.2.0`), `opentelemetry-kube-stack` là `0.13.1`.

## Bước 1 – Đặt URL cho Grafana

Grafana dùng biến `custom_host` để điền `server.domain` và `server.root_url` trong `grafana.ini`. Giá trị gốc là `grafana.example.com`, phải đổi cho khớp với route:

```bash
mkdir -p /etc/genestack/helm-configs/grafana
cat > /etc/genestack/helm-configs/grafana/grafana-helm-overrides.yaml <<'EOF'
custom_host: grafana.api.openstack.example.com
EOF
```

Nếu `root_url` sai, trang đăng nhập vẫn mở được, nhưng các link Grafana tự sinh (redirect sau login, link chia sẻ, link trong cảnh báo) sẽ trỏ sai tên.

Mọi giá trị khác giữ như file gốc trong `/opt/genestack/base-helm-configs/grafana/`. Lab dùng được ngay.

## Bước 2 – Cài Grafana

```bash
/opt/genestack/bin/install-grafana.sh
```

Script làm các việc sau:

1. tạo namespace `monitoring` nếu chưa có;
2. đảm bảo Secret `grafana-db` (gồm `username`, `password`, `root-password`) tồn tại. Nếu chưa có, script lấy từ `/etc/genestack/kubesecrets.yaml` do `create-secrets.sh` sinh ra. Nếu file đó cũng không có, script tự sinh mật khẩu ngẫu nhiên;
3. đọc version chart từ `/etc/genestack/helm-chart-versions.yaml`, rồi `helm upgrade --install grafana grafana/grafana` với các file values theo thứ tự `base-helm-configs/grafana/` → `helm-configs/global_overrides/` → `helm-configs/grafana/`;
4. chạy post-renderer `kustomize/grafana/overlay` để thêm vào release một CR `MariaDB` tên `mariadb-cluster` (1 replica, PVC 10 GiB trên `general`), cùng `Database`, `User`, `Grant` cho user `grafana` và một `Backup` hằng ngày. Grafana lưu user, dashboard, service account trong database này.

Pod Grafana có thể restart vài lần trong lần cài đầu tiên: nó khởi động trước khi MariaDB và user `grafana` sẵn sàng. Đó là hành vi bình thường, chờ vài phút.

Lúc khởi động, Grafana tải hai plugin `camptocamp-prometheus-alertmanager-datasource` và `yesoreyeram-infinity-datasource` từ `grafana.net`, nên pod phải ra được Internet. Lab ra được, nhưng ở môi trường đóng thì đây là chỗ đầu tiên cần xem nếu pod không lên.

### Kiểm tra

```bash
kubectl -n monitoring get mariadb,database,user,grant
kubectl -n monitoring get pods -l app.kubernetes.io/instance=grafana
kubectl -n monitoring get pvc
kubectl -n monitoring logs deploy/grafana -c grafana | grep -iE 'error|database' | tail
```

| Kiểm tra | Kết quả mong đợi |
|---|---|
| `mariadb`, `database`, `user`, `grant` | Tất cả `READY=True` |
| Pod `grafana-*` | `Running`, container `grafana` ready |
| PVC | `grafana` (RWX) và `storage-mariadb-cluster-0` đều `Bound` |
| Log | Không còn lỗi kết nối database |

PVC `grafana` là RWX, nên Longhorn dựng thêm một pod `share-manager-pvc-…` trong `longhorn-system` để làm NFS server ([Phần 4b](/posts/genestack-04b-infrastructure-state/)). Nếu pod Grafana kẹt ở `ContainerCreating`, hãy xem share-manager trước.

## Bước 3 – Truy cập Grafana

Mật khẩu `admin` do chart sinh ra và lưu trong Secret `grafana`:

```bash
kubectl -n monitoring get secret grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Kiểm tra qua gateway từ deployer (deployer đã có `/etc/hosts` cho các hostname của route từ Phần 10, và CA từ Phần 11):

```bash
curl --cacert /etc/genestack/public-endpoint-ca.crt \
  https://grafana.api.openstack.example.com/api/health
```

Kết quả phải là một JSON có `"database": "ok"`.

Mở `https://grafana.api.openstack.example.com` trên trình duyệt. Máy client cần phân giải được hostname này về VIP `10.0.0.200` (Phần 4a) và nên import CA của lab (Phần 11) để không bị cảnh báo chứng chỉ.

Đăng nhập bằng `admin`, vào *Connections → Data sources*, bấm *Test* trên `Prometheus`. Lúc này Prometheus chỉ có metric của kubelet/cAdvisor và của chính nó. Phần còn lại đến ở bước 4.

File gốc còn khai báo sẵn hai datasource `Loki` và `Tempo`. Lab chưa cài hai thành phần này, nên bấm *Test* sẽ báo lỗi. Đó là điều bình thường, chúng sẽ tự hoạt động khi Loki và Tempo được cài.

## Bước 4 – Cài OpenTelemetry collector

```bash
/opt/genestack/bin/install-opentelemetry-kube-stack.sh
```

Trước khi gọi Helm, script chuẩn bị credential để collector đọc được metric của các thành phần có state:

- tạo Secret `mariadb-monitoring` trong `openstack`, apply `User` và `Grant` cho user `monitoring` trên Galera `mariadb-cluster` của OpenStack, rồi copy Secret sang `monitoring`;
- tương tự với Secret `rabbitmq-monitoring-user`, và `User`, `Permission` trên RabbitMQ.

Collector deployment kết nối tới `mariadb-cluster-internal.openstack.svc.cluster.local:3306` và `http://rabbitmq.openstack.svc.cluster.local:15672`. Đây đúng là tên Service trong lab, nên không phải sửa gì.

**Receiver `httpcheck`.** Collector deployment có sẵn một danh sách endpoint `https://nova.api.example.com`, `https://keystone.api.example.com`… để đo thời gian phản hồi của API. Đó là giá trị mẫu, không khớp domain của lab. Không dashboard nào trong bộ import ở bước 5 dùng metric này, nên có thể bỏ qua. Nếu muốn dùng, hãy override `collectors.deployment.config.receivers.httpcheck.targets` với các hostname `*.api.openstack.example.com`. Override này thay **toàn bộ** danh sách. Lưu ý collector cũng phải tin CA của lab.

### Kiểm tra

```bash
kubectl -n monitoring get opentelemetrycollectors
kubectl -n monitoring get pods -l app.kubernetes.io/instance=opentelemetry-kube-stack -o wide
kubectl -n monitoring get pods | grep -E 'node-exporter|kube-state-metrics'
kubectl -n openstack get user.k8s.mariadb.com monitoring
kubectl -n monitoring logs deploy/opentelemetry-kube-stack-deployment-collector --tail=50 \
  | grep -iE 'error|denied|refused'
```

| Kiểm tra | Kết quả mong đợi |
|---|---|
| `opentelemetrycollectors` | Hai collector: `daemon` (mode `daemonset`) và `deployment` |
| Pod collector daemon | Một pod trên **mỗi** node, kể cả compute |
| node-exporter, kube-state-metrics | Có pod, node-exporter chạy trên mọi node |
| User `monitoring` trên Galera | `READY=True` |
| Log collector deployment | Không có `Access denied` (MariaDB) hay `401` (RabbitMQ) lặp lại |

Tên Deployment của collector có thể khác tuỳ version chart. Nếu lệnh `logs` báo không tìm thấy, lấy tên đúng bằng `kubectl -n monitoring get deploy`.

Sau một hai phút, kiểm tra trên Prometheus rằng metric đã về:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 >/dev/null &
for q in 'count(node_uname_info)' 'count(kube_node_info)' 'count(mysql_uptime)' \
         'count(rabbitmq_consumer_count)' 'count(libvirt_domain_info_state)' \
         'count by (job) (up)'; do
  echo "== $q"
  curl -s --get http://127.0.0.1:9090/api/v1/query --data-urlencode "query=$q" \
    | jq -c '.data.result[] | [.metric.job // "", .value[1]]'
done
kill %1
```

`count(node_uname_info)` phải bằng số node (6 trong lab), `count(kube_node_info)` cũng vậy. `libvirt_domain_info_state` chỉ có khi đang có VM chạy trên compute. Truy vấn cuối liệt kê các job đang được scrape. Đó là cách nhanh nhất để biết dashboard nào sẽ có dữ liệu.

## Bước 5 – Import dashboard

Genestack để sẵn 27 file JSON trong `/opt/genestack/etc/grafana-dashboards/`, kèm script `/opt/genestack/scripts/import-grafana-dashboard.py`. Script đọc từng file và gọi API `POST /api/dashboards/import`, gán datasource Prometheus cho input `DS_PROMETHEUS`, và bật `overwrite: true`, nên chạy lại bao nhiêu lần cũng được.

### Script trong bản 2026.2.0.2 có hai hạn chế

1. **Chỉ import được *cả thư mục*,** không chọn được từng file.
2. **Không đọc được định dạng dashboard mới.** Ba file `memcached_metrics.json`, `open_alerts.json` và `project_lookup.json` được export từ Grafana 12 theo schema `dashboard.grafana.app/v2beta1` (khoá gốc là `apiVersion`, `kind`, `spec`, không có `title`). API import chỉ nhận định dạng cũ, nên ba file này sẽ bị Grafana trả lỗi.

Bản script trên nhánh `main` của Genestack đã sửa cả hai: thêm hàm chuyển `v2beta1` sang định dạng cũ, và thêm tuỳ chọn `-f` để chọn từng file. Bản này vẫn nhận tài khoản admin như bản cũ. Script chỉ cần `requests`, không phụ thuộc phần nào khác của repo. Có thể tải riêng script này về dùng mà không phải nâng Genestack. Ghim theo commit để lần sau tải vẫn được đúng bản đã kiểm tra:

```bash
mkdir -p /etc/genestack/scripts
curl -fsSL -o /etc/genestack/scripts/import-grafana-dashboard.py \
  https://raw.githubusercontent.com/rackerlabs/genestack/b445a65cf0b3852b5d95cba98563fbef4810ff0e/scripts/import-grafana-dashboard.py
less /etc/genestack/scripts/import-grafana-dashboard.py   # đọc trước khi chạy
```

Để file trong `/etc/genestack` để nó đi cùng phần cấu hình của bạn, và không lẫn vào mã nguồn upstream trong `/opt/genestack`.

### Thông tin đăng nhập

Script dùng tài khoản `admin`, mật khẩu lấy từ Secret `grafana` như ở bước 3:

```bash
export GRAFANA_URL=https://grafana.api.openstack.example.com
export GRAFANA_USERNAME=admin
export GRAFANA_PASSWORD=$(kubectl -n monitoring get secret grafana -o jsonpath='{.data.admin-password}' | base64 -d)
export REQUESTS_CA_BUNDLE=/etc/genestack/public-endpoint-ca.crt   # cho python requests
```

Khi không có `GRAFANA_TOKEN`, script dùng basic auth với `GRAFANA_USERNAME`/`GRAFANA_PASSWORD` và in ra một dòng cảnh báo rằng cách này đã cũ. Với lab, cảnh báo này có thể bỏ qua.

`REQUESTS_CA_BUNDLE` là biến môi trường mà thư viện `requests` đọc để lấy CA. Thiếu nó, script sẽ lỗi `CERTIFICATE_VERIFY_FAILED` vì chứng chỉ của gateway do CA nội bộ ký.

### Chọn dashboard

Không phải dashboard nào cũng hợp với lab. Bảng dưới phân loại theo nguồn metric:

| File | Dashboard | Nguồn metric | Trong lab |
|---|---|---|---|
| `kubernetes_cluster_monitoring.json` | Kubernetes cluster monitoring | kubelet/cAdvisor (kube-prometheus-stack) | Có |
| `envoy-resource-monitor.json` | Envoy Resources Monitor | cAdvisor | Có |
| `prometheus_overview.json` | Prometheus 2.0 Overview | Chính Prometheus | Có |
| `alertmanager.json` | Alertmanager | Metric của Alertmanager | Có |
| `open_alerts.json` | Open Alerts | Datasource kiểu Alertmanager | Chưa: file gốc không khai báo datasource này |
| `node_metrics.json` | Node Exporter Full | node-exporter | Có, sau bước 4 |
| `kube_state_metrics.json` | kube-state-metrics-v2 | kube-state-metrics | Có, sau bước 4 |
| `kube_proxy_metrics.json` | Kubernetes / Proxy | OTel, job `kube-proxy` | Có, sau bước 4 |
| `kube_ovn_controller.json`, `kube_ovn_cni.json`, `kube_ovn_ovn.json`, `kube_ovn_ovs.json`, `kube_ovn_pinger.json` | Kube-OVN / … | OTel, các job `kube-ovn-*` | Có, sau bước 4 |
| `envoy-gateway.json` | Envoy Gateway Global | OTel, job `envoy-gateway` | Có, sau bước 4 |
| `mariadb_galera.json` | MariaDB / Galera – Cluster Overview | OTel receiver `mysql` | Có, sau bước 4 |
| `rabbitmq_metrics.json` | RabbitMQ Cluster Overview | OTel receiver `rabbitmq` | Có, sau bước 4 |
| `memcached_metrics.json` | Memcached Details | OTel receiver `memcached` | Có, sau bước 4 (file `v2beta1`) |
| `libvirt_exporter_inovex_dashboard.json` | OpenStack Libvirt Advanced | libvirt exporter, qua OTel daemon | Có, khi có VM chạy |
| `kube_ovn_backup.json` | Kube-OVN / OVN-Backup | job `ovn-backup` | Chỉ khi có chạy backup OVN |
| `openstack_metrics.json` | OpenStack Exporter | `openstack-metrics-exporter` | Chưa cài |
| `custom_node_metrics.json` | Custom Node Metrics | Metric tự định nghĩa trên node | Chưa cấu hình |
| `kube-event-exporter.json`, `project_lookup.json`, `loki_disks_usage.json` | | Loki | Chưa cài Loki |
| `ceph_cluster.json`, `ceph_pools.json` | Ceph | Ceph exporter | Lab không có Ceph |
| `postgres_metrics.json` | PostgreSQL | OTel receiver `postgresql` (tắt mặc định) | Lab không có PostgreSQL |

Dashboard *MariaDB / Galera* dựng hoàn toàn từ metric `mysql_*` của receiver `mysql`: QPS, connection, buffer pool, lock, handler. Nó **không** có trạng thái cluster Galera (`wsrep_*`). Để biết Galera còn đủ 3 node và đang `Synced`, vẫn phải dùng các lệnh ở Phần 4b.

### Chạy import

```bash
source /opt/genestack/scripts/genestack.rc      # kích hoạt venv ~/.venvs/genestack
python3 -c 'import requests' || pip install requests
D=/opt/genestack/etc/grafana-dashboards

python3 /etc/genestack/scripts/import-grafana-dashboard.py --dir "$D" --datasource Prometheus \
  -f kubernetes_cluster_monitoring.json -f envoy-resource-monitor.json \
  -f prometheus_overview.json -f alertmanager.json \
  -f node_metrics.json -f kube_state_metrics.json -f kube_proxy_metrics.json \
  -f kube_ovn_controller.json -f kube_ovn_cni.json -f kube_ovn_ovn.json \
  -f kube_ovn_ovs.json -f kube_ovn_pinger.json -f envoy-gateway.json \
  -f mariadb_galera.json -f rabbitmq_metrics.json -f memcached_metrics.json \
  -f libvirt_exporter_inovex_dashboard.json
```

Mỗi file sẽ in ra một dòng `Imported <file>: 200`. Dòng nào là `Failed to import` thì có kèm mã lỗi và thông báo của Grafana. Mọi file đều không có `folderTitle`, nên tất cả dashboard nằm trong folder *General*.

Khi cài thêm Loki hay openstack-metrics-exporter, chỉ cần chạy lại lệnh trên với các file tương ứng. Dashboard đã có sẽ bị ghi đè bằng bản trong file. Vì vậy **đừng sửa trực tiếp dashboard gốc trên giao diện**: hãy *Save as* thành bản mới, hoặc export ra JSON và lưu trong `/etc/genestack`.

Xong việc thì xoá mật khẩu khỏi biến môi trường:

```bash
unset GRAFANA_PASSWORD
```

## Tóm tắt

- Stack monitoring của Genestack có năm phần: kube-prometheus-stack, Loki, Tempo, Grafana, opentelemetry-kube-stack. Bài này cài Grafana và OpenTelemetry. Loki và Tempo để sau.
- Ở bản 2026.2, kube-prometheus-stack chỉ cung cấp Prometheus, Alertmanager và scrape kubelet. node-exporter, kube-state-metrics và hầu hết metric khác do **OpenTelemetry collector** thu thập rồi remote write vào Prometheus. Không cài OpenTelemetry thì phần lớn dashboard sẽ trống.
- Grafana được cài bằng `install-grafana.sh`, trong namespace `monitoring`, lưu dữ liệu trong **một MariaDB riêng** do post-renderer thêm vào release.
- Chỉ cần override `custom_host` cho khớp với HTTPRoute có sẵn từ Phần 4a.
- Script import của bản 2026.2.0.2 không đọc được dashboard `v2beta1` và không chọn được từng file. Bản trên `main` (ghim theo commit) sửa cả hai và chạy được độc lập. Đăng nhập bằng tài khoản `admin` lấy từ Secret `grafana`.
- Chỉ import những dashboard có nguồn metric trong cluster. Không sửa dashboard gốc trên giao diện, vì lần import sau sẽ ghi đè.
