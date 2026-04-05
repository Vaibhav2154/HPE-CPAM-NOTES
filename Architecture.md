# HPE-Monitor — Architecture Diagram (Eraser.io)

> Scope: **OpenSearch only** — Kafka and Logstash stubs are excluded.

---

## System Overview

The project has **two independently runnable subsystems** that share a common data store:

| Subsystem | Entry point | Purpose |
|---|---|---|
| **Background Poller** | `python -m poller` | Continuously collects OpenSearch metrics and writes timestamped JSONL records |
| **CLI Monitor** | `python -m monitor` | Interactive TUI that reads live and historical data and renders rich views |

---

## Component Map

### 1 — Background Poller ([poller/](file:///home/vaibhi/Dev/HPE-Monitor/monitor/metrics_service.py#106-134))

```
python -m poller   (__main__.py)
        │
        ▼
  Poller Orchestrator  (poller.py)
  ┌──────────────────────────────────────┐
  │  ┌─────────────────────────────────┐ │
  │  │  OpenSearch API Collector       │ │  GET /_nodes/stats/process,jvm,
  │  │  (collectors/opensearch_api.py) │─┼──────── fs,thread_pool,indices
  │  └─────────────────────────────────┘ │
  │  ┌─────────────────────────────────┐ │
  │  │  System Collector               │ │  /proc/<pid>/fd
  │  │  (collectors/system.py)         │─┼──────── /proc/<pid>/io
  │  └─────────────────────────────────┘ │
  │                                      │
  │  Rate computation (delta math)       │
  │   · gc_pause_rate_ms_per_s           │
  │   · tp_write/search_rejected_per_s   │
  │   · io_read_bps / io_write_bps       │
  └──────────────────────────────────────┘
        │
        ▼
  JSONL Writer  (storage/writer.py)
        │  daily-rotated files
        ▼
  poller/data/   ← shared data store
```

**Metrics collected per poll cycle:**

| Domain | Metric(s) | Source |
|---|---|---|
| CPU | `cpu_pct` | `process.cpu.percent` via `_nodes/stats` |
| JVM Heap | `heap_pct`, `heap_used_bytes`, `heap_max_bytes` | `jvm.mem.*` |
| Disk | `disk_store_bytes`, `disk_total_bytes`, `disk_pct` | `indices.store` / `fs.total` |
| GC | `gc_pause_rate_ms_per_s` | delta of cumulative `jvm.gc.collectors.*` |
| Thread Pool | `tp_write_queue`, `tp_write_rejected_per_s`, `tp_search_queue`, `tp_search_rejected_per_s` | `thread_pool.write/search` |
| Indexing | `index_total` | `indices.indexing.index_total` |
| File Descriptors | `fd_count`, [fd_limit](file:///home/vaibhi/Dev/HPE-Monitor/poller/collectors/system.py#75-107), `fd_pct` | `/proc/<pid>/fd` |
| I/O | `io_read_bps`, `io_write_bps` | `/proc/<pid>/io` (delta) |

---

### 2 — CLI Monitor (`monitor/`)

```
python -m monitor  (__main__.py)
        │
        ▼
   cli.py  (Click CLI)
        │
        ├──[--service opensearch]──► opensearch_menu()
        └──[no flag]────────────────► main_service_menu()
                                            │
                                            ▼
                                    menus.py  (TerminalMenu)
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    ▼                       ▼                       ▼
            Quick Summary          Historical Trends         Cluster Health
            Node Performance       Index Deep Dive           Shard Overview
            Log Browser            Root Cause Analysis       Data Streams
                    │                       │
                    ▼                       ▼
              client.py             metrics_service.py  (MetricsProvider)
          (OpenSearch HTTP)         (routing & aggregation)
                    │                       │
         ┌──────────┘          ┌────────────┴────────────┐
         ▼                     ▼                         ▼
    OpenSearch            poller/data/          Prometheus
    REST API :9200        JSONL Store           HTTP API :9090
    (real-time)           (historical)          (historical fallback)
                                                         │
                                         ┌───────────────┘
                                         ▼
                              Performance Analyzer :9600
                              (bottleneck triage only)
```

**Metric routing logic inside [MetricsProvider](file:///home/vaibhi/Dev/HPE-Monitor/monitor/metrics_service.py#46-455):**

| Timeframe | Backend used |
|---|---|
| `real-time` | OpenSearch `_nodes/stats` (live) |
| ≤ 1 h | OpenSearch `_nodes/stats` (live snapshot) |
| > 1 h | Poller JSONL → Prometheus PromQL (fallback) |
| `--source poller` | Poller JSONL only |
| `--source prometheus` | Prometheus only |

---

### 3 — External Systems

| System                          | Protocol / Port    | Used by                                                                                                              |
| ------------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| OpenSearch node                 | HTTPS/HTTP `:9200` | Poller (metrics), Monitor (live stats + log search)                                                                  |
| OpenSearch Performance Analyzer | HTTP `:9600`       | Monitor (`MetricsProvider.fetch_performance_analyzer_metrics`)                                                       |
| Prometheus                      | HTTP `:9090`       | Monitor ([MetricsProvider](file:///home/vaibhi/Dev/HPE-Monitor/monitor/metrics_service.py#46-455) historical trends) |
| Linux `/proc` FS                | local              | Poller system collector (FD count, I/O bytes)                                                                        |

---
