# n8n with PostgreSQL (Podman)

Deploy n8n workflow automation platform with PostgreSQL database using Podman.

[繁體中文版](README.zh-TW.md)

## Quick Start

```bash
# 1. Copy environment file
cp .env.example .env

# 2. Edit configuration (optional)
nano .env

# 3. Start services
podman-compose up -d
```

Access n8n at: `http://<SERVER_IP>:5678`

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_USER` | PostgreSQL username | `n8n` |
| `POSTGRES_PASSWORD` | PostgreSQL password | `n8n_password` |
| `POSTGRES_DB` | Database name | `n8n` |
| `N8N_PORT` | n8n exposed port | `5678` |
| `N8N_HOST` | n8n hostname | `localhost` |
| `N8N_PROTOCOL` | Protocol (http/https) | `http` |
| `WEBHOOK_URL` | Webhook callback URL | `http://localhost:5678/` |
| `GENERIC_TIMEZONE` | Timezone | `Asia/Taipei` |

## Common Commands

```bash
# Start services
podman-compose up -d

# Stop services
podman-compose down

# View logs
podman-compose logs -f

# View n8n logs only
podman-compose logs -f n8n

# Restart services
podman-compose restart

# Check status
podman-compose ps
```

## Backup & Restore

### Backup

```bash
# Backup PostgreSQL data
podman exec n8n-postgres pg_dump -U n8n n8n > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restore

```bash
# Restore PostgreSQL data
podman exec -i n8n-postgres psql -U n8n n8n < backup_YYYYMMDD_HHMMSS.sql
```

## Cloudflare Tunnel Integration

When using Cloudflare Tunnel, update `.env`:

```bash
WEBHOOK_URL=https://n8n.yourdomain.com/
```

## Troubleshooting

### n8n container keeps restarting

Check if PostgreSQL is healthy:

```bash
podman-compose logs postgres
```

### Cannot connect to database

Verify PostgreSQL credentials match in both services:

```bash
podman-compose config
```

### Permission denied errors

Ensure volumes have correct permissions:

```bash
podman volume inspect podman_docker_app_postgres_data
podman volume inspect podman_docker_app_n8n_data
```

---

## AI Quick Deploy

> This section is designed for AI assistants to quickly deploy n8n.

### Prerequisites Check

```bash
# Verify podman-compose is installed
podman-compose --version
```

### One-Command Deploy

```bash
cp .env.example .env && podman-compose up -d && podman-compose ps
```

### Verify Deployment

```bash
# Check all containers are running
podman-compose ps

# Check n8n is responding
curl -s http://localhost:5678/healthz || echo "Waiting for n8n to start..."
```

### Quick Teardown

```bash
# Stop and remove containers (keep data)
podman-compose down

# Stop and remove containers AND volumes (delete all data)
podman-compose down -v
```

### Configuration Summary

- **n8n URL**: `http://<SERVER_IP>:5678`
- **Database**: PostgreSQL 16 (internal, port 5432)
- **Authentication**: n8n built-in user system (first user becomes admin)
- **Data persistence**: Named volumes (`postgres_data`, `n8n_data`)
- **Auto-restart**: Enabled (`unless-stopped`)
