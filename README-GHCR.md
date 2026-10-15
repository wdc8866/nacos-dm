# GitHub Container Registry (GHCR) 多架构镜像构建指南

## 目录结构
```
nacos/
├── nacos-dm-datasource-plugin-ext.jar   # 插件 shaded 包（含 base + DM 驱动）
├── DmJdbcDriver18.jar                   # DM JDBC 驱动单独包
├── .github/
│   ├── Dockerfile                       # 镜像定义
│   └── workflows/
│       └── ghcr-build.yml               # GitHub Actions 工作流（推送到 GHCR）
└── README-GHCR.md                       # 本文件
```

## 1. 无需额外配置，直接用 GitHub 自带 GHCR

- **Registry**：`ghcr.io`
- **认证**：GitHub Actions 自带 `GITHUB_TOKEN`，权限在 workflow 里已声明 `packages: write`
- **可见性**：默认私有（仅你可见），可在包设置里改为公开
- **多架构**：原生支持 `linux/amd64,linux/arm64`，无需额外开通

## 2. 推送代码触发构建

```bash
cd C:/Users/work/Documents/workspace/nacos
git init
git add .
git commit -m "init: nacos-dm plugin jars + Dockerfile + GHCR workflow"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

**首次推送会自动触发构建**（或去 Actions 页手动 Run workflow）。

也可以打 tag 触发语义化版本：
```bash
git tag v3.2.2-dm-1
git push origin v3.2.2-dm-1
```

## 3. 构建产物

构建成功后，在 GitHub 仓库主页右侧 **Packages** 或访问：
```
ghcr.io/<你的用户名>/nacos-dm
```
会出现以下 tag（均为多架构 manifest list，含 `linux/amd64` + `linux/arm64`）：
```
latest
v3.2.2-dm-1
v3.2.2
```

## 4. k8s 部署使用

### 4.1 公开包（无需 secret）
```yaml
nacos:
  image:
    repository: ghcr.io/yourname/nacos-dm
    tag: "v3.2.2-dm-1"
```

### 4.2 私有包（需 imagePullSecrets）
```bash
# 创建 secret（用你的 GitHub Personal Access Token，需包含 read:packages 权限）
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<你的GitHub用户名> \
  --docker-password=<GitHub PAT> \
  --namespace=<你的命名空间>
```

```yaml
nacos:
  image:
    repository: ghcr.io/yourname/nacos-dm
    tag: "v3.2.2-dm-1"
  imagePullSecrets:
    - name: ghcr-secret
```

## 5. 环境变量配置（部署时注入）

```yaml
env:
  - name: SPRING_DATASOURCE_PLATFORM
    value: "dm"
  - name: NACOS_PLUGIN_DATASOURCE_DB_NUM
    value: "1"
  - name: NACOS_PLUGIN_DATASOURCE_DB_URL_0
    value: "jdbc:dm://<dm-host>:5236/DMSERVER?schema=NACOS&compatibleMode=mysql&ignoreCase=true&ENCODING=utf-8"
  - name: NACOS_PLUGIN_DATASOURCE_DB_USER_0
    valueFrom:
      secretKeyRef:
        name: nacos-dm-secret
        key: username
  - name: NACOS_PLUGIN_DATASOURCE_DB_PASSWORD_0
    valueFrom:
      secretKeyRef:
        name: nacos-dm-secret
        key: password
  - name: NACOS_PLUGIN_DATASOURCE_DB_POOL_CONFIG_DRIVERCLASSNAME
    value: "dm.jdbc.driver.DmDriver"
  - name: NACOS_PLUGIN_DATASOURCE_LOG_ENABLED
    value: "true"
```

## 6. 前置条件：DM 必须是 MySQL 兼容模式

```bash
# 新建实例（端口如 5237）
dminit PATH=/data/dm_mysql MODE=MPP COMPATIBLE_MODE=4 CASE_SENSITIVE=N PORT=5237
# 建库建用户
disql SYSDBA/SYSDBA @localhost:5237 -c "CREATE SCHEMA NACOS;"
# 跑建表脚本（从插件 jar 解压）
jar xf nacos-dm-datasource-plugin-ext.jar schema/nacos-dm.sql
disql SYSDBA/SYSDBA @localhost:5237 -f nacos-dm.sql
```

## 7. 常见问题

| 问题 | 解决 |
|------|------|
| 推送提示 `denied: installation not allowed` | 仓库 Settings → Actions → General → Workflow permissions 勾选 **Read and write packages** |
| ARM 节点拉取失败 `no matching manifest` | 确认 workflow 里 `platforms: linux/amd64,linux/arm64` 且构建日志显示两架构都 pushed |
| 启动报 `ClassNotFoundException: dm.jdbc.driver.DmDriver` | 确认 `DmJdbcDriver18.jar` 在 `/home/nacos/plugins/`，且版本为 8.1.3.62 |
| 表建不起来 | 确认 DM 实例 `COMPATIBLE_MODE=4`（MySQL 模式），`CASE_SENSITIVE=N` |
| 包不可见/拉不下来 | 仓库主页 Packages 点进包 → Package settings → **Change visibility** 改为 Public，或配置 `imagePullSecrets` |

## 8. 如果一定要用阿里云 ACR

个人版确实不支持多架构。两个选项：
1. **企业版**（付费）支持多架构
2. **GHCR 构建好后，手动同步到 ACR**：
   ```bash
   # 本地或 CI 里跑
   docker pull ghcr.io/yourname/nacos-dm:v3.2.2-dm-1
   docker tag ghcr.io/yourname/nacos-dm:v3.2.2-dm-1 registry.cn-hangzhou.aliyuncs.com/your-ns/nacos-dm:v3.2.2-dm-1
   docker push registry.cn-hangzhou.aliyuncs.com/your-ns/nacos-dm:v3.2.2-dm-1
   ```
   但这会丢失多架构 manifest（变成单架构），**不推荐**。

---

**一句话**：把这目录推 GitHub，啥 Secret 都不用加（用自带 `GITHUB_TOKEN`），push 完 GHCR 直接给你现成的 `amd64+arm64` 双架构镜像。