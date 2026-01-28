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

There are two compose files:

- **`docker-compose-server.yml`** -- Full stack (Prometheus, Grafana, Loki, exporters, Fluent Bit)
- **`docker-compose-node.yml`** -- Exporters only (Node Exporter, cAdvisor, Fluent Bit) for monitored hosts

### Server setup

1. Copy `.env.example` to `.env` and fill in your values:
   ```bash
   cp .env.example .env
   ```

2. Set `COMPOSE_FILE=docker-compose-server.yml` and configure `HOSTNAME` for Traefik routing.

3. Create the external Docker network:
   ```bash
   docker network create proxy
   ```

4. Start the stack:
   ```bash
   docker compose up -d
   ```

### Node setup

Copy the repository to each monitored host, then:

1. Create a `.env` file:
   ```
   COMPOSE_FILE=docker-compose-node.yml
   TAILSCALE_IP=100.x.x.x
   NODE_HOSTNAME=myhost
   ```

2. Optionally set `CADVISOR_PORT` if port 8080 conflicts (defaults to 8080).

3. Start:
   ```bash
   docker compose up -d
   ```

### Adding a new host

1. Deploy the node compose on the new host (see above).
2. Add the host's Tailscale IP to `prometheus/prometheus.yml` under the `node-exporter` and `cadvisor` scrape jobs.
3. Reload Prometheus:
   ```bash
   curl -X POST http://localhost:9090/-/reload
   ```

## Configuration files

| File | Purpose |
|------|---------|
| `prometheus/prometheus.yml` | Scrape targets and intervals |
| `fluent-bit/fluent-bit.conf` | Server log collection config |
| `fluent-bit/fluent-bit-node.conf` | Node log collection config (ships to central Loki) |
| `fluent-bit/parsers.conf` | Log format parsers (Docker JSON, syslog) |

## Network

All hosts communicate over Tailscale. cAdvisor is bound to each host's Tailscale IP to prevent public access. Traefik labels are included for optional HTTPS reverse proxy access to Prometheus, Grafana, and Loki.
