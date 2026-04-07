# Monitor CLI

The `monitor/` module is an interactive terminal application built with **Rich** and **Click**. It provides 7 views into OpenSearch cluster health, routed from a single CLI entry point.

---

## Launching

```bash
# Interactive menus (default)
python -m monitor

# Direct to OpenSearch service
python -m monitor --service opensearch

# Non-interactive one-shot summary (great for scripting/cron)
python -m monitor --summary

# Auto-refresh every 10 seconds
python -m monitor --summary --watch 10

# Historical trends — 6-hour window
python -m monitor --timeframe 6h

# Force data source
python -m monitor --source poller        # JSONL only
python -m monitor --source prometheus    # Prometheus only

# Real-time node metrics (bypass history)
python -m monitor --timeframe real-time
```

---

## CLI Flags

| Flag | Values | Default | Description |
|------|--------|---------|-------------|
| `--timeframe` | `real-time`, `30m`, `1h`, `6h`, `4d` | `1h` | Routes metric source by time window |
| `--source` | `auto`, `poller`, `prometheus` | `auto` | Historical backend selection |
| `--service` | `opensearch` | — | Skip service menu, go direct |
| `--summary` | flag | off | Jump to Quick Summary |
| `--watch` | integer (seconds) | off | Auto-refresh interval |

---

## Views

### View 1 — Quick Summary

> **10-second health check**

The fastest way to answer: *"Is OpenSearch okay right now?"*

**Shows:**
- Cluster status badge (🟢 GREEN / 🟡 YELLOW / 🔴 RED)
- CPU % (cluster average + per-node warnings)
- JVM Heap % — the primary memory alarm
- System RAM (informational — high RAM is normal, page cache)
- Disk usage with watermark warnings
- Active index count + largest index
- Shard counts (active / relocating / initializing / unassigned)
- Per-node alerts for any threshold breaches

**Alert Thresholds:**
| Metric | ⚠️ Warn | 🔴 Critical |
|--------|--------|------------|
| CPU | > 70% | > 90% |
| JVM Heap | > 75% | > 90% |
| Disk | > 80% | > 90% |

---

### View 2 — Historical Trends

> **5-minute bucket time series**

Shows CPU %, JVM Heap, and Indexing Rate over a selected time window.

**Source routing:**
- `--timeframe real-time` → Live API
- `≤ 1h` → Poller JSONL first, Prometheus fallback
- `> 1h` → Prometheus (long-window PromQL)

**Prometheus metrics used:**
- `opensearch_jvm_mem_heap_used_bytes`
- `opensearch_os_cpu_percent`
- `opensearch_indices_indexing_index_total` (primary)
- `opensearch_indices_indexing_index_count` (fallback)

---

### View 3 — Cluster Health

> **Detailed cluster status**

Plain-English explanations of cluster state with full shard counts and pending task details.

---

### View 4 — Index Deep Dive

> **Index table + shard drill-down**

All indices sorted by size (descending). Select any index to drill into its shard layout (primary/replica placement, state, size).

---

### View 5 — Node Performance

> **Per-node diagnostics**

For each node:
- CPU %, JVM Heap %, OS RAM %, Disk %
- Indexing ops count + Search query count
- Bottleneck diagnostics (appears only when CPU or Disk crosses threshold)

**Bottleneck source:** Performance Analyzer API at port `9600`:
- `Disk_Utilization` + `IO_TotWait` → plain-English explanation

---

### View 6 — Shard Overview

> **All shards by state**

Groups shards into: `STARTED` / `RELOCATING` / `INITIALIZING` / `UNASSIGNED`. Unassigned shards are highlighted in red with their `reason` field displayed.

> [!warning] Unassigned Shards
> Unassigned shards on a single-node cluster are usually replicas that can't be placed (no other node). Set `number_of_replicas: 0` for single-node setups.

---

### View 7 — Data Streams

> **Pipeline staleness detection**

Lists all data streams sorted by size. Key feature: **staleness alert** based on `maximum_timestamp` (newest document epoch).

| Staleness | Meaning |
|-----------|---------|
| < 60 min | ✅ Normal |
| 60–240 min | ⚠️ Warn — check upstream pipeline |
| > 240 min | 🔴 Critical — Logstash/Kafka/Beats likely down |

> [!note] System RAM is Normal
> OpenSearch aggressively uses available RAM as filesystem page cache. High OS memory usage is expected and healthy. Only JVM Heap % triggers alarms.

---

## Code Architecture

```
cli.py      →  Click flags → calls menus.py or a direct view
menus.py    →  Arrow-key service/view selection
client.py   →  All HTTP (one place, all views call this)
config.py   →  Thresholds + connection params
utils.py    →  format_bytes(), status_symbol(), timeframe_to_minutes()
metrics_service.py  →  MetricsProvider — routes between poller and Prometheus
poller_history.py   →  JSONL reader + 5-min bucket aggregator
views/*.py  →  Pure presentation (Rich panels, tables, bars)
```

---

## Related Notes

- [[07 - OpenSearch API Reference]] — endpoints used by each view
- [[02 - Architecture]] — data flow diagram
- [[06 - Configuration]] — thresholds you can tune
- [[09 - Alert Thresholds & Triage]] — when to act on an alert
