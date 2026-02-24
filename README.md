# Monitoring Stack

A Docker-based monitoring stack using Prometheus, Grafana, Loki, and Fluent Bit. Designed for multi-host monitoring over a Tailscale network.

## Requirements

There are a number of requirements/presumptions in order to run this project:
- An existing Docker setup
- A functioning Tailscale network, and the associated IP addresses for each host, is required for this stack to work.
- A revese proxy setup of some form - I use Traefik, and Grafana has the relevant tags as a result, but you could sub out any of the other Docker-focused reverse proxies.

## Architecture

```
Monitored Hosts (nodes)              Central Server
┌─────────────────────┐              ┌──────────────────────────┐
│  Node Exporter :9100│──(metrics)──>│  Prometheus :9090        │
│  cAdvisor      :8080│──(metrics)──>│  Grafana    :3000        │
│  Fluent Bit         │──(logs)─────>│  Loki       :3100        │
└─────────────────────┘              │  Node Exporter / cAdvisor│
                                     │  Fluent Bit              │
                                     └──────────────────────────┘
```

## Components

| Component | Purpose | Port |
|-----------|---------|------|
| **Prometheus** | Metrics collection and storage (20GB retention) | 9090 |
| **Grafana** | Dashboards and visualization | 3000 |
| **Loki** | Log aggregation | 3100 |
| **Fluent Bit** | Log shipping (journald + Docker container logs) | - |
| **Node Exporter** | Host system metrics | 9100 |
| **cAdvisor** | Docker container metrics | 8080 |

## Deployment

Stacks are managed by **Portainer** and automatically redeployed via GitHub Actions when configuration changes are pushed to `main`.

There are two compose files:

- **`docker-compose-server.yml`** -- Full stack (Prometheus, Grafana, Loki, exporters, Fluent Bit) — runs on pimento
- **`docker-compose-node.yml`** -- Exporters only (Node Exporter, cAdvisor, Fluent Bit) — runs on each monitored host

### How it works

1. Push changes to `main` that touch compose files, `prometheus/`, or `fluent-bit/` configs
2. GitHub Actions workflow (`.github/workflows/deploy.yml`) fires
3. Workflow POSTs to Portainer webhook URLs for each stack (pimento, chipotle, dev.dailyword)
4. Portainer pulls the latest repo and redeploys the stack on each host

### Initial setup — Portainer agent

Each remote host needs the Portainer agent running so Portainer can manage it:

```bash
cd agent/
docker compose up -d
```

Then register the host as an environment in the Portainer UI.

### Initial setup — creating stacks in Portainer

For each host, create a stack in Portainer:

1. **Source**: Git repository → this repo
2. **Compose file**:
   - pimento (server): `docker-compose-server.yml`
   - chipotle / dev.dailyword (nodes): `docker-compose-node.yml`
3. **Environment variables**: Set the required vars from `.env.example` (e.g. `HOSTNAME`, `TAILSCALE_IP`, `NODE_HOSTNAME`, `CADVISOR_PORT`)
4. **Enable webhook** on the stack and copy the webhook URL
5. **Pimento only**: Ensure the `proxy` Docker network exists (`docker network create proxy`)

### Initial setup — GitHub Actions secrets

Add these secrets to the GitHub repo:

| Secret | Purpose |
|--------|---------|
| `CF_ACCESS_CLIENT_ID` | Cloudflare Access service token (for Portainer tunnel) |
| `CF_ACCESS_CLIENT_SECRET` | Cloudflare Access service token secret |
| `PORTAINER_WEBHOOK_PIMENTO` | Webhook URL for the server stack |
| `PORTAINER_WEBHOOK_CHIPOTLE` | Webhook URL for the chipotle node stack |
| `PORTAINER_WEBHOOK_DAILYWORD` | Webhook URL for the dev.dailyword node stack |

### Adding a new host

1. Deploy the Portainer agent on the new host (`agent/docker-compose.yml`)
2. Register the host as an environment in Portainer
3. Create a stack in Portainer pointing to this repo with `docker-compose-node.yml`
4. Set environment variables (`TAILSCALE_IP`, `NODE_HOSTNAME`, `CADVISOR_PORT`)
5. Enable the webhook and copy the URL
6. Add the webhook URL as a new secret in GitHub (e.g. `PORTAINER_WEBHOOK_NEWHOST`)
7. Add the new host to the matrix in `.github/workflows/deploy.yml`
8. Add the host's Tailscale IP to `prometheus/prometheus.yml` under the scrape jobs
9. Reload Prometheus: `curl -X POST http://localhost:9090/-/reload`

## Configuration files

| File | Purpose |
|------|---------|
| `prometheus/prometheus.yml` | Scrape targets and intervals |
| `fluent-bit/fluent-bit.conf` | Server log collection config |
| `fluent-bit/fluent-bit-node.conf` | Node log collection config (ships to central Loki) |
| `fluent-bit/parsers.conf` | Log format parsers (Docker JSON, syslog) |

## Network

All hosts communicate over Tailscale. cAdvisor is bound to each host's Tailscale IP to prevent public access. Traefik labels are included for optional HTTPS reverse proxy access to Prometheus, Grafana, and Loki.
