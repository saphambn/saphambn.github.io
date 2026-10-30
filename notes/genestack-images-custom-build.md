---
layout: page
title: "Notes: build image Magnum từ fork genestack-images"
permalink: /notes/genestack-images-custom-build/
sitemap: false
render_with_liquid: false
---

Ghi chú cách build lại image Magnum từ repo [`genestack-images`](https://github.com/rackerlabs/genestack-images), với driver `magnum-capi-helm` lấy từ một repo khác. Dùng khi cần image build đầy đủ như của rackerlabs (constraints, script vá, quét Trivy), thay vì chỉ thêm một layer lên image có sẵn.

## Repo genestack-images build image Magnum thế nào

| Đường dẫn | Nội dung |
|---|---|
| `ContainerFiles/<service>` | Containerfile của từng service. Magnum là `ContainerFiles/magnum` |
| `ContainerFiles/openstack-venv` | Image nền: Python 3.12 trên Debian bookworm, có sẵn virtualenv `/var/lib/openstack` |
| `scripts/<service>-*.sh` | Script vá CVE, vá lỗi, chạy trong lúc build |
| `.github/workflows/container-build-<service>.yaml` | Workflow GitHub Actions: build, quét Trivy, push lên GHCR |

`ContainerFiles/magnum` là một build hai tầng:

1. **Tầng build** dựa trên `openstack-venv`, cài Magnum và driver bằng một lệnh `pip install`, có `upper-constraints` của nhánh OpenStack:

   ```dockerfile
   RUN /var/lib/openstack/bin/pip install --constraint https://opendev.org/openstack/requirements/raw/branch/${OS_CONSTRAINTS}/upper-constraints.txt \
       git+https://opendev.org/openstack/magnum.git@${OS_VERSION}#egg=magnum \
       git+https://opendev.org/openstack/magnum-capi-helm@${OS_CAPI_PLUGIN_VERSION}#egg=magnum_capi_helm \
       PyMySQL python-memcached uwsgi
   ```

   Sau đó chạy `magnum-cve-patching.sh`, `magnum-db-migration-patching.sh` (bỏ charset `utf8` trong migration, chỉ cho `stable/2025.1` và `stable/2026.1`) và cài `helm` 3.
2. **Tầng chạy** là `python:3.12-slim-bookworm`, chỉ chép virtualenv và `helm` sang, tạo user `magnum` (UID 42424). Không có `git`.

Workflow `container-build-magnum.yaml` build ma trận nhánh OpenStack (`stable/2025.1`, `stable/2026.1`) × version driver (`1.4.0`, `1.2.1`, `master`), push tag `2025.1-latest` và `2025.1-<timestamp>`.

- URL driver cố định là opendev. `OS_CAPI_PLUGIN_VERSION` chỉ đổi được tag, nhánh, commit.
- Version driver không nằm trong tag: ba bản driver cùng push vào `2025.1-latest`, bản nào xong sau thì thắng.

## Sửa Containerfile

```diff
 ARG OS_CAPI_PLUGIN_VERSION=master
+ARG OS_CAPI_PLUGIN_REPO=https://opendev.org/openstack/magnum-capi-helm
 ...
-                                       git+https://opendev.org/openstack/magnum-capi-helm@${OS_CAPI_PLUGIN_VERSION}#egg=magnum_capi_helm \
+                                       git+${OS_CAPI_PLUGIN_REPO}@${OS_CAPI_PLUGIN_VERSION}#egg=magnum_capi_helm \
```

Mặc định vẫn là opendev, build thường không đổi gì. `pip` clone đủ lịch sử, nên `pbr` tính được version (ví dụ `1.4.1.dev23`).

## Sửa workflow

1. `osCapiPluginVersion`: đổi `type: choice` sang `type: string`.
2. Thêm input `osCapiPluginRepo` và `image-tag-suffix`:

```yaml
      osCapiPluginRepo:
        description: 'Git repository of the Magnum CAPI Plugin, e.g. a fork carrying unreleased changes'
        required: false
        default: 'https://opendev.org/openstack/magnum-capi-helm'
        type: string
      image-tag-suffix:
        description: 'Tag suffix used instead of "latest", e.g. a short name for a custom plugin build'
        required: false
        type: string

env:
  capi_plugin_repo: ${{ github.event.inputs.osCapiPluginRepo || 'https://opendev.org/openstack/magnum-capi-helm' }}
  tag_suffix: ${{ github.event.inputs.image-tag-suffix }}
```

3. Ở cả hai bước `docker/build-push-action`, thêm `OS_CAPI_PLUGIN_REPO=${{ env.capi_plugin_repo }}` vào `build-args`.
4. Ở bước push, đổi tag:

```yaml
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.OS_VERSION_PARSE }}-${{ env.tag_suffix || 'latest' }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.OS_VERSION_PARSE }}-${{ env.tag_suffix && format('{0}-', env.tag_suffix) || '' }}${{ env.MY_DATE }}
```

Bản đã làm: nhánh [`capi-plugin-repo-arg`](https://github.com/saphambn/genestack-images/tree/capi-plugin-repo-arg) trên fork `saphambn/genestack-images`.

## Chạy trên fork

- Fork mới chưa bật Actions: **Settings → Actions → General → Allow all actions and reusable workflows → Save**.
- Nếu tab Actions báo *No workflows found* sau khi bật: push một commit lên nhánh mặc định (`main`) của fork để GitHub đăng ký workflow.
- **Run workflow** trên nhánh chứa thay đổi, với:
  - `openstack-constraints`: `stable/2025.1` (khớp version cloud đang chạy)
  - `osCapiPluginVersion`: `nodegroup-server-group`
  - `osCapiPluginRepo`: `https://github.com/saphambn/magnum-capi-helm`
  - `image-tag-suffix`: `sg`
- Kết quả: `ghcr.io/saphambn/genestack-images/magnum:2025.1-sg` và `...:2025.1-sg-<timestamp>`. Package mới trên GHCR mặc định là private.

## Build bằng Docker

```bash
docker buildx build -f ContainerFiles/magnum \
  --build-arg OS_VERSION=stable/2025.1 \
  --build-arg OS_CONSTRAINTS=stable/2025.1 \
  --build-arg OS_CAPI_PLUGIN_REPO=https://github.com/saphambn/magnum-capi-helm \
  --build-arg OS_CAPI_PLUGIN_VERSION=nodegroup-server-group \
  --build-arg CACHEBUST=$(date +%s) \
  -t ghcr.io/saphambn/genestack-images/magnum:2025.1-sg-$(date +%s) --push .
```

`CACHEBUST` phải đổi mỗi lần, nếu không Docker dùng lại layer `pip install` cũ.
