# Self-Hosted Server Monitoring with Grafana, Alloy, and Prometheus Exporters

A complete walkthrough for setting up a production-grade Grafana monitoring stack on a self-hosted Ubuntu 24.04 server. This guide covers host metrics, Docker containers, databases, web servers, and application-level observability — all wired together using Grafana Alloy as the collection agent.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [What's Already Installed](#whats-already-installed)
3. [Node Exporter — Host Metrics](#1-node-exporter--host-metrics)
4. [cAdvisor — Docker Container Metrics](#2-cadvisor--docker-container-metrics)
5. [postgres_exporter — PostgreSQL Metrics](#3-postgres_exporter--postgresql-metrics)
6. [redis_exporter — Redis Metrics](#4-redis_exporter--redis-metrics)
7. [apache_exporter — Apache2 Metrics](#5-apache_exporter--apache2-metrics)
8. [mysqld_exporter — MySQL Metrics (Ghost Blog)](#6-mysqld_exporter--mysql-metrics-ghost-blog)
9. [Blackbox Exporter — HTTP Uptime Probes](#7-blackbox-exporter--http-uptime-probes)
10. [Ollama Metrics](#8-ollama-metrics)
11. [Configuring Grafana Alloy](#9-configuring-grafana-alloy)
12. [Importing Dashboards into Grafana](#10-importing-dashboards-into-grafana)
13. [Optional: Loki for Log Aggregation](#11-optional-loki-for-log-aggregation)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Ubuntu 24.04 Host                                                  │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ node_exporter│  │  cAdvisor    │  │ pg_exporter  │  ...more    │
│  │  :9100       │  │  :9093 (dock)│  │  :9187       │  exporters  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                  │                     │
│         └─────────────────┴──────────────────┘                    │
│                           │  scrape                                │
│                    ┌──────▼──────┐                                 │
│                    │ Grafana     │                                  │
│                    │ Alloy       │                                  │
│                    │ :12345      │                                  │
│                    └──────┬──────┘                                 │
│                           │  remote_write / store                  │
│                    ┌──────▼──────┐    ┌──────────────┐            │
│                    │  Prometheus │    │   Loki        │            │
│                    │  (via Alloy)│    │  (log store)  │            │
│                    └──────┬──────┘    └──────┬───────┘             │
│                           │                  │                     │
│                    ┌──────▼──────────────────▼────┐               │
│                    │       Grafana :3000           │               │
│                    └──────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

Grafana Alloy acts as the central agent: it scrapes all Prometheus exporters, processes the metrics, and stores them locally (or forwards to a remote TSDB). Grafana then queries the data and renders dashboards.

---

## What's Already Installed

| Component | Version | Status |
|---|---|---|
| Ubuntu | 24.04.3 LTS | Running |
| Grafana | 12.4.0 | Running on `:3000` |
| Grafana Alloy | v1.13.2 | Running, **config needed** |
| Apache2 | 2.4.58 | Running |
| PostgreSQL | 16.11 | Running on `:5432` |
| Redis | 7.0.15 | Running on `127.0.0.1:6379` |
| Ollama | 0.11.4 | Running on `:11434` |
| Docker | 29.2.1 | Running |

Alloy's config file lives at `/etc/alloy/config.alloy`. It's currently empty — all the snippets in this guide will be combined into that file at the end.

---

## 1. Node Exporter — Host Metrics

**What it monitors:** CPU usage per core, RAM, swap, disk I/O, filesystem usage, network throughput, system load, running processes, systemd service states.

### Install

```bash
# Download the latest Node Exporter
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.9.1.linux-amd64.tar.gz
tar xvf node_exporter-1.9.1.linux-amd64.tar.gz
sudo mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter
```

### Create a systemd service

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.processes \
  --collector.thermal_zone

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

### Verify

```bash
curl http://localhost:9100/metrics | head -20
```

### Grafana Dashboard

Import dashboard **ID 1860** — "Node Exporter Full". This is the gold-standard community dashboard with 200+ panels covering everything Node Exporter exposes.

---

## 2. cAdvisor — Docker Container Metrics

**What it monitors:** Per-container CPU, RAM, network I/O, block I/O, and OOM events for every Docker container on the host — including Ghost, MediaCMS, Portainer, and any others you spin up.

### Install as a Docker container

```bash
docker run -d \
  --name cadvisor \
  --restart unless-stopped \
  --privileged \
  --volume /:/rootfs:ro \
  --volume /var/run:/var/run:ro \
  --volume /sys:/sys:ro \
  --volume /var/lib/docker/:/var/lib/docker:ro \
  --volume /dev/disk/:/dev/disk:ro \
  --publish 9093:8080 \
  gcr.io/cadvisor/cadvisor:latest
```

> **Port note:** cAdvisor's default internal port is `8080`. We map it to `9093` on the host to avoid conflicts with other services.

### Verify

```bash
curl http://localhost:9093/metrics | grep container_cpu | head -5
```

### Grafana Dashboard

Import dashboard **ID 893** — "cAdvisor Exporter". Shows per-container resource usage with time series and tables.

---

## 3. postgres_exporter — PostgreSQL Metrics

**What it monitors:** Active connections, query throughput, cache hit ratios, lock waits, table sizes, replication lag, transaction rates, and dead tuple counts for your host PostgreSQL 16 instance.

### Install

```bash
wget https://github.com/prometheus-community/postgres_exporter/releases/latest/download/postgres_exporter-0.17.1.linux-amd64.tar.gz
tar xvf postgres_exporter-0.17.1.linux-amd64.tar.gz
sudo mv postgres_exporter-0.17.1.linux-amd64/postgres_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false postgres_exporter
```

### Create a monitoring user in PostgreSQL

```bash
sudo -u postgres psql
```

```sql
CREATE USER monitoring WITH PASSWORD 'choose_a_strong_password';
GRANT pg_monitor TO monitoring;
\q
```

### Create the environment file

```bash
sudo nano /etc/postgres_exporter.env
```

```env
DATA_SOURCE_NAME=postgresql://monitoring:choose_a_strong_password@localhost:5432/postgres?sslmode=disable
```

```bash
sudo chmod 600 /etc/postgres_exporter.env
```

### Create a systemd service

```bash
sudo nano /etc/systemd/system/postgres_exporter.service
```

```ini
[Unit]
Description=PostgreSQL Exporter
Wants=network-online.target
After=network-online.target postgresql@16-main.service

[Service]
User=postgres_exporter
Group=postgres_exporter
EnvironmentFile=/etc/postgres_exporter.env
Type=simple
ExecStart=/usr/local/bin/postgres_exporter

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now postgres_exporter
```

### Verify

```bash
curl http://localhost:9187/metrics | grep pg_up
# Should return: pg_up 1
```

### Grafana Dashboard

Import dashboard **ID 9628** — "PostgreSQL Database". Shows connections, query performance, and cache stats.

---

## 4. redis_exporter — Redis Metrics

**What it monitors:** Memory usage, cache hit/miss ratio, connected clients, commands per second, keyspace stats, and eviction counts for your host Redis instance.

### Install

```bash
wget https://github.com/oliver006/redis_exporter/releases/latest/download/redis_exporter-v1.67.0.linux-amd64.tar.gz
tar xvf redis_exporter-v1.67.0.linux-amd64.tar.gz
sudo mv redis_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false redis_exporter
```

### Create a systemd service

```bash
sudo nano /etc/systemd/system/redis_exporter.service
```

```ini
[Unit]
Description=Redis Exporter
Wants=network-online.target
After=network-online.target redis-server.service

[Service]
User=redis_exporter
Group=redis_exporter
Type=simple
ExecStart=/usr/local/bin/redis_exporter \
  --redis.addr=redis://127.0.0.1:6379

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now redis_exporter
```

### Verify

```bash
curl http://localhost:9121/metrics | grep redis_up
# Should return: redis_up 1
```

### Grafana Dashboard

Import dashboard **ID 763** — "Redis Dashboard for Prometheus Redis Exporter".

---

## 5. apache_exporter — Apache2 Metrics

**What it monitors:** Total requests, bytes transferred, requests per second, workers (busy/idle), scoreboard states, and virtual host traffic.

### Enable Apache mod_status first

```bash
sudo a2enmod status
sudo nano /etc/apache2/mods-enabled/status.conf
```

Add or confirm this block is present:

```apache
<Location "/server-status">
    SetHandler server-status
    Require local
</Location>
ExtendedStatus On
```

```bash
sudo systemctl reload apache2
curl http://localhost/server-status?auto
# Should return Apache status output
```

### Install apache_exporter

```bash
wget https://github.com/Lusitaniae/apache_exporter/releases/latest/download/apache_exporter-1.0.9.linux-amd64.tar.gz
tar xvf apache_exporter-1.0.9.linux-amd64.tar.gz
sudo mv apache_exporter-1.0.9.linux-amd64/apache_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false apache_exporter
```

### Create a systemd service

```bash
sudo nano /etc/systemd/system/apache_exporter.service
```

```ini
[Unit]
Description=Apache Exporter
Wants=network-online.target
After=network-online.target apache2.service

[Service]
User=apache_exporter
Group=apache_exporter
Type=simple
ExecStart=/usr/local/bin/apache_exporter \
  --scrape_uri=http://localhost/server-status?auto

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now apache_exporter
```

### Verify

```bash
curl http://localhost:9117/metrics | grep apache_up
# Should return: apache_up 1
```

### Grafana Dashboard

Import dashboard **ID 3894** — "Apache Overview".

---

## 6. mysqld_exporter — MySQL Metrics (Ghost Blog)

**What it monitors:** Query throughput, slow queries, connection pool usage, InnoDB buffer pool, table locks, and replication status for the Ghost blog's MySQL container.

### Create a monitoring user in Ghost's MySQL container

```bash
docker exec -it ghost-blog-db-1 mysql -u root -p
```

```sql
CREATE USER 'monitoring'@'%' IDENTIFIED BY 'choose_a_strong_password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'monitoring'@'%';
FLUSH PRIVILEGES;
EXIT;
```

### Install mysqld_exporter on the host

```bash
wget https://github.com/prometheus/mysqld_exporter/releases/latest/download/mysqld_exporter-0.17.2.linux-amd64.tar.gz
tar xvf mysqld_exporter-0.17.2.linux-amd64.tar.gz
sudo mv mysqld_exporter-0.17.2.linux-amd64/mysqld_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false mysqld_exporter
```

### Create credentials file

```bash
sudo nano /etc/mysqld_exporter.cnf
```

```ini
[client]
user=monitoring
password=choose_a_strong_password
host=127.0.0.1
port=3306
```

```bash
sudo chmod 600 /etc/mysqld_exporter.cnf
```

> **Port note:** Ghost's MySQL container maps to host port `3306` is NOT exposed by default. Check with `docker inspect ghost-blog-db-1 | grep HostPort`. You may need to expose the port in the Ghost `docker-compose.yml` under `ports:` for the `db` service: `"127.0.0.1:3306:3306"`, then restart the stack.

### Create a systemd service

```bash
sudo nano /etc/systemd/system/mysqld_exporter.service
```

```ini
[Unit]
Description=MySQL Exporter
Wants=network-online.target
After=network-online.target docker.service

[Service]
User=mysqld_exporter
Group=mysqld_exporter
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/etc/mysqld_exporter.cnf

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysqld_exporter
```

### Verify

```bash
curl http://localhost:9104/metrics | grep mysql_up
# Should return: mysql_up 1
```

### Grafana Dashboard

Import dashboard **ID 7362** — "MySQL Overview".

---

## 7. Blackbox Exporter — HTTP Uptime Probes

**What it monitors:** HTTP response time, status codes, SSL certificate expiry, and availability for any URL. Perfect for monitoring Ghost, MediaCMS, your portfolio, and any other web service as an end-to-end health check.

### Install

```bash
wget https://github.com/prometheus/blackbox_exporter/releases/latest/download/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar xvf blackbox_exporter-0.25.0.linux-amd64.tar.gz
sudo mv blackbox_exporter-0.25.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false blackbox_exporter
sudo mkdir /etc/blackbox_exporter
```

### Create the config

```bash
sudo nano /etc/blackbox_exporter/config.yml
```

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 301, 302]
      follow_redirects: true
      preferred_ip_protocol: ip4

  http_post_2xx:
    prober: http
    http:
      method: POST

  tcp_connect:
    prober: tcp
    timeout: 5s
```

### Create a systemd service

```bash
sudo nano /etc/systemd/system/blackbox_exporter.service
```

```ini
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/config.yml

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now blackbox_exporter
```

### Verify

```bash
curl "http://localhost:9115/probe?target=https://example.com&module=http_2xx" | grep probe_success
# Should return: probe_success 1
```

### Grafana Dashboard

Import dashboard **ID 7587** — "Prometheus Blackbox Exporter". Lists all your probed targets with status, latency, and SSL expiry.

---

## 8. Ollama Metrics

**What it monitors:** Model load times, inference request count, token generation throughput, and GPU/CPU utilization per model.

Ollama 0.11.4 exposes Prometheus metrics but requires an environment variable to enable the endpoint.

### Enable the metrics endpoint

Edit the Ollama systemd service override:

```bash
sudo systemctl edit ollama.service
```

Add the following:

```ini
[Service]
Environment="OLLAMA_PROMETHEUS_ENABLED=true"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### Verify

```bash
curl http://localhost:11434/metrics | head -20
```

> **Note:** If this endpoint still returns 404, your Ollama build may not yet support Prometheus metrics natively. An alternative is to use a community exporter such as `ollama-exporter` which wraps the Ollama REST API.

---

## 9. Configuring Grafana Alloy

Now that all exporters are running, configure Alloy to scrape them all and feed the data into Grafana. Alloy uses its own HCL-like configuration language called River.

```bash
sudo nano /etc/alloy/config.alloy
```

```hcl
// ─────────────────────────────────────────────────────────────────
// Prometheus scrape configuration
// ─────────────────────────────────────────────────────────────────

// Node Exporter — host metrics
prometheus.scrape "node_exporter" {
  targets = [{"__address__" = "localhost:9100"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
  job_name = "node_exporter"
}

// cAdvisor — Docker container metrics
prometheus.scrape "cadvisor" {
  targets = [{"__address__" = "localhost:9093"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
  job_name = "cadvisor"
}

// PostgreSQL
prometheus.scrape "postgres" {
  targets = [{"__address__" = "localhost:9187"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "30s"
  job_name = "postgresql"
}

// Redis
prometheus.scrape "redis" {
  targets = [{"__address__" = "localhost:9121"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
  job_name = "redis"
}

// Apache
prometheus.scrape "apache" {
  targets = [{"__address__" = "localhost:9117"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
  job_name = "apache"
}

// MySQL (Ghost blog)
prometheus.scrape "mysql" {
  targets = [{"__address__" = "localhost:9104"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "30s"
  job_name = "mysql"
}

// Ollama
prometheus.scrape "ollama" {
  targets = [{"__address__" = "localhost:11434"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
  job_name = "ollama"
}

// ─────────────────────────────────────────────────────────────────
// Blackbox Exporter — HTTP probes
// Add or remove targets to match your services
// ─────────────────────────────────────────────────────────────────

prometheus.scrape "blackbox_http" {
  targets = [
    {
      "__address__" = "localhost:9115",
      "__param_target" = "http://localhost:2368",
      "__param_module" = "http_2xx",
      "instance" = "ghost-blog",
    },
    {
      "__address__" = "localhost:9115",
      "__param_target" = "http://localhost:8555",
      "__param_module" = "http_2xx",
      "instance" = "termix",
    },
    {
      "__address__" = "localhost:9115",
      "__param_target" = "http://localhost:4004",
      "__param_module" = "http_2xx",
      "instance" = "it-tools",
    },
    {
      "__address__" = "localhost:9115",
      "__param_target" = "https://localhost:9443",
      "__param_module" = "http_2xx",
      "instance" = "portainer",
    },
  ]
  forward_to = [prometheus.remote_write.local.receiver]
  metrics_path = "/probe"
  scrape_interval = "30s"
  job_name = "blackbox"
}

// ─────────────────────────────────────────────────────────────────
// Local Prometheus-compatible storage (Alloy built-in)
// Grafana reads from this endpoint
// ─────────────────────────────────────────────────────────────────

prometheus.remote_write "local" {
  endpoint {
    url = "http://localhost:9090/api/v1/write"
  }
}
```

> **Important:** The `prometheus.remote_write` above assumes you have a Prometheus instance running on port 9090. If you don't, Alloy can store metrics locally. See the note below.

### Option A: Run Prometheus alongside Alloy (recommended)

```bash
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-3.2.1.linux-amd64.tar.gz
tar xvf prometheus-3.2.1.linux-amd64.tar.gz
sudo mv prometheus-3.2.1.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo mkdir /etc/prometheus /var/lib/prometheus
sudo useradd --no-create-home --shell /bin/false prometheus
```

Create `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

storage:
  tsdb:
    retention.time: 30d
```

Create `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.enable-remote-write-receiver \
  --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
```

```bash
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

### Option B: Use Alloy's built-in metrics storage (simpler, less flexible)

Replace the `prometheus.remote_write` block in `config.alloy` with:

```hcl
// Use Alloy's own Prometheus-compatible storage
prometheus.remote_write "local" {
  endpoint {
    url = "http://localhost:12345/api/v1/push"
  }
}
```

Then add Grafana's data source pointing at `http://localhost:12345` and selecting type "Prometheus".

### Reload Alloy after editing the config

```bash
sudo systemctl restart alloy

# Check for config errors
sudo journalctl -u alloy -f --no-pager | head -30
```

---

## 10. Importing Dashboards into Grafana

### Add Prometheus as a data source

1. Open Grafana at `http://localhost:3000`
2. Go to **Connections → Data Sources → Add data source**
3. Select **Prometheus**
4. Set URL to `http://localhost:9090`
5. Click **Save & Test** — should show "Data source is working"

### Import community dashboards

1. Go to **Dashboards → New → Import**
2. Enter the dashboard ID from the table below
3. Select your Prometheus data source
4. Click **Import**

| Dashboard | ID | Covers |
|---|---|---|
| Node Exporter Full | 1860 | CPU, RAM, disk, network, systemd |
| cAdvisor (Docker) | 893 | Per-container resource usage |
| PostgreSQL | 9628 | Queries, connections, cache |
| Redis | 763 | Memory, hit rate, throughput |
| Apache | 3894 | Requests, workers, bytes |
| MySQL Overview | 7362 | Ghost's MySQL instance |
| Blackbox Exporter | 7587 | HTTP uptime and latency |

---

## 11. Optional: Loki for Log Aggregation

If you want to monitor **logs** (SSH attempts, Apache access logs, application errors) alongside metrics in the same Grafana instance, add Loki.

### Run Loki as a Docker container

```bash
mkdir -p /opt/loki/data

docker run -d \
  --name loki \
  --restart unless-stopped \
  --volume /opt/loki/data:/loki \
  --publish 3100:3100 \
  grafana/loki:latest \
  -config.file=/etc/loki/local-config.yaml
```

### Configure Alloy to collect logs

Add this to `/etc/alloy/config.alloy`:

```hcl
// ─────────────────────────────────────────────────────────────────
// Loki log collection
// ─────────────────────────────────────────────────────────────────

loki.source.journal "system" {
  forward_to = [loki.write.local.receiver]
  labels = {component = "systemd-journal"}
}

local.file_match "apache_logs" {
  path_targets = [
    {
      "__path__" = "/var/log/apache2/*.log",
      "job"      = "apache",
    },
  ]
}

loki.source.file "apache" {
  targets    = local.file_match.apache_logs.targets
  forward_to = [loki.write.local.receiver]
}

loki.source.file "auth" {
  targets = [{
    "__path__" = "/var/log/auth.log",
    "job"      = "auth",
  }]
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
}
```

### Add Loki as a data source in Grafana

1. **Connections → Data Sources → Add data source**
2. Select **Loki**
3. URL: `http://localhost:3100`
4. Save & Test

You can now use Grafana Explore to query logs with LogQL, or create a unified dashboard showing metrics and logs side by side.

---

## Quick Reference: Exporter Ports

| Exporter | Port | Endpoint |
|---|---|---|
| Node Exporter | 9100 | `/metrics` |
| cAdvisor | 9093 | `/metrics` |
| postgres_exporter | 9187 | `/metrics` |
| redis_exporter | 9121 | `/metrics` |
| apache_exporter | 9117 | `/metrics` |
| mysqld_exporter | 9104 | `/metrics` |
| blackbox_exporter | 9115 | `/probe` |
| Ollama (native) | 11434 | `/metrics` |
| Prometheus | 9090 | `/metrics` |
| Grafana Alloy | 12345 | `/metrics` |
| Loki | 3100 | `/metrics` |

---

## Troubleshooting Tips

**Exporter won't start:**
```bash
sudo journalctl -u <service-name> -n 50 --no-pager
```

**Alloy config syntax error:**
```bash
/usr/bin/alloy fmt /etc/alloy/config.alloy  # auto-formats and validates
```

**Metrics not appearing in Grafana:**
- Confirm the exporter is returning data: `curl http://localhost:<port>/metrics`
- Check Alloy logs: `sudo journalctl -u alloy -f`
- Verify the Prometheus data source in Grafana shows "Data source is working"
- In Grafana Explore, query `up` — each scrape target should appear with value `1`

**Docker container exporter access:**
- Exporters running on the host can reach Docker container ports that are published to `localhost` or `0.0.0.0`
- Containers in bridge networks are NOT directly reachable from host by container IP without extra routing — always publish ports or use `--network=host`

**PostgreSQL exporter can't connect:**
```bash
sudo -u postgres psql -c "\du monitoring"  # verify user exists
```

---

*All version numbers in download URLs were current at time of writing. Always check the respective GitHub releases page for the latest stable version before downloading.*
