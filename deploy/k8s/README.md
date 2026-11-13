# Kubernetes 部署

基于 kustomize，两种模式：

| 模式 | 命令 | 说明 |
|---|---|---|
| 外部数据库（默认） | `kubectl apply -k deploy/k8s/base` | 只部署应用，MySQL/Redis 用现成的 |
| 内置数据库 | `kubectl apply -k deploy/k8s/overlays/bundled-db` | 额外部署 MySQL 8.0 + Redis 8.0（对应 docker-compose_all） |

## 部署前必改

1. **`base/secret.yaml`**：填真实密码（`SPRING_DATASOURCE_DRUID_PASSWORD` / `SPRING_DATA_REDIS_PASSWORD`），不要提交真实密码。
2. **`base/config.yaml`**：改 MySQL/Redis 地址。`SPRING_DATASOURCE_DRUID_URL` 里的库名固定为 `xiaozhi_esp32_server`（Liquibase 启动自动建表，库本身需提前创建，内置模式下 MYSQL_DATABASE 自动建）。

## 组件

| 服务 | 镜像 | 端口 | 说明 |
|---|---|---|---|
| xiaozhi-server | `:server_latest` | 8000(ws) / 8003(http) | PVC `xiaozhi-server-data`：subPath `data/` 放 `.config.yaml`，subPath `models/` 放模型 |
| xiaozhi-manager-api | `:web_latest` | 8002(智控台) / 8003(api) | PVC `xiaozhi-uploadfile` |

## 首次启动

- 模型由 initContainer 自动从 ModelScope 下载 SenseVoiceSmall（约 900MB，已存在则跳过）。
- python 服务首次启动会在 PVC 的 `data/` 下生成 `.config.yaml` 模板，`kubectl -n xiaozhi exec deploy/xiaozhi-server -- vi /opt/xiaozhi-esp32-server/data/.config.yaml` 填好 API key 等，其中 manager-api 地址填 `http://xiaozhi-manager-api:8003`（或走 8002 nginx），然后 `kubectl -n xiaozhi rollout restart deploy/xiaozhi-server`。
- 智控台入口：`kubectl -n xiaozhi port-forward svc/xiaozhi-manager-api 8002:8002`，或自己加 Ingress/NodePort。

## 国内环境换镜像源

新建 overlay：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base        # 或 ../../overlays/bundled-db
images:
  - name: ghcr.io/xinnan-tech/xiaozhi-esp32-server
    newName: ghcr.nju.edu.cn/xinnan-tech/xiaozhi-esp32-server
```
