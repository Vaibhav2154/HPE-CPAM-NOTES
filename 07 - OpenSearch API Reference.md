# OpenSearch API Reference

All HTTP calls in the monitor and poller are made through `monitor/client.py`. No view or collector module makes raw HTTP calls directly — everything goes through the client layer.

---

## Poller API

### `GET /_nodes/stats`

**Used in:** `poller/collectors/opensearch_api.py`

Fetches per-node statistics for CPU, JVM, filesystem, indices, and thread pools.

**Filter path used:**
```
nodes.*.jvm,os,process,fs,indices,thread_pool
```

**Fields extracted per node:**

| API Path | Poller Field | Notes |
|----------|-------------|-------|
| `process.cpu.percent` | `cpu_pct` | OpenSearch process CPU only |
| `jvm.mem.heap_used_in_bytes` | `heap_used_bytes` | |
| `jvm.mem.heap_max_in_bytes` | `heap_max_bytes` | |
| `jvm.gc.collectors.young.collection_time_in_millis` | `gc_young_ms` | Cumulative |
| `jvm.gc.collectors.old.collection_time_in_millis` | `gc_old_ms` | Cumulative |
| `indices.store.size_in_bytes` | `disk_store_bytes` | OpenSearch-owned data |
| `fs.total.total_in_bytes` | `disk_total_bytes` | Filesystem capacity |
| `indices.indexing.index_total` | `index_total` | Cumulative ops counter |
| `thread_pool.write.queue` | `thread_pool.write.queue` | |
| `thread_pool.write.active` | `thread_pool.write.active` | |
| `thread_pool.write.rejected` | `thread_pool.write.rejected` | Cumulative |
| `thread_pool.search.queue` | `thread_pool.search.queue` | |
| `thread_pool.search.active` | `thread_pool.search.active` | |
| `thread_pool.search.rejected` | `thread_pool.search.rejected` | Cumulative |

---

## Monitor API Calls (`monitor/client.py`)

### `GET /_cluster/health`
**Function:** `fetch_cluster_health()`  
**Used in:** Quick Summary, Cluster Health

| Field | Purpose |
|-------|---------|
| `status` | `green` / `yellow` / `red` — primary health badge |
| `number_of_nodes` | Active node count |
| `active_shards` | Shard panel totals |
| `relocating_shards` | Shard panel warning |
| `initializing_shards` | Shard panel warning |
| `unassigned_shards` | 🔴 Red alert — potential data unavailability |
| `number_of_pending_tasks` | Cluster Health view detail |

---

### `GET /_cluster/stats`
**Function:** `fetch_cluster_stats()`  
**Used in:** Quick Summary

Pre-aggregated cluster-wide totals (OpenSearch sums across all nodes server-side).

| Path | Purpose |
|------|---------|
| `nodes.os.cpu.percent` | Cluster-average CPU % |
| `nodes.os.mem.used_in_bytes` / `total_in_bytes` | System RAM (informational) |
| `nodes.jvm.mem.heap_used_in_bytes` / `heap_max_in_bytes` | JVM Heap — primary alarm |
| `nodes.fs.total_in_bytes` / `available_in_bytes` | Disk watermark warnings |
| `indices.docs.count` | Total document count |
| `indices.indexing.index_total` | Cumulative indexing operations |
| `indices.search.query_total` | Cumulative search query count |

---

### `GET /_nodes/stats/os,jvm,fs,indices`
**Function:** `fetch_node_stats()`  
**Used in:** Quick Summary (per-node warnings), Node Performance

Per-node breakdown for the monitor's live views.

| Path | Purpose |
|------|---------|
| `os.cpu.percent` | Per-node CPU % |
| `jvm.mem.heap_used_in_bytes` / `heap_max_in_bytes` | Per-node JVM Heap |
| `os.mem.used_in_bytes` / `total_in_bytes` | Per-node RAM (informational) |
| `fs.total.total_in_bytes` / `available_in_bytes` | Per-node disk (byte-accurate) |
| `indices.indexing.index_total` | Per-node indexing ops |
| `indices.search.query_total` | Per-node search queries |

---

### `GET /_cat/allocation?v&format=json`
**Function:** `fetch_disk_allocation()`  
**Used in:** Quick Summary (per-node watermark warning text)

| Field | Purpose |
|-------|---------|
| `node` | Node name for targeted warning |
| `disk.used` | Human-readable used disk |
| `disk.total` | Human-readable total disk |

> Node Performance uses `fs.total` from `/_nodes/stats` (byte-accurate) — this endpoint is only for warning labels.

---

### `GET /_cat/indices?v&s=store.size:desc&format=json`
**Function:** `fetch_indices()`  
**Used in:** Quick Summary (largest index), Index Deep Dive

| Field | Purpose |
|-------|---------|
| `index` | Index name |
| `store.size` | Size string (parsed to bytes internally) |
| `docs.count` | Document count |
| `health` | Colour-coded health badge |
| `pri` / `rep` | Primary / replica shard counts |

---

### `GET /_cat/shards?v&format=json`
**Function:** `fetch_shards(index=None)`  
**Used in:** Quick Summary, Shard Overview, Index Deep Dive

Optional `index` parameter to filter to a single index for drill-down.

| Field | Purpose |
|-------|---------|
| `state` | `STARTED` / `RELOCATING` / `INITIALIZING` / `UNASSIGNED` |
| `index` | Index the shard belongs to |
| `shard` | Shard number |
| `prirep` | `p` (primary) or `r` (replica) |
| `node` | Node the shard lives on |
| `store` | Shard size on disk |
| `reason` | Why shard is unassigned (UNASSIGNED only) |

---

### `GET /_data_stream`
**Function:** `fetch_data_streams()`  
**Used in:** Data Streams

| Field | Purpose |
|-------|---------|
| `name` | Stream name |
| `store_size` / `store_size_bytes` | Disk usage |
| `indices` | Backing index count |
| `maximum_timestamp` | Newest document epoch ms → pipeline staleness detection |

---

### `GET /api/v1/query_range` (Prometheus — port 9090)
**Function:** `fetch_historical_trends()` via `MetricsProvider`  
**Used in:** Historical Trends (fallback / long-window routing)

PromQL queries executed (5-minute `max_over_time` buckets):
```promql
max(max_over_time(opensearch_os_cpu_percent[5m]))
max(max_over_time(opensearch_jvm_mem_heap_used_bytes[5m]))
increase(opensearch_indices_indexing_index_total[5m])
```

---

### `GET /_plugins/_performanceanalyzer/metrics` (Performance Analyzer — port 9600)
**Function:** `fetch_bottleneck_metrics(node_name)`  
**Used in:** Node Performance diagnostics (CPU or Disk above threshold)

Metrics fetched: `Disk_Utilization`, `IO_TotWait`  
→ Used to generate plain-English bottleneck explanation in the UI.

---

## Related Notes

- [[03 - Monitor CLI]] — which view uses which endpoint
- [[05 - Metrics Reference]] — field-to-metric mapping
- [[04 - Poller Daemon]] — poller API collection
