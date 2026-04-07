# Configuration

All configuration is driven by `.env` file and environment variables. Both `monitor/` and `poller/` load the same `.env` via `python-dotenv`.

---

## Setup

```bash
# Copy the template
cp .env.example .env

# Edit with your cluster details
nano .env
```

---

## `.env` Variables

### OpenSearch Connection

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENSEARCH_HOST` | `localhost` | OpenSearch hostname or IP |
| `OPENSEARCH_PORT` | `9200` | OpenSearch API port |
| `OPENSEARCH_USER` | `admin` | HTTP basic auth username |
| `OPENSEARCH_PASS` | `admin` | HTTP basic auth password |
| `OPENSEARCH_SSL` | `false` | Enable HTTPS (`true`/`false`) |

### Prometheus (Monitor fallback)

| Variable | Default | Description |
|----------|---------|-------------|
| `PROMETHEUS_HOST` | same as `OPENSEARCH_HOST` | Prometheus server host |
| `PROMETHEUS_PORT` | `9090` | Prometheus HTTP port |
| `PROMETHEUS_SCHEME` | `http` | `http` or `https` |
| `PROMETHEUS_TIMEOUT_SECONDS` | `6` | Request timeout |

### Performance Analyzer (Monitor bottleneck diagnostics)

| Variable | Default | Description |
|----------|---------|-------------|
| `PA_HOST` | same as `OPENSEARCH_HOST` | Performance Analyzer host |
| `PA_PORT` | `9600` | PA plugin port |
| `PA_SCHEME` | `http` | `http` or `https` |
| `PA_TIMEOUT_SECONDS` | `4` | Request timeout |

### Poller Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `POLL_INTERVAL_SECONDS` | `15` | Polling interval in seconds |
| `OS_PROCESS_KEYWORD` | `org.opensearch.bootstrap.OpenSearch` | Substring used by `pgrep` to find OpenSearch PID |
| `OS_DATA_DEVICE` | `` (auto) | Block device override (e.g. `sda`); auto-detected if empty |

### Monitor/History Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `POLLER_DATA_DIR` | `poller/data` | Path to JSONL files for historical trends |
| `HISTORICAL_METRICS_SOURCE` | `auto` | `auto` \| `poller` \| `prometheus` |
| `OPENSEARCH_INDEX` | `system-logs-*` | Index pattern for log queries |

---

## Alert Thresholds (`monitor/config.py`)

These are Python constants — edit `monitor/config.py` directly to change thresholds:

```python
# CPU thresholds (percentage)
CPU_WARN = 70
CPU_CRIT = 90

# JVM Heap thresholds (percentage)  ← most important
HEAP_WARN = 75
HEAP_CRIT = 90

# System RAM thresholds (informational only)
MEM_WARN  = 85
MEM_CRIT  = 95

# Disk thresholds (percentage)
DISK_WARN = 80
DISK_CRIT = 90
```

> [!tip] Data Stream Staleness
> Stream staleness thresholds are currently hardcoded in `data_streams.py`:
> - Warn: 60 minutes since last document
> - Critical: 240 minutes

---

## Keyword Tags & Root Cause Patterns

`monitor/config.py` also contains `KEYWORD_TAGS` and `ROOT_CAUSE_PATTERNS` used by the anomaly interpretation engine:

```python
KEYWORD_TAGS = {
    "CPU":    ["cpu", "merge", "aggregat", "lucene"],
    "HEAP":   ["heap", "memory", "outofmemory", "oom"],
    "GC":     ["gc", "garbage", "pause", "overhead"],
    "DISK":   ["disk", "watermark", "flood", "space", "read-only"],
    "THREAD": ["rejected", "queue", "bulk", "thread pool"],
    "SEARCH": ["timeout", "slowlog", "slow", "circuit"],
}
```

These patterns map OpenSearch log keywords to root cause labels for the diagnostics panel.

---

## Related Notes

- [[03 - Monitor CLI]] — CLI flags that override some config
- [[04 - Poller Daemon]] — poller-specific config
- [[09 - Alert Thresholds & Triage]] — what to do when thresholds are breached
