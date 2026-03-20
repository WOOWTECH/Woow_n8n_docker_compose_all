# Woow n8n

n8n — workflow automation platform for self-hosted deployment.

## Deployment Options

| Platform | Branch | Description |
|----------|--------|-------------|
| Docker / Podman | [`podman`](../../tree/podman) | Docker Compose deployment |
| Kubernetes (K3s) | [`k3s`](../../tree/k3s) | K8s manifests with Kustomize |
| Home Assistant | [`ha`](../../tree/ha) | HA add-on with one-click install |

## Quick Start

Choose your platform and switch to the corresponding branch for deployment instructions.

### Docker / Podman
```bash
git clone -b podman https://github.com/WOOWTECH/Woow_n8n_docker_compose_all.git
cd Woow_n8n_docker_compose_all
cp .env.example .env
docker compose up -d
```

### Kubernetes (K3s)
```bash
git clone -b k3s https://github.com/WOOWTECH/Woow_n8n_docker_compose_all.git
cd Woow_n8n_docker_compose_all
kubectl apply -k .
```

### Home Assistant
[![Add repository to Home Assistant](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2FWOOWTECH%2FWoow_n8n_docker_compose_all)

## Ports

| Port | Description |
|------|-------------|
| 5678 | n8n Web UI |

## License

See [LICENSE](LICENSE) for details.
