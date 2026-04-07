# Architecture

## System Overview

HPE-Monitor has two independent processes that share an OpenSearch connection and collaborate through the filesystem (JSONL files).

```
                          ┌──────────────────────────────────────────────┐
                          │              OpenSearch Cluster               │
                          │                  :9200                        │
                          └───────────┬─────────────────┬────────────────┘
                                      │ REST API         │ REST API
                                      │                  │
                          ┌───────────▼──────┐  ┌────────▼──────────────┐
                          │  poller daemon   │  │    monitor CLI         │
                          │  (background)    │  │    (interactive)       │
                          └───────────┬──────┘  └────────┬──────────────┘
                                      │                  │
                              writes JSONL         reads JSONL
                               (daily files)     (historical trends)
                                      │                  │
                          ┌───────────▼──────────────────▼──────────────┐
                          │          poller/data/metrics_YYYY-MM-DD.jsonl│
                          └─────────────────────────────────────────────┘
```

---

## Component Map

### `poller/` — Metrics Collector

```
poller/
├── __main__.py          Entry: python -m poller
├── poller.py            ← Main polling loop (orchestrator)
├── config.py            Connection + settings (reads .env)
├── collectors/
│   ├── opensearch_api.py  Hits /_nodes/stats — CPU, heap, disk, GC, threadpool
│   └── system.py          Reads /proc/<pid> — FD count, I/O bytes
├── storage/
│   └── writer.py          append_record() — writes JSONL with daily rotation
└── data/                  Output: metrics_YYYY-MM-DD.jsonl
```

**Polling loop (every N seconds):**
1. `opensearch_api.collect()` — fetch per-node API metrics
2. `system.collect()` — find OpenSearch PID, read `/proc/<pid>`
3. Compute rate metrics (GC rate, rejected/s, I/O bps) via delta from previous cycle
4. Assemble full record `{ts, timestamp, nodes, host}`
5. `append_record()` — append JSON line to today's file

### `monitor/` — Interactive CLI

```
monitor/
├── __main__.py          Entry: python -m monitor
├── cli.py               Click flags: --timeframe, --source, --service, --summary, --watch
├── client.py            All HTTP calls (single source of truth — no view touches HTTP)
├── config.py            Thresholds + connection (reads .env)
├── menus.py             Arrow-key navigation menus
├── metrics_service.py   MetricsProvider: poller JSONL + Prometheus routing
├── poller_history.py    Reads + aggregates poller JSONL files
├── utils.py             format_bytes, status_symbol, timeframe_to_minutes, …
└── Opensearch/views/
    ├── quick_summary.py      View 1
    ├── trends.py             View 2
    ├── cluster_health.py     View 3
    ├── index_deep_dive.py    View 4
    ├── node_performance.py   View 5
    ├── shard_overview.py     View 6
    └── data_streams.py       View 7
```

---

## Data Flow — Historical Trends

```
--timeframe real-time  →  Live OpenSearch API call
--timeframe ≤ 1h       →  poller JSONL  ──(fallback)──►  Prometheus API
--timeframe > 1h       →  Prometheus API (long window, auto-routed)
--source poller        →  Force JSONL only
--source prometheus    →  Force Prometheus only
--source auto          →  Poller first, Prometheus if not enough data
```

### JSONL Aggregation (5-minute buckets)

`poller_history.py` reads all `.jsonl` files in `poller/data/`, filters records in the requested time window, and aggregates into 5-minute buckets picking max values for:
- `cpu_pct` → max bucket value
- `heap_used_bytes` → max bucket value  
- `index_total` → converted to indexing rate (ops/s) via delta

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| JSONL instead of SQLite | Append-only, zero schema migrations, `tail -f` friendly |
| Per-process metrics from `/proc` | `/proc/<pid>` scopes FD + I/O to OpenSearch only, not system-wide |
| Delta clamped to 0 | Handles node restarts and counter resets gracefully |
| Single `client.py` for all HTTP | Views stay pure presentation; no HTTP logic scattered |
| Prometheus as fallback only | Prometheus may not always be deployed; poller is always local |

---

## Related Notes

- [[04 - Poller Daemon]] — polling loop details
- [[03 - Monitor CLI]] — view descriptions
- [[07 - OpenSearch API Reference]] — all endpoints
- [[05 - Metrics Reference]] — what each metric means
