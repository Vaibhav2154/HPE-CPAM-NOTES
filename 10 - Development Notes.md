# Development Notes

Technical notes for extending and maintaining HPE-Monitor.

---

## Code Conventions

### HTTP Layer — `monitor/client.py`

**Rule:** No view or service module makes raw HTTP calls. All HTTP goes through `client.py`.

This ensures:
- Single place to change auth, SSL, timeouts
- Views remain pure presentation (Rich panels only)
- Easy to mock for testing

```python
# ✅ Correct — call a client function
from monitor.client import fetch_node_stats
stats = fetch_node_stats()

# ❌ Wrong — raw HTTP in a view
import requests
resp = requests.get("http://localhost:9200/_nodes/stats")
```

---

### Safe Delta Pattern — `poller/poller.py`

All derived rate metrics use `_safe_delta()` to handle counter resets:

```python
def _safe_delta(current, previous):
    if previous is None:
        return 0.0
    return max(0.0, float(current - previous))
```

**Always use this pattern** when computing a rate from cumulative counters (GC ms, I/O bytes, rejected counts). Never compute `current - previous` directly.

---

### Adding a New Collected Metric

**Step 1 — Collector** (`poller/collectors/opensearch_api.py` or `system.py`)

Add the raw value to the snapshot dict returned by `collect()`.

**Step 2 — Poller** (`poller/poller.py`)

If it needs a delta/rate: compute it in the polling loop using `_safe_delta`. Add it to `nodes_out` or `host_out`.

**Step 3 — Storage**

Nothing needed — JSONL is schema-free. New fields appear automatically in new records.

**Step 4 — Monitor** (`monitor/poller_history.py`)

If you want to trend the metric historically, add it to the bucket aggregation logic in `poller_history.py`.

**Step 5 — View**

Add a Rich panel or table row in the appropriate view file under `monitor/Opensearch/views/`.

**Step 6 — Docs**

Add to [[05 - Metrics Reference]].

---

### Adding a New Monitor View

1. Create `monitor/Opensearch/views/my_view.py`
2. Implement a `render(console, ...)` function using Rich
3. Import and register in `monitor/menus.py`
4. Use only `client.py` functions for any HTTP

---

## Project File Map (Code-Level)

| File | Responsibility |
|------|---------------|
| `poller/__main__.py` | CLI entry, calls `run()` from `poller.py` |
| `poller/poller.py` | Main loop, orchestrates collectors, computes rates, writes records |
| `poller/config.py` | Reads `.env`, exposes typed constants |
| `poller/collectors/opensearch_api.py` | Fetches `/_nodes/stats`, returns raw snapshot dict |
| `poller/collectors/system.py` | Finds PID via `pgrep`, reads `/proc/<pid>/*` |
| `poller/storage/writer.py` | `append_record()` — JSONL write + daily file rotation |
| `monitor/__main__.py` | Entry point, invokes CLI |
| `monitor/cli.py` | Click command definitions, flag → view routing |
| `monitor/client.py` | All OpenSearch HTTP calls (100% of HTTP layer) |
| `monitor/config.py` | `.env` + thresholds + keyword patterns |
| `monitor/menus.py` | Arrow-key navigation menus with `simple-term-menu` |
| `monitor/metrics_service.py` | `MetricsProvider` — routes between poller JSONL and Prometheus |
| `monitor/poller_history.py` | JSONL file reader + 5-minute bucket aggregator |
| `monitor/utils.py` | `format_bytes()`, `status_symbol()`, `timeframe_to_minutes()` |
| `monitor/Opensearch/views/*.py` | 7 view modules — pure Rich rendering |

---

## Dependencies (`requirements.txt`)

| Package | Purpose |
|---------|---------|
| `opensearch-py` | OpenSearch REST client |
| `rich` | Terminal UI (tables, panels, progress bars) |
| `click` | CLI flag parsing |
| `simple-term-menu` | Arrow-key interactive menus |
| `python-dotenv` | `.env` file loading |
| `urllib3` | HTTP (transitive, SSL warning suppression) |

---

## Known Limitations / TODOs

- [ ] **No alerting notifications** — thresholds are displayed only, no email/Slack/PagerDuty
- [ ] **Single cluster** — no multi-cluster support
- [ ] **No historical FD/IO trending** — host metrics are in JSONL but not surfaced in trends view
- [ ] **Data stream staleness hardcoded** — 60/240 min thresholds are in `data_streams.py`, not `config.py`
- [ ] **No automated tests** — would benefit from mocked client + collector unit tests
- [ ] **Prometheus metrics names may drift** — if the OpenSearch Prometheus exporter changes metric names, the fallback will silently return no data

---

## Testing Locally

```bash
# Verify OpenSearch connection
curl -s -u admin:admin http://localhost:9200/_cluster/health | python3 -m json.tool

# Test poller (single run, verbose)
python -m poller --interval 999 --verbose
# Ctrl+C after first cycle

# Test monitor
python -m monitor --summary

# Inspect JSONL output
tail -1 poller/data/metrics_$(date +%F).jsonl | python3 -m json.tool
```

---

## Related Notes

- [[02 - Architecture]] — component overview
- [[05 - Metrics Reference]] — all metric definitions
- [[07 - OpenSearch API Reference]] — raw API → metric mapping
