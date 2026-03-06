# Advanced & Professional Grafana Observability

A deep-dive guide to production-grade observability patterns for your self-hosted Ubuntu 24.04 server. This goes beyond metric collection into distributed tracing, continuous profiling, security monitoring, SLOs, and infrastructure-as-code — the skills that appear in senior SRE and DevOps roles.

This guide assumes you have completed Part 1 (Node Exporter, cAdvisor, exporters, Alloy, Prometheus, and Grafana running).

---

## Table of Contents

1. [The Four Pillars — Completing the Set](#1-the-four-pillars--completing-the-set)
2. [Distributed Tracing with Grafana Tempo](#2-distributed-tracing-with-grafana-tempo)
3. [Auto-Instrumentation with Beyla (eBPF)](#3-auto-instrumentation-with-beyla-ebpf)
4. [Continuous Profiling with Grafana Pyroscope](#4-continuous-profiling-with-grafana-pyroscope)
5. [Observability Methodologies: RED, USE, and SLOs](#5-observability-methodologies-red-use-and-slos)
6. [PostgreSQL Deep Dive](#6-postgresql-deep-dive)
7. [LLM Observability — Custom Ollama Exporter](#7-llm-observability--custom-ollama-exporter)
8. [Security Observability](#8-security-observability)
9. [Alertmanager — Production-Grade Alerting](#9-alertmanager--production-grade-alerting)
10. [Dashboards as Code](#10-dashboards-as-code)
11. [Cardinality Management](#11-cardinality-management)
12. [Long-Term Metric Storage with Grafana Mimir](#12-long-term-metric-storage-with-grafana-mimir)

---

## 1. The Four Pillars — Completing the Set

The industry has converged on four signals that together give you full system understanding. After Part 1, you have Metrics and (optionally) Logs. This guide adds the remaining two.

```
┌─────────────────────────────────────────────────────────────────┐
│  Pillar         Tool              Answers the question           │
│  ─────────────────────────────────────────────────────────────  │
│  Metrics    →   Prometheus        Is something wrong?           │
│  Logs       →   Loki              What happened?                │
│  Traces     →   Grafana Tempo     Where is the slowness?        │
│  Profiles   →   Grafana Pyroscope Why is it slow? (code level)  │
└─────────────────────────────────────────────────────────────────┘
```

The real power comes from **correlation**: clicking a spike on a metrics graph, jumping to the logs for that time window, finding the trace ID in the log line, opening the flame graph for that trace. Grafana calls this "Explore" mode with linked data sources.

---

## 2. Distributed Tracing with Grafana Tempo

### What it is

Every request that enters your system (e.g., HTTP to Apache → Ghost → MySQL) generates a **trace** — a tree of spans showing each hop with its duration. Tempo stores these traces; Grafana lets you search and visualise them.

### Why it matters professionally

Distributed tracing is the primary tool used to debug latency issues in microservice architectures. Understanding it is essentially mandatory for senior backend and SRE roles.

### Install Tempo

```bash
mkdir -p /opt/tempo/data

cat > /opt/tempo/tempo.yaml << 'EOF'
stream_over_http_enabled: true

server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/blocks
    wal:
      path: /tmp/tempo/wal

compactor:
  compaction:
    block_retention: 336h   # 14 days

metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /tmp/tempo/generator/wal
    remote_write:
      - url: http://localhost:9090/api/v1/write
        send_exemplars: true

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
EOF

docker run -d \
  --name tempo \
  --restart unless-stopped \
  --network host \
  --volume /opt/tempo/tempo.yaml:/etc/tempo/tempo.yaml \
  --volume /opt/tempo/data:/tmp/tempo \
  grafana/tempo:latest \
  -config.file=/etc/tempo/tempo.yaml
```

### Configure Alloy to receive and forward traces

Add to `/etc/alloy/config.alloy`:

```hcl
// ─────────────────────────────────────────────────────────────────
// OpenTelemetry trace receiver — accepts OTLP from instrumented apps
// ─────────────────────────────────────────────────────────────────

otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "localhost:4317"
    tls {
      insecure = true
    }
  }
}
```

### Add Tempo as a Grafana data source

1. **Connections → Data Sources → Add data source → Tempo**
2. URL: `http://localhost:3200`
3. Under **Trace to logs**: link to your Loki data source, tag `traceID`
4. Under **Trace to metrics**: link to Prometheus
5. **Save & Test**

### Link Trace IDs into logs (Loki pipeline)

This is what enables "click trace → jump to logs" in Grafana. When your apps emit a `traceID` field in their logs, add this to Alloy's log pipeline:

```hcl
loki.process "apache" {
  forward_to = [loki.write.local.receiver]

  stage.regex {
    expression = `(?P<traceID>[a-f0-9]{32})`
  }
  stage.labels {
    values = {
      traceID = "",
    }
  }
}
```

---

## 3. Auto-Instrumentation with Beyla (eBPF)

### What it is

Beyla is Grafana's eBPF-based auto-instrumentation agent. It attaches to a running process at the kernel level and extracts traces and RED metrics **without any code changes**. This means you can get traces from Ghost (Node.js), Apache, or any other service without touching their source code.

eBPF (Extended Berkeley Packet Filter) is a Linux kernel technology that allows safe programs to run in kernel space. It is one of the most important technologies in modern infrastructure tooling — Kubernetes networking (Cilium), observability (Pixie, Beyla), and security (Falco) all use it.

### Kernel requirement

```bash
uname -r
# You have 6.8.0-100-generic — Beyla requires 5.8+, you're fine
```

### Install Beyla as a systemd service

```bash
wget https://github.com/grafana/beyla/releases/latest/download/beyla-linux-amd64
chmod +x beyla-linux-amd64
sudo mv beyla-linux-amd64 /usr/local/bin/beyla
```

### Configure Beyla for Ghost (Node.js on port 2368)

```bash
sudo mkdir /etc/beyla
sudo nano /etc/beyla/ghost.yaml
```

```yaml
open_port: 2368

otel_traces_export:
  endpoint: http://localhost:4318

otel_metrics_export:
  endpoint: http://localhost:4318

attributes:
  kubernetes:
    enable: false

log:
  level: info
```

```bash
sudo nano /etc/systemd/system/beyla-ghost.service
```

```ini
[Unit]
Description=Beyla eBPF auto-instrumentation for Ghost
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/beyla --config=/etc/beyla/ghost.yaml
Environment="BEYLA_SERVICE_NAME=ghost-blog"
Environment="BEYLA_OTEL_EXPORTER_ENDPOINT=http://localhost:4318"
AmbientCapabilities=CAP_SYS_ADMIN CAP_SYS_PTRACE CAP_NET_ADMIN CAP_BPF
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_SYS_PTRACE CAP_NET_ADMIN CAP_BPF

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now beyla-ghost
```

Beyla will now emit traces and RED metrics for every HTTP request Ghost handles — with zero changes to Ghost itself. You can create a second Beyla service for Apache by pointing it at port `80`.

### What you get from Beyla

- HTTP request traces per route (`GET /`, `GET /api/posts`, etc.)
- Duration histogram (p50, p95, p99)
- Error rate by status code
- Automatic span creation for database calls (if detectable)
- Service graph showing Ghost → MySQL request flow

---

## 4. Continuous Profiling with Grafana Pyroscope

### What it is

A **flame graph** shows you exactly which functions in your code consumed CPU during a time period. Pyroscope does this continuously in production, sampling running processes every few milliseconds and storing the profiles over time.

This is the answer to "metrics say CPU is high but why?" — profiling tells you the specific function call chain that is hot.

### Install Pyroscope

```bash
mkdir -p /opt/pyroscope/data

docker run -d \
  --name pyroscope \
  --restart unless-stopped \
  --network host \
  --volume /opt/pyroscope/data:/data \
  grafana/pyroscope:latest \
  --config.file="" \
  --storage.path=/data \
  --server.http-listen-port=4040
```

### Configure Alloy to scrape profiles via eBPF (whole-system profiling)

This profiles every process on the host with zero per-process configuration — one of the most powerful single additions you can make to an observability stack.

Add to `/etc/alloy/config.alloy`:

```hcl
// ─────────────────────────────────────────────────────────────────
// Pyroscope — continuous profiling via eBPF (profiles all processes)
// ─────────────────────────────────────────────────────────────────

pyroscope.ebpf "system" {
  forward_to = [pyroscope.write.local.receiver]
  targets_only = false
}

pyroscope.write "local" {
  endpoint {
    url = "http://localhost:4040"
  }
}
```

### Profile Ollama specifically (pprof)

Ollama exposes a Go pprof endpoint. Add targeted profiling:

```hcl
pyroscope.scrape "ollama" {
  targets = [{"__address__" = "localhost:11434", "service_name" = "ollama"}]
  forward_to = [pyroscope.write.local.receiver]
  profiling_config {
    profile.process_cpu {
      enabled = true
    }
    profile.memory {
      enabled = true
    }
  }
}
```

### Add Pyroscope as a Grafana data source

1. **Connections → Data Sources → Pyroscope**
2. URL: `http://localhost:4040`
3. Save & Test

### Linking profiles to traces (exemplars)

In Tempo's config, you can attach a `profile.id` label to traces. When viewing a slow trace, you can click "View profile" to jump directly to the flame graph for that exact request period — this is called **trace-to-profile correlation** and is genuinely rare expertise.

---

## 5. Observability Methodologies: RED, USE, and SLOs

### The RED Method — apply to every service

RED stands for Rate, Errors, Duration. Build one RED dashboard per service you run. This is the standard used at Google, Netflix, Cloudflare, and most large engineering orgs.

**Dashboard panel structure (repeat for each service):**

| Panel | PromQL | Description |
|---|---|---|
| Request rate | `rate(http_requests_total[5m])` | Requests per second |
| Error rate % | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100` | % of requests failing |
| p50 latency | `histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))` | Median response time |
| p95 latency | `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` | 95th percentile |
| p99 latency | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` | Worst 1% of requests |

When using Beyla, the metric names are `http.server.request.duration` (OTLP naming).

### The USE Method — apply to every resource

USE stands for Utilization, Saturation, Errors. It answers "is this resource a bottleneck?"

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))` | `node_load1 / count(node_cpu_seconds_total{mode="idle"})` | `rate(node_cpu_seconds_total{mode="irq"}[5m])` |
| Memory | `1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes` | `rate(node_vmstat_pgmajfault[5m])` (major page faults) | N/A |
| Disk | `rate(node_disk_io_time_seconds_total[5m])` | `rate(node_disk_io_time_weighted_seconds_total[5m])` | `rate(node_disk_read_errors_total[5m])` |
| Network | `rate(node_network_transmit_bytes_total[5m]) / <link_speed>` | `rate(node_network_transmit_drop_total[5m])` | `rate(node_network_transmit_errs_total[5m])` |

Your server is a 4-core i5-4200U with 5.6GB RAM — saturation metrics will tell you when you're approaching limits before things actually break.

### SLOs and Error Budgets

An **SLO (Service Level Objective)** is a target reliability level for a service. An **error budget** is how much unreliability you are allowed before breaching the SLO.

Example SLO: "Ghost blog responds with HTTP 200 in under 800ms for 99.5% of requests in a 30-day window."

If you have 30 days × 24h × 60m = 43,200 minutes, a 99.5% SLO means you can afford **216 minutes of budget per month** for slowness or errors.

#### Enable the Grafana SLO plugin

```bash
sudo grafana-cli plugins install grafana-slo-panel
sudo systemctl restart grafana-server
```

#### Define an SLO in Grafana

1. Go to **Alerts & IRM → SLOs → Create SLO**
2. **Service:** Ghost Blog
3. **Metric:** `probe_success` from Blackbox Exporter
4. **Target:** 99.5%
5. **Window:** 30 days

Grafana will automatically generate:
- The error budget burn rate alert (fires when you're burning budget too fast)
- A dashboard showing remaining budget, burn rate, and projected breach

#### Burn rate alerting

The key insight: a 1x burn rate means you'll exhaust your budget exactly at the end of the window. A **14.4x burn rate** means you'll exhaust it in 2 hours. Grafana SLOs alert at high burn rates (critical) and low burn rates over longer windows (warning) — this is called the **multi-window, multi-burn-rate** alerting strategy from Google's SRE workbook.

---

## 6. PostgreSQL Deep Dive

### Enable pg_stat_statements

This extension tracks execution statistics for every SQL query. It is the most important tool for PostgreSQL performance analysis.

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Find and set:

```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000
track_activity_query_size = 2048
log_min_duration_statement = 1000   # log queries slower than 1 second
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

```bash
sudo systemctl restart postgresql@16-main
sudo -u postgres psql -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
```

### Slow query dashboard via Loki

Configure Alloy to ship PostgreSQL logs to Loki, then build a Loki dashboard panel using LogQL to surface slow queries:

```hcl
local.file_match "postgres_logs" {
  path_targets = [{
    "__path__" = "/var/log/postgresql/postgresql-16-main.log",
    "job"      = "postgresql",
    "service"  = "postgres",
  }]
}

loki.source.file "postgres" {
  targets    = local.file_match.postgres_logs.targets
  forward_to = [loki.process.postgres.receiver]
}

loki.process "postgres" {
  forward_to = [loki.write.local.receiver]

  // Parse the log_line_prefix format
  stage.regex {
    expression = `user=(?P<user>\S+),db=(?P<db>\S+),app=(?P<app>\S+)`
  }
  stage.labels {
    values = {
      user = "",
      db   = "",
    }
  }

  // Extract slow query duration
  stage.regex {
    expression = `duration: (?P<duration_ms>[\d.]+) ms`
  }
  stage.metrics {
    metric.histogram {
      name        = "pg_slow_query_duration_ms"
      description = "PostgreSQL slow query duration"
      source      = "duration_ms"
      buckets     = [100, 250, 500, 1000, 2500, 5000, 10000]
    }
  }
}
```

**LogQL query to find all slow queries:**

```logql
{job="postgresql"} |= "duration:" | regexp `duration: (?P<duration_ms>[\d.]+) ms` | duration_ms > 500
```

### Useful pg_stat_statements queries

Connect as the monitoring user and run these to build custom panels via Grafana's PostgreSQL data source:

**Top 10 queries by total time:**
```sql
SELECT
  round(total_exec_time::numeric, 2) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct,
  left(query, 80) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Table bloat (rows needing VACUUM):**
```sql
SELECT
  schemaname,
  tablename,
  n_dead_tup AS dead_tuples,
  n_live_tup AS live_tuples,
  round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**Currently waiting queries (lock contention):**
```sql
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query,
  state,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - pg_stat_activity.query_start > interval '5 seconds'
ORDER BY duration DESC;
```

Add the host PostgreSQL as a **data source** in Grafana (type: PostgreSQL), then build panels using these queries directly — this lets you create database health dashboards using SQL rather than PromQL.

---

## 7. LLM Observability — Custom Ollama Exporter

### Why this matters

LLM observability is an emerging field with no dominant standard yet. Writing your own exporter right now puts you ahead of the curve. Your server runs phi3, qwen3, deepseek-r1:8b, and nomic-embed-text — you have a real multi-model setup to monitor.

### What to track

| Metric | Description | Insight |
|---|---|---|
| `ollama_model_loaded` (gauge) | Which models are currently in memory | Resource planning |
| `ollama_request_duration_seconds` (histogram) | End-to-end request time per model | Latency SLO |
| `ollama_tokens_per_second` (gauge) | Token generation rate per model | Throughput |
| `ollama_prompt_tokens_total` (counter) | Tokens processed in prompts | Cost proxy |
| `ollama_completion_tokens_total` (counter) | Tokens generated | Usage tracking |
| `ollama_queue_depth` (gauge) | Requests waiting (if API is busy) | Saturation |
| `ollama_model_load_duration_seconds` (histogram) | Time to load a model from disk | Cold start latency |

### Write the exporter in Python

```bash
pip3 install prometheus_client requests
sudo nano /usr/local/bin/ollama_exporter.py
```

```python
#!/usr/bin/env python3
"""
Prometheus exporter for Ollama.
Exposes metrics from the Ollama REST API.
"""

import time
import threading
import requests
from prometheus_client import (
    start_http_server,
    Gauge,
    Histogram,
    Counter,
    REGISTRY,
)

OLLAMA_BASE = "http://localhost:11434"
SCRAPE_INTERVAL = 15

# ── Gauges ──────────────────────────────────────────────────────────────────

ollama_up = Gauge(
    "ollama_up",
    "Whether the Ollama API is reachable",
)

model_loaded = Gauge(
    "ollama_model_loaded",
    "1 if the model is currently loaded in memory",
    ["model", "parameter_size"],
)

model_size_bytes = Gauge(
    "ollama_model_size_bytes",
    "Size of the model on disk in bytes",
    ["model"],
)

tokens_per_second = Gauge(
    "ollama_tokens_per_second",
    "Token generation speed from the last completed request",
    ["model"],
)

# ── Histograms ───────────────────────────────────────────────────────────────

request_duration = Histogram(
    "ollama_request_duration_seconds",
    "End-to-end generation request duration",
    ["model", "status"],
    buckets=[0.5, 1, 2, 5, 10, 20, 30, 60, 120],
)

load_duration = Histogram(
    "ollama_model_load_duration_seconds",
    "Time taken to load a model into memory",
    ["model"],
    buckets=[0.1, 0.5, 1, 2, 5, 10, 30],
)

# ── Counters ─────────────────────────────────────────────────────────────────

prompt_tokens = Counter(
    "ollama_prompt_tokens_total",
    "Total prompt tokens processed",
    ["model"],
)

completion_tokens = Counter(
    "ollama_completion_tokens_total",
    "Total completion tokens generated",
    ["model"],
)

request_total = Counter(
    "ollama_requests_total",
    "Total requests processed",
    ["model", "status"],
)


def collect_loaded_models():
    """Poll /api/ps for currently loaded models."""
    try:
        r = requests.get(f"{OLLAMA_BASE}/api/ps", timeout=5)
        r.raise_for_status()
        data = r.json()
        ollama_up.set(1)

        # Reset all model_loaded gauges to 0 before setting active ones
        # This handles models being unloaded between scrapes
        for model_info in data.get("models", []):
            name = model_info.get("name", "unknown")
            param_size = model_info.get("details", {}).get("parameter_size", "unknown")
            model_loaded.labels(model=name, parameter_size=param_size).set(1)

    except Exception as e:
        ollama_up.set(0)
        print(f"[error] Could not reach Ollama: {e}")


def collect_model_library():
    """Poll /api/tags for all local models and their disk sizes."""
    try:
        r = requests.get(f"{OLLAMA_BASE}/api/tags", timeout=5)
        r.raise_for_status()
        data = r.json()

        for m in data.get("models", []):
            name = m.get("name", "unknown")
            size = m.get("size", 0)
            model_size_bytes.labels(model=name).set(size)

    except Exception as e:
        print(f"[error] Could not fetch model library: {e}")


def probe_model(model_name: str, prompt: str = "Reply with one word: ready"):
    """
    Send a real generation request and record timing/token metrics.
    Use a minimal prompt to keep overhead low.
    """
    payload = {
        "model": model_name,
        "prompt": prompt,
        "stream": False,
    }
    start = time.time()
    status = "success"
    try:
        r = requests.post(
            f"{OLLAMA_BASE}/api/generate",
            json=payload,
            timeout=120,
        )
        r.raise_for_status()
        data = r.json()

        duration = time.time() - start
        request_duration.labels(model=model_name, status=status).observe(duration)

        # Token metrics from the response
        p_tokens = data.get("prompt_eval_count", 0)
        c_tokens = data.get("eval_count", 0)
        eval_duration_ns = data.get("eval_duration", 1)  # nanoseconds
        load_dur_ns = data.get("load_duration", 0)

        prompt_tokens.labels(model=model_name).inc(p_tokens)
        completion_tokens.labels(model=model_name).inc(c_tokens)
        request_total.labels(model=model_name, status="success").inc()

        # tokens/sec: eval_count / eval_duration_seconds
        if eval_duration_ns > 0:
            tps = c_tokens / (eval_duration_ns / 1e9)
            tokens_per_second.labels(model=model_name).set(tps)

        # Model load duration (0 if model was already in memory)
        if load_dur_ns > 0:
            load_duration.labels(model=model_name).observe(load_dur_ns / 1e9)

    except Exception as e:
        duration = time.time() - start
        request_duration.labels(model=model_name, status="error").observe(duration)
        request_total.labels(model=model_name, status="error").inc()
        print(f"[error] Probe failed for {model_name}: {e}")


def collection_loop():
    """Main collection loop — runs every SCRAPE_INTERVAL seconds."""
    # Models to actively probe (lightweight prompt every scrape)
    # nomic-embed-text uses /api/embed, not /api/generate — excluded here
    probe_models = ["phi3:latest", "qwen3:0.6b"]

    while True:
        collect_loaded_models()
        collect_model_library()

        for model in probe_models:
            probe_model(model)

        time.sleep(SCRAPE_INTERVAL)


if __name__ == "__main__":
    print("[ollama_exporter] Starting on :9500")
    start_http_server(9500)
    t = threading.Thread(target=collection_loop, daemon=True)
    t.start()
    # Keep main thread alive
    while True:
        time.sleep(60)
```

### Create a systemd service

```bash
sudo nano /etc/systemd/system/ollama_exporter.service
```

```ini
[Unit]
Description=Ollama Prometheus Exporter
Wants=network-online.target ollama.service
After=network-online.target ollama.service

[Service]
User=nobody
Group=nogroup
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/ollama_exporter.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo chmod +x /usr/local/bin/ollama_exporter.py
sudo systemctl daemon-reload
sudo systemctl enable --now ollama_exporter
curl http://localhost:9500/metrics | grep ollama
```

### Add to Alloy

```hcl
prometheus.scrape "ollama_custom" {
  targets = [{"__address__" = "localhost:9500"}]
  forward_to = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
  job_name = "ollama_custom"
}
```

### Grafana dashboard panels for Ollama

| Panel | PromQL |
|---|---|
| Tokens/sec per model | `ollama_tokens_per_second` |
| Total tokens generated | `increase(ollama_completion_tokens_total[1h])` |
| Request latency p95 | `histogram_quantile(0.95, rate(ollama_request_duration_seconds_bucket[5m]))` |
| Error rate | `rate(ollama_requests_total{status="error"}[5m])` |
| Model disk size | `ollama_model_size_bytes` |
| Ollama availability | `ollama_up` |

---

## 8. Security Observability

### SSH brute force detection with Loki + AlertManager

Configure Alloy to ship `auth.log`:

```hcl
loki.source.file "auth" {
  targets = [{
    "__path__" = "/var/log/auth.log",
    "job"      = "auth",
    "host"     = "server",
  }]
  forward_to = [loki.process.auth.receiver]
}

loki.process "auth" {
  forward_to = [loki.write.local.receiver]

  // Extract the source IP from failed login lines
  stage.regex {
    expression = `Failed password for .* from (?P<src_ip>\d+\.\d+\.\d+\.\d+)`
  }
  stage.labels {
    values = { src_ip = "" }
  }

  // Create a metric for failed logins (queryable in Prometheus)
  stage.metrics {
    metric.counter {
      name        = "ssh_failed_logins_total"
      description = "Total SSH failed login attempts"
      source      = "src_ip"
      match_all   = true
      labels_from_source = ["src_ip"]
    }
  }
}
```

### Grafana alert rule for brute force

In Grafana → **Alerting → Alert Rules → New rule**:

```
Expression: sum by (src_ip) (rate(ssh_failed_logins_total[5m])) * 300 > 10

Meaning: more than 10 failed logins from the same IP in the last 5 minutes
```

Set a notification policy to send this to email or a webhook (e.g., a Telegram bot).

### Useful LogQL security queries for Explore

```logql
# Failed SSH logins with source IPs
{job="auth"} |= "Failed password"

# Successful logins (useful to confirm legitimacy)
{job="auth"} |= "Accepted publickey"

# New user/group creation
{job="auth"} |= "useradd" or "groupadd"

# sudo usage
{job="auth"} |= "sudo"

# Unexpected cron jobs
{job="cron"} != "CMD" |= "CRON"
```

### Falco — runtime security (kernel-level)

Falco watches Linux syscalls and fires alerts when a process does something unexpected — like a container writing to `/etc`, spawning a shell, or opening a sensitive file.

**Install:**

```bash
# Add Falco repository
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] \
  https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list

sudo apt update
sudo apt install -y falco
```

During install, choose **eBPF driver** (works without custom kernel module on kernel 6.8).

**Falco emits structured JSON logs to syslog by default.** Ship them to Loki via Alloy for searching in Grafana.

**Key default rules Falco fires on:**

- Terminal shell in a container
- Write below `/etc` in a container
- Sensitive file opened for reading (e.g., `/etc/shadow`)
- Unexpected outbound connection from a container
- Container running as root when it shouldn't

### Docker container escape detection

Add a custom Falco rule:

```bash
sudo nano /etc/falco/rules.d/custom.yaml
```

```yaml
- rule: Container attempted privileged escalation
  desc: A container tried to execute a privileged operation
  condition: >
    spawned_process and container
    and (proc.name in (nsenter, unshare)
         or proc.cmdline contains "docker.sock")
  output: >
    Privileged escalation attempt in container
    (user=%user.name cmd=%proc.cmdline container=%container.name)
  priority: CRITICAL
  tags: [container, security]
```

---

## 9. Alertmanager — Production-Grade Alerting

### Why default Grafana alerting isn't enough

Grafana has built-in alerting but it fires each alert independently. **Alertmanager** adds:
- **Routing trees** — route by severity, service, or team
- **Inhibition rules** — suppress child alerts when a parent is firing
- **Grouping** — batch 50 alerts into one notification
- **Silences** — mute during maintenance windows
- **Dead man's switch** — know when your monitoring itself is broken

### Install

```bash
wget https://github.com/prometheus/alertmanager/releases/latest/download/alertmanager-0.28.0.linux-amd64.tar.gz
tar xvf alertmanager-0.28.0.linux-amd64.tar.gz
sudo mv alertmanager-0.28.0.linux-amd64/{alertmanager,amtool} /usr/local/bin/
sudo mkdir /etc/alertmanager /var/lib/alertmanager
sudo useradd --no-create-home --shell /bin/false alertmanager
```

### Configuration

```bash
sudo nano /etc/alertmanager/alertmanager.yml
```

```yaml
global:
  # Default time to wait before resending an unresolved alert
  resolve_timeout: 5m

  # Email config (update with your SMTP details)
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@yourdomain.com'
  smtp_auth_username: 'alerts@yourdomain.com'
  smtp_auth_password: 'your_app_password'

# ── Routing tree ─────────────────────────────────────────────────────────────
# Alerts flow from the root down to the first matching route
route:
  group_by: ['alertname', 'job']
  group_wait: 30s         # wait 30s to batch alerts in the same group
  group_interval: 5m      # minimum time between sending group updates
  repeat_interval: 4h     # resend unresolved alerts after 4 hours
  receiver: 'default-email'

  routes:
    # Critical alerts — notify immediately, repeat every hour
    - match:
        severity: critical
      receiver: 'critical-email'
      repeat_interval: 1h

    # Security alerts — their own channel
    - match:
        category: security
      receiver: 'security-email'
      repeat_interval: 30m

    # Watchdog / Dead man's switch — must always be firing
    # If this alert stops, something is wrong with your monitoring
    - match:
        alertname: Watchdog
      receiver: 'null'   # silence it unless it stops firing

# ── Receivers ────────────────────────────────────────────────────────────────
receivers:
  - name: 'null'

  - name: 'default-email'
    email_configs:
      - to: 'you@yourdomain.com'
        send_resolved: true
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'

  - name: 'critical-email'
    email_configs:
      - to: 'you@yourdomain.com'
        send_resolved: true
        headers:
          Subject: '🔴 CRITICAL: {{ .GroupLabels.alertname }}'

  - name: 'security-email'
    email_configs:
      - to: 'you@yourdomain.com'
        send_resolved: false
        headers:
          Subject: '🔒 SECURITY: {{ .GroupLabels.alertname }}'

# ── Inhibition rules ─────────────────────────────────────────────────────────
# If the source alert is firing, suppress the target alerts
inhibit_rules:
  # If the whole host is unreachable, don't also fire alerts for every service on it
  - source_match:
      alertname: 'HostDown'
    target_match_re:
      alertname: '.+'
    equal: ['instance']

  # If PostgreSQL is down, suppress "high connection count" alert
  - source_match:
      alertname: 'PostgreSQLDown'
    target_match:
      alertname: 'PostgreSQLTooManyConnections'
    equal: ['instance']
```

```bash
sudo nano /etc/systemd/system/alertmanager.service
```

```ini
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=0.0.0.0:9093
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

```bash
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
```

### Define Prometheus alert rules

```bash
sudo mkdir /etc/prometheus/rules
sudo nano /etc/prometheus/rules/host.yml
```

```yaml
groups:
  - name: host
    rules:
      # ── Dead man's switch ─────────────────────────────────────────
      # This alert should ALWAYS be firing.
      # If it stops, your alerting pipeline is broken.
      - alert: Watchdog
        expr: vector(1)
        labels:
          severity: none
        annotations:
          summary: "Alerting pipeline is alive"

      # ── Host alerts ───────────────────────────────────────────────
      - alert: HighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU above 85% for 10 minutes on {{ $labels.instance }}"

      - alert: LowMemory
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Less than 10% RAM available on {{ $labels.instance }}"

      - alert: DiskAlmostFull
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk {{ $labels.mountpoint }} is {{ $value | printf \"%.0f\" }}% full"

  - name: services
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }} on {{ $labels.instance }} is down"

      - alert: GhostBlogDown
        expr: probe_success{instance="ghost-blog"} == 0
        for: 2m
        labels:
          severity: critical
          category: availability
        annotations:
          summary: "Ghost blog is not responding to HTTP probes"

      - alert: SSHBruteForce
        expr: sum by (src_ip) (rate(ssh_failed_logins_total[5m])) * 300 > 10
        for: 1m
        labels:
          severity: critical
          category: security
        annotations:
          summary: "SSH brute force from {{ $labels.src_ip }} — {{ $value | printf \"%.0f\" }} attempts in 5 min"
```

Add to `/etc/prometheus/prometheus.yml`:

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```

```bash
promtool check rules /etc/prometheus/rules/host.yml
sudo systemctl reload prometheus
```

### Connect Grafana to Alertmanager

1. **Connections → Data Sources → Alertmanager**
2. URL: `http://localhost:9093`
3. Save & Test
4. Silences, alert groups, and inhibitions are now visible inside Grafana

---

## 10. Dashboards as Code

### Why this matters

Clicking through Grafana UI to build dashboards doesn't scale, isn't reproducible, and can't be reviewed in a pull request. Infrastructure as Code (IaC) is a core DevOps competency and being able to version-control your dashboards is a genuine differentiator.

### Option A: Grafana Terraform Provider

Terraform manages Grafana resources (dashboards, data sources, alert rules, folders) declaratively. It is the most common approach in professional environments.

**Install Terraform:**

```bash
wget https://releases.hashicorp.com/terraform/1.10.5/terraform_1.10.5_linux_amd64.zip
unzip terraform_1.10.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

**Project structure:**

```
grafana-iac/
├── main.tf
├── variables.tf
├── dashboards/
│   ├── host-overview.json
│   ├── ollama.json
│   └── ghost-slo.json
└── alerts/
    └── host-alerts.tf
```

**`main.tf`:**

```hcl
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 3.0"
    }
  }
}

provider "grafana" {
  url  = "http://localhost:3000"
  auth = var.grafana_api_key
}

# Create an API key in Grafana → Administration → API Keys → Add
variable "grafana_api_key" {
  type      = string
  sensitive = true
}

# Data source
resource "grafana_data_source" "prometheus" {
  type = "prometheus"
  name = "Prometheus"
  url  = "http://localhost:9090"
}

# Import a dashboard from JSON file
resource "grafana_dashboard" "ollama" {
  config_json = file("${path.module}/dashboards/ollama.json")
  folder      = grafana_folder.monitoring.id
}

resource "grafana_folder" "monitoring" {
  title = "Server Monitoring"
}

# Alert rule as code
resource "grafana_rule_group" "host_alerts" {
  name             = "host-alerts"
  folder_uid       = grafana_folder.monitoring.uid
  interval_seconds = 60

  rule {
    name      = "High CPU"
    condition = "A"

    data {
      ref_id = "A"
      datasource_uid = grafana_data_source.prometheus.uid
      query_type = ""
      relative_time_range {
        from = 300
        to   = 0
      }
      model = jsonencode({
        expr  = "100 - (avg(rate(node_cpu_seconds_total{mode='idle'}[5m])) * 100) > 85"
        refId = "A"
      })
    }

    annotations = {
      summary = "CPU usage is above 85%"
    }
    labels = {
      severity = "warning"
    }
  }
}
```

**Usage:**

```bash
cd grafana-iac
terraform init
terraform plan
terraform apply
```

All your dashboards and alerts are now in Git. Changes go through pull requests. Rollback is `git revert` + `terraform apply`.

### Option B: Grafonnet (Jsonnet)

Grafonnet lets you write dashboards programmatically in Jsonnet — useful if you want to generate many similar dashboards from a template (e.g., one RED dashboard per service).

```bash
# Install jsonnet and jsonnet-bundler
sudo apt install jsonnet
go install github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb@latest

mkdir grafonnet-dashboards && cd grafonnet-dashboards
jb init
jb install github.com/grafana/grafonnet/gen/grafonnet-latest@main
```

**`ollama-dashboard.jsonnet`:**

```jsonnet
local grafonnet = import 'github.com/grafana/grafonnet/gen/grafonnet-latest/main.libsonnet';
local dashboard = grafonnet.dashboard;
local panel = grafonnet.panel;
local prometheus = grafonnet.query.prometheus;

dashboard.new('Ollama Metrics')
+ dashboard.withUid('ollama-metrics')
+ dashboard.withDescription('LLM inference observability')
+ dashboard.withPanels([
  panel.timeSeries.new('Tokens per Second')
  + panel.timeSeries.queryOptions.withTargets([
    prometheus.new('$datasource', 'ollama_tokens_per_second')
    + prometheus.withLegendFormat('{{ model }}'),
  ]),

  panel.timeSeries.new('Request Latency p95')
  + panel.timeSeries.queryOptions.withTargets([
    prometheus.new(
      '$datasource',
      'histogram_quantile(0.95, rate(ollama_request_duration_seconds_bucket[5m]))'
    )
    + prometheus.withLegendFormat('{{ model }}'),
  ]),
])
```

```bash
# Compile to JSON and import to Grafana
jsonnet -J vendor ollama-dashboard.jsonnet > ollama-dashboard.json
curl -X POST http://localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d "{\"dashboard\": $(cat ollama-dashboard.json), \"overwrite\": true}"
```

---

## 11. Cardinality Management

### What is cardinality and why does it matter

Every unique combination of label values creates a separate time series in Prometheus. This is called **cardinality**. High cardinality is the most common cause of Prometheus running out of memory in production.

**Safe — low cardinality:**
```
http_requests_total{method="GET", status="200"}     # 2 label values
http_requests_total{method="POST", status="500"}    # different combination
# Total series: small and bounded
```

**Dangerous — high cardinality:**
```
http_requests_total{user_id="user-12345", url="/api/posts/very-long-slug-here"}
# user_id alone could create millions of series
```

### Detect cardinality problems

**In Prometheus, query the cardinality:**

```promql
# Top 10 metrics by time series count
topk(10, count by (__name__)({__name__=~".+"}))
```

**Using promtool:**

```bash
# Check TSDB stats — shows highest cardinality metrics
promtool tsdb analyze /var/lib/prometheus
```

### Recording rules — pre-aggregation

Recording rules pre-compute expensive queries and store results as new metrics. This is essential for:
- Dashboards with slow queries
- Reducing query load on Prometheus
- Making alert evaluation cheaper

```bash
sudo nano /etc/prometheus/rules/recording.yml
```

```yaml
groups:
  - name: recording_rules
    interval: 1m
    rules:
      # Pre-aggregate CPU across all cores
      - record: job:node_cpu_usage:rate5m
        expr: >
          1 - avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      # Pre-aggregate HTTP error rate per job
      - record: job:http_error_rate:rate5m
        expr: >
          sum by (job, instance) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job, instance) (rate(http_requests_total[5m]))

      # Pre-aggregate Ollama token throughput per model
      - record: model:ollama_tokens:rate5m
        expr: rate(ollama_completion_tokens_total[5m])
```

Reference recording rules in dashboards instead of the raw expensive queries — dashboards load faster and Prometheus memory usage drops.

---

## 12. Long-Term Metric Storage with Grafana Mimir

### Why local Prometheus isn't enough

Prometheus stores data locally with typically 15-30 day retention. For capacity planning, trend analysis, and year-over-year comparisons, you need longer retention. More importantly, understanding how a production metrics backend works is core SRE knowledge.

**Grafana Mimir** is what Grafana Cloud runs under the hood. It is a horizontally scalable, long-term Prometheus-compatible TSDB. Even running it on a single server is valuable experience.

### Install Mimir as a Docker container

```bash
mkdir -p /opt/mimir/data

cat > /opt/mimir/mimir.yaml << 'EOF'
# Single-process mode — all components in one binary
target: all

server:
  http_listen_port: 9009
  grpc_listen_port: 9095

common:
  storage:
    backend: filesystem
    filesystem:
      dir: /data/mimir

blocks_storage:
  tsdb:
    dir: /data/mimir/tsdb
  bucket_store:
    sync_dir: /data/mimir/tsdb-sync

compactor:
  data_dir: /data/mimir/compactor

store_gateway:
  sharding_ring:
    replication_factor: 1

ingester:
  ring:
    replication_factor: 1

limits:
  # Retention: 1 year
  compactor_blocks_retention_period: 8760h
  # Increase cardinality limits for self-hosted
  max_global_series_per_user: 1500000
EOF

docker run -d \
  --name mimir \
  --restart unless-stopped \
  --network host \
  --volume /opt/mimir/mimir.yaml:/etc/mimir/mimir.yaml \
  --volume /opt/mimir/data:/data/mimir \
  grafana/mimir:latest \
  --config.file=/etc/mimir/mimir.yaml
```

### Update Alloy to write to Mimir

Replace or add the `prometheus.remote_write` target in `/etc/alloy/config.alloy`:

```hcl
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://localhost:9009/api/v1/push"
    headers = {
      "X-Scope-OrgID" = "anonymous",
    }
  }
}
```

Update all `forward_to` references in your Alloy config to include `prometheus.remote_write.mimir.receiver`.

### Add Mimir as a Grafana data source

1. **Data Sources → Prometheus** (Mimir is Prometheus-compatible)
2. URL: `http://localhost:9009/prometheus`
3. Custom HTTP headers: `X-Scope-OrgID: anonymous`
4. Save & Test

You now have 1-year metric retention, the same storage backend used by large enterprises, and experience with Mimir's architecture.

---

## Professional Progression Summary

Roughly in order of impact and learnability:

| Stage | What to implement | Skill it demonstrates |
|---|---|---|
| 1 | Alertmanager with routing + inhibition | Production alerting, not just metric collection |
| 2 | SLOs with error budget burn rate alerts | Google SRE methodology |
| 3 | Custom Ollama exporter (Python) | Writing your own exporters, understanding the Prometheus data model |
| 4 | Loki + auth.log security alerting | Security observability, log parsing |
| 5 | PostgreSQL pg_stat_statements + slow query dashboard | Database performance engineering |
| 6 | Beyla eBPF auto-instrumentation on Ghost | eBPF, zero-code instrumentation |
| 7 | Grafana Tempo distributed traces | Distributed tracing, OpenTelemetry |
| 8 | Grafana Pyroscope continuous profiling | The fourth pillar, flame graphs |
| 9 | Dashboards as code with Terraform | Infrastructure as Code for observability |
| 10 | Grafana Mimir for long-term storage | Production-scale metric backends |
| 11 | Recording rules + cardinality management | Prometheus internals, performance tuning |

---

## Reference: Full Port Map

| Service | Port |
|---|---|
| Grafana | 3000 |
| Prometheus | 9090 |
| Alertmanager | 9093 |
| Grafana Alloy UI | 12345 |
| Grafana Tempo (HTTP) | 3200 |
| Grafana Tempo (OTLP gRPC) | 4317 |
| Grafana Tempo (OTLP HTTP) | 4318 |
| Grafana Pyroscope | 4040 |
| Grafana Mimir | 9009 |
| Loki | 3100 |
| Node Exporter | 9100 |
| cAdvisor | 9093 *(check conflict with Alertmanager)* |
| postgres_exporter | 9187 |
| redis_exporter | 9121 |
| apache_exporter | 9117 |
| mysqld_exporter | 9104 |
| blackbox_exporter | 9115 |
| Ollama custom exporter | 9500 |

> **Port conflict note:** cAdvisor was mapped to `9093` in Part 1 and Alertmanager also defaults to `9093`. Change cAdvisor to `9094` (`--publish 9094:8080`) and update the Alloy scrape config accordingly.

---

*Written for Ubuntu 24.04.3 LTS · Grafana 12.4.0 · Alloy v1.13.2 · Prometheus 3.x · Kernel 6.8*
