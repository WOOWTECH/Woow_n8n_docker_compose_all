# n8n + PostgreSQL (Podman)

使用 Podman 部署 n8n 工作流程自動化平台，搭配 PostgreSQL 資料庫。

[English Version](README.md)

## 快速開始

```bash
# 1. 複製環境變數檔案
cp .env.example .env

# 2. 編輯設定（可選）
nano .env

# 3. 啟動服務
podman-compose up -d
```

存取 n8n：`http://<伺服器IP>:5678`

## 環境變數

| 變數 | 說明 | 預設值 |
|------|------|--------|
| `POSTGRES_USER` | PostgreSQL 使用者名稱 | `n8n` |
| `POSTGRES_PASSWORD` | PostgreSQL 密碼 | `n8n_password` |
| `POSTGRES_DB` | 資料庫名稱 | `n8n` |
| `N8N_PORT` | n8n 對外埠號 | `5678` |
| `N8N_HOST` | n8n 主機名稱 | `localhost` |
| `N8N_PROTOCOL` | 協定（http/https） | `http` |
| `WEBHOOK_URL` | Webhook 回呼網址 | `http://localhost:5678/` |
| `GENERIC_TIMEZONE` | 時區 | `Asia/Taipei` |

## 常用指令

```bash
# 啟動服務
podman-compose up -d

# 停止服務
podman-compose down

# 查看日誌
podman-compose logs -f

# 只查看 n8n 日誌
podman-compose logs -f n8n

# 重啟服務
podman-compose restart

# 檢查狀態
podman-compose ps
```

## 備份與還原

### 備份

```bash
# 備份 PostgreSQL 資料
podman exec n8n-postgres pg_dump -U n8n n8n > backup_$(date +%Y%m%d_%H%M%S).sql
```

### 還原

```bash
# 還原 PostgreSQL 資料
podman exec -i n8n-postgres psql -U n8n n8n < backup_YYYYMMDD_HHMMSS.sql
```

## Cloudflare Tunnel 整合

使用 Cloudflare Tunnel 時，請更新 `.env`：

```bash
WEBHOOK_URL=https://n8n.yourdomain.com/
```

## 疑難排解

### n8n 容器持續重啟

檢查 PostgreSQL 是否健康：

```bash
podman-compose logs postgres
```

### 無法連線到資料庫

確認兩個服務的 PostgreSQL 憑證一致：

```bash
podman-compose config
```

### 權限被拒絕錯誤

確保 volumes 有正確的權限：

```bash
podman volume inspect podman_docker_app_postgres_data
podman volume inspect podman_docker_app_n8n_data
```

---

## AI 快速部署

> 此區塊專為 AI 助手快速部署 n8n 設計。

### 前置檢查

```bash
# 確認 podman-compose 已安裝
podman-compose --version
```

### 一鍵部署

```bash
cp .env.example .env && podman-compose up -d && podman-compose ps
```

### 驗證部署

```bash
# 檢查所有容器是否運行中
podman-compose ps

# 檢查 n8n 是否回應
curl -s http://localhost:5678/healthz || echo "等待 n8n 啟動..."
```

### 快速移除

```bash
# 停止並移除容器（保留資料）
podman-compose down

# 停止並移除容器和 volumes（刪除所有資料）
podman-compose down -v
```

### 設定摘要

- **n8n 網址**：`http://<伺服器IP>:5678`
- **資料庫**：PostgreSQL 16（內部，埠號 5432）
- **認證**：n8n 內建帳號系統（第一位使用者成為管理員）
- **資料持久化**：Named volumes（`postgres_data`、`n8n_data`）
- **自動重啟**：已啟用（`unless-stopped`）
