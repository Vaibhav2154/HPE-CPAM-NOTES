# Poller Daemon

The `poller/` daemon is a lightweight background process that continuously samples OpenSearch health metrics and appends them to daily JSONL files. It is the primary data source for [[03 - Monitor CLI|Historical Trends]] in the monitor.

---

## Running the Poller

```bash
# Standard run (API metrics only — no FD/IO)
python -m poller --interval 15

# Full metrics (FD + I/O from /proc) — requires same OS user as OpenSearch or sudo
sudo venv/bin/python -m poller --interval 15

# Fast polling with live output
python -m poller --interval 5 --verbose

# Custom output directory
python -m poller --interval 15 --output-dir /var/log/opensearch-metrics
```

> [!important] Sudo Requirement for FD/IO
> File descriptor and I/O metrics are read from `/proc/<pid>/fd` and `/proc/<pid>/io`. These require the poller to run as the same OS user as OpenSearch, or with `sudo` using the venv Python binary. Without those permissions, the `host.fd_note` field will explain what's missing.

---

## Polling Loop — Step by Step

Each cycle (every `interval` seconds):

```
1.  opensearch_api.collect(client)
    └─► GET /_nodes/stats?filter_path=nodes.*.jvm,os,process,fs,indices,thread_pool
        Returns: cpu_pct, heap_*, disk_*, index_total, gc_*_ms, thread_pool.*

2.  system.collect(OS_PROCESS_KEYWORD)
    └─► Find PID: pgrep "org.opensearch.bootstrap.OpenSearch"
        Read /proc/<pid>/fd  → fd_count, fd_limit
        Read /proc/<pid>/io  → io_read_bytes, io_write_bytes

3.  Compute delta metrics (vs previous cycle)
    ├─► gc_pause_rate_ms_per_s = (Δyoung_gc_ms + Δold_gc_ms) / elapsed_s
    ├─► tp_write_rejected_per_s  = Δwrite_rejected / elapsed_s
    ├─► tp_search_rejected_per_s = Δsearch_rejected / elapsed_s
    ├─► io_read_bps  = Δio_read_bytes / elapsed_s
    └─► io_write_bps = Δio_write_bytes / elapsed_s

4.  Assemble record:  {ts, timestamp, nodes: {…}, host: {…}}

5.  append_record(output_dir, record)
    └─► Appends one JSON line to  poller/data/metrics_YYYY-MM-DD.jsonl
```

---

## Output Record Schema

```json
{
  "ts": 1774539708,
  "timestamp": "2026-03-26T21:11:48.587716+05:30",
  "nodes": {
    "node-1": {
      "cpu_pct": 12.5,
      "heap_pct": 68.2,
      "heap_used_bytes": 4294967296,
      "heap_max_bytes": 6291456000,
      "disk_store_bytes": 17179869184,
      "disk_total_bytes": 107374182400,
      "disk_pct": 16.0,
      "index_total": 4827341,
      "gc_pause_rate_ms_per_s": 3.142,
      "thread_pool": {
        "write": {"queue": 0, "active": 2, "rejected": 0},
        "search": {"queue": 0, "active": 1, "rejected": 0}
      },
      "tp_write_rejected_per_s": 0.0,
      "tp_search_rejected_per_s": 0.0
    }
  },
  "host": {
    "pid": 12345,
    "fd_count": 892,
    "fd_limit": 65536,
    "fd_pct": 1.36,
    "io_read_bytes": 9876543210,
    "io_write_bytes": 1234567890,
    "io_read_bps": 0.0,
    "io_write_bps": 24576.0
  }
}
```

> [!note] First Poll Cycle
> Rate metrics (`gc_pause_rate_ms_per_s`, `io_*_bps`, `tp_*_rejected_per_s`) are `0.0` on the first cycle because there is no previous snapshot to diff against. `elapsed_s` defaults to `interval` on the first tick.

---

## Counter Reset Safety

The `_safe_delta(current, previous)` function clamps all deltas to `0`:

```python
delta = current - previous
return max(0.0, float(delta))
```

This silently discards negative deltas that occur when:
- OpenSearch node restarts (counters reset to 0)
- PID changes (new process, fresh counters)
- API returns unexpected values

---

## File Storage

| Pattern | Example |
|---------|---------|
| `poller/data/metrics_YYYY-MM-DD.jsonl` | `metrics_2026-04-06.jsonl` |

- New file created daily at midnight (derived from `datetime.now().date()`)
- Each file is one JSON object per line (JSONL format)
- Files are safe to `tail -f`, `jq`, `grep`, `wc -l`

```bash
# Count records today
wc -l poller/data/metrics_$(date +%F).jsonl

# Pretty-print the last record
tail -1 poller/data/metrics_$(date +%F).jsonl | python3 -m json.tool

# Watch CPU in real time
while sleep 5; do tail -1 poller/data/metrics_$(date +%F).jsonl | python3 -c "
import sys, json
r=json.load(sys.stdin)
for n,v in r['nodes'].items(): print(n, 'cpu=', v['cpu_pct'])
"; done
```

---

## Configuration

See [[06 - Configuration]] for all environment variables. Key poller settings:

| Variable | Default | Meaning |
|----------|---------|---------|
| `OPENSEARCH_HOST` | `localhost` | OpenSearch hostname |
| `OPENSEARCH_PORT` | `9200` | OpenSearch API port |
| `POLL_INTERVAL_SECONDS` | `15` | Polling interval (overridden by `--interval` flag) |
| `OS_PROCESS_KEYWORD` | `org.opensearch.bootstrap.OpenSearch` | String used to find OpenSearch PID via `pgrep` |
| `OS_DATA_DEVICE` | `` (auto) | Block device for disk stats; auto-detected from `/` mount if empty |

---

## Related Notes

- [[05 - Metrics Reference]] — what every field means
- [[02 - Architecture]] — how poller integrates with monitor
- [[06 - Configuration]] — environment variables
