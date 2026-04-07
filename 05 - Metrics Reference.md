# Metrics Reference

Every metric collected by the [[04 - Poller Daemon|poller]], their source, formula, unit, and why it matters.

---

## Metric Categories

| Category | Metrics | Signal |
|----------|---------|--------|
| Compute | `cpu_pct`, `gc_pause_rate_ms_per_s` | Is the node overloaded? |
| Memory | `heap_pct`, `heap_used_bytes`, `heap_max_bytes` | Is memory pressure causing GC? |
| Storage (Capacity) | `disk_store_bytes`, `disk_total_bytes`, `disk_pct` | Are we approaching disk limits? |
| Storage (Throughput) | `io_read_bps`, `io_write_bps` | What's the I/O workload? |
| Indexing | `index_total` | What's the cumulative indexing volume? |
| Thread Pools | `thread_pool.write.*`, `thread_pool.search.*`, `tp_*_rejected_per_s` | Is there backpressure? |
| Process | `fd_count`, `fd_limit`, `fd_pct`, `pid` | Are we approaching OS limits? |

---

## Node Metrics (from `/_nodes/stats`)

### `cpu_pct`
- **Source:** `process.cpu.percent`
- **Unit:** Percentage (0–100)
- **Meaning:** CPU usage of the OpenSearch JVM process specifically — not system-wide
- **Why:** Detects CPU saturation causing indexing/search latency

### `heap_used_bytes` / `heap_max_bytes` / `heap_pct`
- **Source:** `jvm.mem.heap_used_in_bytes`, `jvm.mem.heap_max_in_bytes`
- **Formula:** `heap_pct = heap_used_bytes / heap_max_bytes × 100`
- **Why:** High heap triggers GC pressure → search/index latency spikes
- **Thresholds:** Warn 75% / Critical 90%

> [!warning] Heap vs System RAM
> System RAM can be 90%+ utilized with no problem — OpenSearch uses it as the page cache. **Only JVM heap triggers alarms.**

### `disk_store_bytes` / `disk_total_bytes` / `disk_pct`
- **Source:**
  - `indices.store.size_in_bytes` — bytes OpenSearch index data actually occupies
  - `fs.total.total_in_bytes` — total filesystem capacity
- **Formula:** `disk_pct = disk_store_bytes / disk_total_bytes × 100`
- **Why:** Disk exhaustion causes shard allocation failures (cluster turns RED)

### `index_total`
- **Source:** `indices.indexing.index_total` (cumulative counter, resets on restart)
- **Unit:** Operations (count)
- **Why:** Baseline counter → used for indexing rate trends via delta over time

### `gc_young_ms` / `gc_old_ms` → `gc_pause_rate_ms_per_s`
- **Source:** `jvm.gc.collectors.young.collection_time_in_millis`, `…old…`
- **Formula:** `(Δyoung_ms + Δold_ms) / elapsed_seconds`
- **Unit:** Milliseconds of GC pause per second of wall time
- **Range:** 0–1000 (values > 100 = noticeable pressure)
- **Why:** Converts cumulative GC counters into an immediate pressure signal

---

## Thread Pool Metrics (per pool: `write`, `search`)

### `thread_pool.<pool>.queue`
- Number of requests waiting for a worker thread
- **High value** = backpressure building up → responses slowing

### `thread_pool.<pool>.active`
- Currently executing tasks
- **High active + full queue** = node is saturated

### `thread_pool.<pool>.rejected` → `tp_<pool>_rejected_per_s`
- **Source:** Cumulative rejected counter from `thread_pool.<pool>.rejected`
- **Formula:** `Δrejected / elapsed_seconds`
- **Unit:** Rejected tasks per second
- **Why:** Live overload rate — non-zero means users are seeing errors right now

| Signal | Action |
|--------|--------|
| `queue > 0` | Monitor — backpressure building |
| `queue > 50` | Investigate — indexing/search may slow |
| `rejected_per_s > 0` | 🔴 Immediate — users getting errors |

---

## Host Metrics (from `/proc/<pid>`)

### `pid`
- OpenSearch process ID found via `pgrep OS_PROCESS_KEYWORD`
- Helps confirm which process was sampled, detect restarts

### `fd_count` / `fd_limit` / `fd_pct`
- **Source:** Count of files in `/proc/<pid>/fd/` + `/proc/<pid>/limits`
- **Formula:** `fd_pct = fd_count / fd_limit × 100`
- **Why:** FD exhaustion prevents new connections, segment opens, and merges

> [!caution] FD Exhaustion
> When the FD limit is hit, OpenSearch cannot open new files. This causes connection failures and Lucene segment merge failures — cluster effectively goes read-only.

### `io_read_bytes` / `io_write_bytes`
- **Source:** `/proc/<pid>/io` fields `read_bytes` and `write_bytes`
- Cumulative byte counters since process start

### `io_read_bps` / `io_write_bps`
- **Formula:** `Δio_bytes / elapsed_seconds`
- **Unit:** Bytes per second
- **Why:** Reveals storage throughput workload and detects abnormal I/O spikes

---

## Diagnostic Talk Track

Use this order when presenting metrics:

```
1. CPU + Heap + GC rate      →  compute and memory pressure
2. Threadpool queue/rejected →  user-facing overload risk
3. Disk + FD usage           →  hard capacity limits
4. I/O rates + index_total   →  workload behavior over time
```

---

## Related Notes

- [[04 - Poller Daemon]] — how metrics are collected
- [[09 - Alert Thresholds & Triage]] — what to do when values are high
- [[07 - OpenSearch API Reference]] — raw API fields mapped to metrics
