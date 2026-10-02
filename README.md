# Podman Monitoring Stack

A lightweight local observability stack for monitoring Podman containers and host resources.

It provides container metrics, host metrics, and centralized logs through Grafana.

## Components

- **Grafana** — Dashboards and data visualization
- **Prometheus** — Metrics collection and storage
- **Podman Exporter** — Podman container metrics
- **Node Exporter** — Host CPU, memory, disk, and network metrics
- **Loki** — Log aggregation
- **Grafana Alloy** — Collects Podman container logs

## Architecture

```text
Podman Containers
       │
       ├── Podman Exporter ──→ Prometheus ──┐
       │                                    │
Host ──┴── Node Exporter ─────→ Prometheus ─┤
                                            ├──→ Grafana
Podman Containers ──→ Alloy ──→ Loki ──────┘
```

## Requirements

- Linux
- Podman
- Podman Compose or `podman-compose`
- Rootless Podman is supported

## Setup

Enable the Podman API socket:

```bash
systemctl --user enable --now podman.socket
```

Verify it:

```bash
systemctl --user status podman.socket
ls -lah "$XDG_RUNTIME_DIR/podman/podman.sock"
```

Start the monitoring stack:

```bash
podman compose up -d
```

Alternatively:

```bash
podman-compose up -d
```

Check the containers:

```bash
podman ps
```

## Access

| Service | URL |
|---|---|
| Grafana | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |
| Loki | `http://localhost:3100` |

Default Grafana credentials:

```text
Username: admin
Password: admin
```

## Verify Metrics

Open Prometheus or Grafana Explore and query:

```promql
up
```

Podman container information:

```promql
podman_container_info
```

Container memory usage:

```promql
podman_container_mem_usage_bytes
```

Container CPU usage:

```promql
rate(podman_container_cpu_seconds_total[1m])
```

## Verify Logs

Open **Grafana → Explore → Loki** and query:

```logql
{container=~".+"}
```

This displays logs collected from Podman containers.

## Stop the Stack

```bash
podman compose down
```

To also remove monitoring data:

```bash
podman compose down -v
```

## Purpose

This stack is intended for local development, troubleshooting, performance testing, and resource monitoring.

It can be used to establish resource-consumption baselines before proposing CPU, memory, and scaling requirements for production environments.
