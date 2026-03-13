# Skill: Deploy n8n + PostgreSQL (Podman)

## Metadata

- **Name**: deploy-n8n-postgres
- **Description**: Deploy n8n workflow automation with PostgreSQL on Podman
- **Trigger**: User asks to deploy n8n, set up n8n, or run n8n with Docker/Podman

## Prerequisites

- Podman v4.0+
- podman-compose v1.0+
- Repository: https://github.com/WOOWTECH/Woow_n8n_docker_compose_all.git

## Deployment Steps

### 1. Clone and configure

```bash
git clone https://github.com/WOOWTECH/Woow_n8n_docker_compose_all.git
cd Woow_n8n_docker_compose_all
cp .env.example .env
```

### 2. Set server IP in .env

```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
sed -i "s|WEBHOOK_URL=http://localhost:15678/|WEBHOOK_URL=http://${SERVER_IP}:15678/|" .env
```

### 3. (Optional) Set secure password

```bash
sed -i "s|POSTGRES_PASSWORD=change_me_to_secure_password|POSTGRES_PASSWORD=YOUR_SECURE_PASSWORD|" .env
```

### 4. Start services

```bash
podman-compose up -d
```

### 5. Verify

```bash
sleep 10
podman-compose ps
curl -s http://localhost:15678/healthz
# Expected: {"status":"ok"}
```

### 6. Access

Open `http://<SERVER_IP>:15678` in browser. First user registered becomes admin.

## Architecture

```
Services:
  postgres (postgres:16-alpine)
    - Internal port 5432
    - Named volume: postgres_data
    - Healthcheck: pg_isready

  n8n (n8nio/n8n:latest)
    - Exposed port: 15678 -> 5678
    - Named volume: n8n_data
    - DB_TYPE: postgresdb
    - N8N_SECURE_COOKIE: false (HTTP login)
    - N8N_HOST: 0.0.0.0 (allow LAN access)
    - depends_on: postgres (healthy)

Network: n8n-network
```

## Files Reference

### docker-compose.yml

```yaml
name: n8n

services:
  postgres:
    image: postgres:16-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-n8n}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-n8n_password}
      POSTGRES_DB: ${POSTGRES_DB:-n8n}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-n8n} -d ${POSTGRES_DB:-n8n}"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "${N8N_PORT:-5678}:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB:-n8n}
      - DB_POSTGRESDB_USER=${POSTGRES_USER:-n8n}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD:-n8n_password}
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - WEBHOOK_URL=${WEBHOOK_URL:-http://localhost:5678/}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE:-Asia/Taipei}
      - N8N_SECURE_COOKIE=false
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
  n8n_data:

networks:
  default:
    name: n8n-network
```

### .env.example

```env
# PostgreSQL Configuration
POSTGRES_USER=n8n
POSTGRES_PASSWORD=change_me_to_secure_password
POSTGRES_DB=n8n

# n8n Configuration
N8N_PORT=15678
N8N_HOST=0.0.0.0
N8N_PROTOCOL=http

# Webhook URL
# For internal network: http://<SERVER_IP>:15678/
# For Cloudflare Tunnel: https://n8n.yourdomain.com/
WEBHOOK_URL=http://localhost:15678/

# Timezone
GENERIC_TIMEZONE=Asia/Taipei
```

## Common Operations

### Stop (keep data)

```bash
podman-compose down
```

### Stop and delete all data

```bash
podman-compose down -v
```

### Backup PostgreSQL

```bash
podman exec n8n-postgres pg_dump -U n8n n8n > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restore PostgreSQL

```bash
podman exec -i n8n-postgres psql -U n8n n8n < backup_FILE.sql
```

### Update n8n to latest

```bash
podman-compose pull n8n && podman-compose up -d
```

### View logs

```bash
podman-compose logs -f n8n
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Cannot login via HTTP | Ensure `N8N_SECURE_COOKIE=false` in docker-compose.yml |
| Port conflict | Change `N8N_PORT` in .env, restart |
| Cannot access from LAN | Ensure `N8N_HOST=0.0.0.0` in .env |
| Container keeps restarting | Check logs: `podman-compose logs postgres` |
| Cloudflare Tunnel | Update `WEBHOOK_URL` to `https://your.domain.com/` |

## Port Change

If port 15678 is occupied, edit `.env`:

```bash
N8N_PORT=25678
```

Then: `podman-compose down && podman-compose up -d`
