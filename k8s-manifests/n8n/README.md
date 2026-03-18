# n8n K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

Workflow automation platform with 400+ integrations, serving as a self-hosted alternative to Zapier and Make.com. n8n allows you to build complex automations connecting APIs, databases, and services through a visual node-based editor. This deployment also includes n8n-MCP, an AI-powered Model Context Protocol server that enables AI agents (such as Claude) to manage n8n workflows programmatically.

> **GitHub Repo (Podman/Docker):** [Woow_n8n_docker_compose_all](https://github.com/WOOWTECH/Woow_n8n_docker_compose_all)

### Architecture

```
                             n8n K3s Architecture
 ============================================================================

   External Access                K3s Cluster (namespace: n8n)
  +----------------+       +--------------------------------------------------+
  |                |       |                                                  |
  |  Browser       |       |   +------------------------------------------+   |
  |  :30678  ------+--NodePort->|  Service: n8n                           |   |
  |                |       |   |  ClusterIP :5678                         |   |
  +----------------+       |   +-------------------+----------------------+   |
                           |                       |                          |
  +----------------+       |                       v                          |
  |                |       |   +------------------------------------------+   |
  |  AI Agent      |       |   |  Pod: n8n (Deployment)                   |   |
  |  (Claude)      |       |   |  Image: n8nio/n8n                        |   |
  |  :30300  ------+--NodePort->|  Port: 5678                             |   |
  |                |       |   |  Volume: /home/node/.n8n (5Gi)           |   |
  +----------------+       |   |  Strategy: Recreate (SQLite lock safety) |   |
                           |   +-------------------+----------------------+   |
                           |                       ^                          |
                           |                       | n8n REST API             |
                           |                       | (cluster-internal)       |
                           |                       |                          |
                           |   +-------------------+----------------------+   |
                           |   |  Pod: n8n-mcp (Deployment)               |   |
                           |   |  Image: czlonkowski/n8n-mcp              |   |
                           |   |  Port: 3000                              |   |
                           |   |  Volume: /app/logs (1Gi)                 |   |
                           |   |                                          |   |
                           |   |  Init Container: wait-for-n8n            |   |
                           |   +------------------------------------------+   |
                           |                                                  |
                           |   Service: n8n-mcp (ClusterIP :3000)             |
                           +--------------------------------------------------+
```

### Features

- 400+ built-in integrations and visual node-based editor
- Self-hosted alternative to Zapier and Make.com
- Webhook support for event-driven workflows
- n8n-MCP server for AI agent workflow management
- SQLite database with Recreate deployment strategy
- HTTP basic authentication for secure access
- Credential encryption with configurable encryption key

### Quick Start

```bash
# 1. Update secrets before deploying
nano k8s-manifests/n8n/secret.yaml

# 2. Deploy all n8n components
kubectl apply -k k8s-manifests/n8n/

# 3. Verify pods are running
kubectl -n n8n get pods

# 4. Watch n8n startup logs
kubectl -n n8n logs deploy/n8n -f
```

### Configuration

#### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `N8N_BASIC_AUTH_ACTIVE` | Enable HTTP basic auth | `true` | No |
| `N8N_BASIC_AUTH_USER` | Basic auth username | `admin` | Yes (if auth active) |
| `N8N_HOST` | n8n host binding | `localhost` | No |
| `N8N_PORT` | n8n server port | `5678` | No |
| `N8N_PROTOCOL` | Protocol (http/https) | `http` | No |
| `WEBHOOK_URL` | Public webhook URL | `http://localhost:5678/` | Yes |
| `N8N_MODE` | Enable n8n mode for MCP | `true` | No |
| `MCP_MODE` | MCP transport mode | `http` | No |
| `N8N_API_URL` | n8n API URL for MCP | `http://n8n:5678` | Yes (for MCP) |

#### Secrets

Edit `secret.yaml` before deploying. The following secrets must be configured:

| Secret Key | Description | Default (change me!) |
|------------|-------------|----------------------|
| `N8N_BASIC_AUTH_PASSWORD` | n8n basic auth password | `changeme-n8n-password` |
| `N8N_ENCRYPTION_KEY` | Encryption key for stored credentials | `changeme-generate-with-openssl-rand-hex-32` |
| `N8N_API_KEY` | n8n REST API key (used by MCP) | `changeme-n8n-api-key` |
| `MCP_AUTH_TOKEN` | Bearer token protecting the MCP HTTP endpoint | `changeme-mcp-auth-token` |

```bash
# Generate a secure encryption key
openssl rand -hex 32

# Edit secrets
nano k8s-manifests/n8n/secret.yaml
```

**Important:** The `N8N_API_KEY` must be generated inside n8n after first login: Settings > API > Create API Key. Then update the secret and restart.

### Accessing the Service

| Endpoint | URL | Protocol |
|----------|-----|----------|
| n8n Web UI | `http://<node-ip>:30678` | HTTP (NodePort) |
| n8n-MCP API | `http://<node-ip>:30300` | HTTP (NodePort) |
| n8n Internal | `http://n8n.n8n.svc.cluster.local:5678` | HTTP |
| MCP Internal | `http://n8n-mcp.n8n.svc.cluster.local:3000` | HTTP |

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `n8n-data` | `/home/node/.n8n` | 5Gi | Workflows, credentials, SQLite database, settings |
| `mcp-logs` | `/app/logs` | 1Gi | n8n-MCP server logs |

All PVCs use the `local-path` storage class (k3s default).

### Backup & Restore

#### Backup

```bash
# 1. Export all workflows via n8n CLI
kubectl -n n8n exec deploy/n8n -- n8n export:workflow --all --output=/home/node/.n8n/backups/workflows.json

# 2. Export all credentials
kubectl -n n8n exec deploy/n8n -- n8n export:credentials --all --output=/home/node/.n8n/backups/credentials.json

# 3. Backup the entire n8n data directory
kubectl -n n8n exec deploy/n8n -- tar czf /tmp/n8n-backup.tar.gz /home/node/.n8n
kubectl -n n8n cp n8n/<n8n-pod>:/tmp/n8n-backup.tar.gz ./n8n-backup.tar.gz
```

#### Restore

```bash
# 1. Restore the data directory
kubectl -n n8n cp ./n8n-backup.tar.gz n8n/<n8n-pod>:/tmp/n8n-backup.tar.gz
kubectl -n n8n exec deploy/n8n -- tar xzf /tmp/n8n-backup.tar.gz -C /

# 2. Or import workflows individually
kubectl -n n8n exec deploy/n8n -- n8n import:workflow --input=/home/node/.n8n/backups/workflows.json

# 3. Restart n8n
kubectl -n n8n rollout restart deploy/n8n
```

### Useful Commands

```bash
# Check pod status
kubectl -n n8n get pods

# View n8n logs
kubectl -n n8n logs deploy/n8n -f

# View n8n-MCP logs
kubectl -n n8n logs deploy/n8n-mcp -f

# Restart n8n
kubectl -n n8n rollout restart deploy/n8n

# Export all workflows
kubectl -n n8n exec deploy/n8n -- n8n export:workflow --all --output=/home/node/.n8n/backups/workflows.json

# Export all credentials
kubectl -n n8n exec deploy/n8n -- n8n export:credentials --all --output=/home/node/.n8n/backups/credentials.json

# Generate a secure encryption key
openssl rand -hex 32
```

### Troubleshooting

#### n8n-MCP pod stuck in Init (waiting for n8n)

The n8n-MCP pod has an init container that waits for n8n to be healthy. Check n8n status first:

```bash
kubectl -n n8n get pods -l component=n8n
kubectl -n n8n logs deploy/n8n
```

#### Webhook URLs not working

Update `WEBHOOK_URL` in `configmap.yaml` to match your actual public URL:

```yaml
WEBHOOK_URL: "https://n8n.yourdomain.com/"
```

Then re-apply and restart:

```bash
kubectl apply -k k8s-manifests/n8n/
kubectl -n n8n rollout restart deploy/n8n
```

#### "Encryption key changed" error

The `N8N_ENCRYPTION_KEY` must remain the same across restarts. If lost, existing credentials cannot be decrypted and must be re-created.

#### n8n-MCP authentication failures

Verify the `N8N_API_KEY` in the secret matches the API key generated in n8n's UI:

```bash
# Check MCP logs for auth errors
kubectl -n n8n logs deploy/n8n-mcp -f
```

#### SQLite database locked errors

n8n uses SQLite by default. The deployment uses `strategy: Recreate` to prevent concurrent access. Ensure only one replica is running:

```bash
kubectl -n n8n get pods -l component=n8n
```

### File Structure

```
n8n/
├── configmap.yaml            # Environment variables (auth, webhook URL, MCP settings)
├── kustomization.yaml        # Kustomize resource list
├── n8n-deployment.yaml       # Deployment for n8n (Recreate strategy)
├── n8n-mcp-deployment.yaml   # Deployment for n8n-MCP (with init container)
├── n8n-mcp-service.yaml      # NodePort service for MCP (30300 -> 3000)
├── n8n-service.yaml          # NodePort service for n8n (30678 -> 5678)
├── namespace.yaml            # Namespace: n8n
├── pvc.yaml                  # PersistentVolumeClaims for n8n data and MCP logs
├── README.md                 # This file
└── secret.yaml               # Auth passwords, encryption key, API keys (change before deploy!)
```

---

## 中文

### 概述

n8n 是工作流程自動化平台，擁有 400+ 整合，作為 Zapier 和 Make.com 的自架替代方案。n8n 讓您透過視覺化節點編輯器建立連接 API、資料庫及服務的複雜自動化流程。本部署還包含 n8n-MCP，一個 AI 驅動的 Model Context Protocol 伺服器，讓 AI 代理（如 Claude）能以程式化方式管理 n8n 工作流程。

> **GitHub Repo (Podman/Docker):** [Woow_n8n_docker_compose_all](https://github.com/WOOWTECH/Woow_n8n_docker_compose_all)

### 架構圖

```
                             n8n K3s 架構
 ============================================================================

   外部存取                     K3s 叢集 (namespace: n8n)
  +----------------+       +--------------------------------------------------+
  |                |       |                                                  |
  |  瀏覽器        |       |   +------------------------------------------+   |
  |  :30678  ------+--NodePort->|  Service: n8n                           |   |
  |                |       |   |  ClusterIP :5678                         |   |
  +----------------+       |   +-------------------+----------------------+   |
                           |                       |                          |
  +----------------+       |                       v                          |
  |                |       |   +------------------------------------------+   |
  |  AI 代理       |       |   |  Pod: n8n (Deployment)                   |   |
  |  (Claude)      |       |   |  映像: n8nio/n8n                         |   |
  |  :30300  ------+--NodePort->|  埠號: 5678                             |   |
  |                |       |   |  磁碟區: /home/node/.n8n (5Gi)           |   |
  +----------------+       |   |  策略: Recreate（SQLite 鎖定安全）        |   |
                           |   +-------------------+----------------------+   |
                           |                       ^                          |
                           |                       | n8n REST API             |
                           |                       | (叢集內部)               |
                           |                       |                          |
                           |   +-------------------+----------------------+   |
                           |   |  Pod: n8n-mcp (Deployment)               |   |
                           |   |  映像: czlonkowski/n8n-mcp               |   |
                           |   |  埠號: 3000                              |   |
                           |   |  磁碟區: /app/logs (1Gi)                 |   |
                           |   |                                          |   |
                           |   |  Init Container: wait-for-n8n            |   |
                           |   +------------------------------------------+   |
                           |                                                  |
                           |   Service: n8n-mcp (ClusterIP :3000)             |
                           +--------------------------------------------------+
```

### 功能特色

- 400+ 內建整合與視覺化節點編輯器
- Zapier 和 Make.com 的自架替代方案
- Webhook 支援事件驅動工作流程
- n8n-MCP 伺服器用於 AI 代理工作流程管理
- SQLite 資料庫搭配 Recreate 部署策略
- HTTP 基本認證確保安全存取
- 可設定加密金鑰的憑證加密

### 快速開始

```bash
# 1. 部署前先更新密鑰
nano k8s-manifests/n8n/secret.yaml

# 2. 部署所有 n8n 元件
kubectl apply -k k8s-manifests/n8n/

# 3. 確認 Pod 運作中
kubectl -n n8n get pods

# 4. 查看 n8n 啟動日誌
kubectl -n n8n logs deploy/n8n -f
```

### 設定

#### 環境變數

| 變數 | 說明 | 預設值 | 必填 |
|------|------|--------|------|
| `N8N_BASIC_AUTH_ACTIVE` | 啟用 HTTP 基本認證 | `true` | 否 |
| `N8N_BASIC_AUTH_USER` | 基本認證使用者名稱 | `admin` | 是（若啟用認證） |
| `N8N_HOST` | n8n 主機繫結 | `localhost` | 否 |
| `N8N_PORT` | n8n 伺服器埠號 | `5678` | 否 |
| `N8N_PROTOCOL` | 協定（http/https） | `http` | 否 |
| `WEBHOOK_URL` | 公開 Webhook URL | `http://localhost:5678/` | 是 |
| `N8N_MODE` | 為 MCP 啟用 n8n 模式 | `true` | 否 |
| `MCP_MODE` | MCP 傳輸模式 | `http` | 否 |
| `N8N_API_URL` | MCP 用的 n8n API URL | `http://n8n:5678` | 是（MCP 用） |

#### Secrets（密鑰）

部署前請編輯 `secret.yaml`，需設定以下密鑰：

| Secret Key | 說明 | 預設值（請變更！） |
|------------|------|-------------------|
| `N8N_BASIC_AUTH_PASSWORD` | n8n 基本認證密碼 | `changeme-n8n-password` |
| `N8N_ENCRYPTION_KEY` | 儲存憑證的加密金鑰 | `changeme-generate-with-openssl-rand-hex-32` |
| `N8N_API_KEY` | n8n REST API 金鑰（MCP 使用） | `changeme-n8n-api-key` |
| `MCP_AUTH_TOKEN` | 保護 MCP HTTP 端點的 Bearer Token | `changeme-mcp-auth-token` |

```bash
# 產生安全的加密金鑰
openssl rand -hex 32

# 編輯密鑰
nano k8s-manifests/n8n/secret.yaml
```

**重要：** `N8N_API_KEY` 必須在首次登入 n8n 後產生：Settings > API > Create API Key。然後更新 Secret 並重啟。

### 存取服務

| 端點 | URL | 協定 |
|------|-----|------|
| n8n Web UI | `http://<node-ip>:30678` | HTTP (NodePort) |
| n8n-MCP API | `http://<node-ip>:30300` | HTTP (NodePort) |
| n8n 叢集內部 | `http://n8n.n8n.svc.cluster.local:5678` | HTTP |
| MCP 叢集內部 | `http://n8n-mcp.n8n.svc.cluster.local:3000` | HTTP |

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|----------|----------|------|------|
| `n8n-data` | `/home/node/.n8n` | 5Gi | 工作流程、憑證、SQLite 資料庫、設定 |
| `mcp-logs` | `/app/logs` | 1Gi | n8n-MCP 伺服器日誌 |

所有 PVC 使用 `local-path` 儲存類別（k3s 預設）。

### 備份與還原

#### 備份

```bash
# 1. 透過 n8n CLI 匯出所有工作流程
kubectl -n n8n exec deploy/n8n -- n8n export:workflow --all --output=/home/node/.n8n/backups/workflows.json

# 2. 匯出所有憑證
kubectl -n n8n exec deploy/n8n -- n8n export:credentials --all --output=/home/node/.n8n/backups/credentials.json

# 3. 備份整個 n8n 資料目錄
kubectl -n n8n exec deploy/n8n -- tar czf /tmp/n8n-backup.tar.gz /home/node/.n8n
kubectl -n n8n cp n8n/<n8n-pod>:/tmp/n8n-backup.tar.gz ./n8n-backup.tar.gz
```

#### 還原

```bash
# 1. 還原資料目錄
kubectl -n n8n cp ./n8n-backup.tar.gz n8n/<n8n-pod>:/tmp/n8n-backup.tar.gz
kubectl -n n8n exec deploy/n8n -- tar xzf /tmp/n8n-backup.tar.gz -C /

# 2. 或個別匯入工作流程
kubectl -n n8n exec deploy/n8n -- n8n import:workflow --input=/home/node/.n8n/backups/workflows.json

# 3. 重啟 n8n
kubectl -n n8n rollout restart deploy/n8n
```

### 實用指令

```bash
# 查看 Pod 狀態
kubectl -n n8n get pods

# 查看 n8n 日誌
kubectl -n n8n logs deploy/n8n -f

# 查看 n8n-MCP 日誌
kubectl -n n8n logs deploy/n8n-mcp -f

# 重啟 n8n
kubectl -n n8n rollout restart deploy/n8n

# 匯出所有工作流程
kubectl -n n8n exec deploy/n8n -- n8n export:workflow --all --output=/home/node/.n8n/backups/workflows.json

# 匯出所有憑證
kubectl -n n8n exec deploy/n8n -- n8n export:credentials --all --output=/home/node/.n8n/backups/credentials.json

# 產生安全的加密金鑰
openssl rand -hex 32
```

### 疑難排解

#### n8n-MCP Pod 停在 Init（等待 n8n）

n8n-MCP Pod 有 Init Container 等待 n8n 健康就緒。請先檢查 n8n 狀態：

```bash
kubectl -n n8n get pods -l component=n8n
kubectl -n n8n logs deploy/n8n
```

#### Webhook URL 無法運作

更新 `configmap.yaml` 中的 `WEBHOOK_URL` 以匹配您的實際公開 URL：

```yaml
WEBHOOK_URL: "https://n8n.yourdomain.com/"
```

然後重新套用並重啟：

```bash
kubectl apply -k k8s-manifests/n8n/
kubectl -n n8n rollout restart deploy/n8n
```

#### 「Encryption key changed」錯誤

`N8N_ENCRYPTION_KEY` 在重啟間必須保持不變。若遺失，現有憑證將無法解密，必須重新建立。

#### n8n-MCP 認證失敗

確認 Secret 中的 `N8N_API_KEY` 與 n8n UI 中產生的 API Key 一致：

```bash
# 檢查 MCP 日誌中的認證錯誤
kubectl -n n8n logs deploy/n8n-mcp -f
```

#### SQLite 資料庫鎖定錯誤

n8n 預設使用 SQLite。部署使用 `strategy: Recreate` 防止並行存取。確認只有一個副本在運行：

```bash
kubectl -n n8n get pods -l component=n8n
```

### 檔案結構

```
n8n/
├── configmap.yaml            # 環境變數（認證、Webhook URL、MCP 設定）
├── kustomization.yaml        # Kustomize 資源列表
├── n8n-deployment.yaml       # n8n 的 Deployment（Recreate 策略）
├── n8n-mcp-deployment.yaml   # n8n-MCP 的 Deployment（含 Init Container）
├── n8n-mcp-service.yaml      # MCP 的 NodePort 服務 (30300 -> 3000)
├── n8n-service.yaml          # n8n 的 NodePort 服務 (30678 -> 5678)
├── namespace.yaml            # 命名空間: n8n
├── pvc.yaml                  # 持久卷宣告（n8n 資料與 MCP 日誌）
├── README.md                 # 本文件
└── secret.yaml               # 認證密碼、加密金鑰、API 金鑰（部署前請變更！）
```
