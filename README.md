# Podman Monitoring Stack

A lightweight local observability stack for monitoring Podman containers and host resources.

Designed for local development, troubleshooting, performance testing, and collecting resource baselines before production deployment.

## Stack

- **Grafana** — Dashboards and visualization
- **Prometheus** — Metrics collection and storage
- **Podman Exporter** — Container CPU, memory, network, and status metrics
- **Node Exporter** — Host CPU, memory, disk, and network metrics
- **Loki** — Log aggregation
- **Grafana Alloy** — Podman container log collection

## Architecture

```text id="w4ou9a"
Podman Containers
       │
       ├── Podman Exporter ──→ Prometheus ──┐
       │                                    │
Host ────── Node Exporter ────→ Prometheus ─┤
                                            ├──→ Grafana
Podman Containers ──→ Alloy ──→ Loki ──────┘
```

## Requirements

- Linux
- Podman
- `podman compose` or `podman-compose`
- systemd user services

Rootless Podman is supported.

## Quick Start

### 1. Enable Podman API Socket

```bash id="vb6stc"
systemctl --user enable --now podman.socket
```

Verify:

```bash id="2z1m1a"
systemctl --user status podman.socket
```

The rootless Podman socket is normally available at:

```bash id="8i20ko"
echo "$XDG_RUNTIME_DIR/podman/podman.sock"
```

Test the API:

```bash id="vb6qyu"
curl --unix-socket "$XDG_RUNTIME_DIR/podman/podman.sock" \
  http://d/_ping
```

Expected result:

```text id="rd4x86"
OK
```

### 2. Configure Socket Permissions

Podman may create the rootless socket with owner-only permissions:

```text id="6ht50b"
srw------- podman.sock
```

This prevents monitoring containers from accessing the API.

For local development, allow access to the socket:

```bash id="g6nklq"
chmod 666 "$XDG_RUNTIME_DIR/podman/podman.sock"
```

Verify:

```bash id="c8iyjn"
ls -ln "$XDG_RUNTIME_DIR/podman/podman.sock"
```

> **Security note:** `0666` should only be used for a trusted local development environment. Access to the Podman socket effectively provides control over the user's containers. For shared or production systems, use restricted group permissions such as `0660`.

### 3. Verify Container Access

Before starting the monitoring stack, verify that a rootless container can access the Podman API:

```bash id="xaxunw"
podman run --rm \
  --userns=keep-id \
  --security-opt label=disable \
  -v "$XDG_RUNTIME_DIR/podman/podman.sock:/run/podman/podman.sock" \
  docker.io/curlimages/curl:latest \
  --unix-socket /run/podman/podman.sock \
  http://d/_ping
```

Expected result:

```text id="a8zggc"
OK
```

### 4. Start Monitoring

```bash id="6g2pdq"
podman compose up -d
```

Or:

```bash id="9d7cpw"
podman-compose up -d
```

Check all services:

```bash id="c6l2hb"
podman ps
```

## Access

| Service | URL |
|---|---|
| Grafana | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |
| Loki | `http://localhost:3100` |

Default Grafana credentials:

```text id="r6e38k"
Username: admin
Password: admin
```

## Verify Metrics

Check Prometheus targets:

```promql id="r8mnn3"
up
```

Expected jobs include:

```text id="6xq85i"
prometheus
node
podman
```

Check Podman metrics:

```promql id="ulokv7"
podman_container_info
```

Container CPU:

```promql id="84m2h8"
rate(podman_container_cpu_seconds_total[1m])
```

Container memory:

```promql id="7yld5i"
podman_container_mem_usage_bytes
```

You can also verify the exporter directly from the Prometheus container:

```bash id="86m6si"
podman exec monitor-prometheus \
  wget -qO- http://podman-exporter:9882/metrics | head -30
```

## Verify Logs

Open:

```text id="yct51j"
Grafana → Explore → Loki
```

Query:

```logql id="a8vmg6"
{container=~".+"}
```

## Stop

Stop the monitoring stack:

```bash id="dhfprf"
podman compose down
```

Remove the stack and monitoring data:

```bash id="wmmz3h"
podman compose down -v
```

## Troubleshooting

### Podman socket permission denied

Error:

```text id="5g92mp"
unable to connect to Podman socket:
dial unix /run/podman/podman.sock: connect: permission denied
```

Check the socket:

```bash id="78m8ij"
ls -ln "$XDG_RUNTIME_DIR/podman/podman.sock"
```

Test access from the host:

```bash id="ph9grl"
curl --unix-socket "$XDG_RUNTIME_DIR/podman/podman.sock" \
  http://d/_ping
```

Then test access from a container:

```bash id="m9i0fw"
podman run --rm \
  --userns=keep-id \
  --security-opt label=disable \
  -v "$XDG_RUNTIME_DIR/podman/podman.sock:/run/podman/podman.sock" \
  docker.io/curlimages/curl:latest \
  --unix-socket /run/podman/podman.sock \
  http://d/_ping
```

Both should return:

```text id="6gqzq7"
OK
```

## Use Cases

This stack can be used as a local performance lab to measure:

- Container CPU and memory consumption
- Host CPU, RAM, disk, and network utilization
- Container network traffic
- Application and infrastructure logs
- Resource behavior during load testing
- Baseline resource requirements for production sizing

A typical workflow is:

```text id="2hz4nk"
Load Test
    ↓
Grafana Metrics
    ↓
Identify Bottlenecks
    ↓
Measure Resource Consumption
    ↓
Estimate Production Resources
    ↓
Validate with Production/Staging Load Test
```

Local measurements should be treated as a baseline rather than directly converted into production sizing because hardware, storage, networking, virtualization, and workload characteristics may differ.
