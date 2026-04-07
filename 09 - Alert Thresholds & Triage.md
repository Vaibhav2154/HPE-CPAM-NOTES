# Alert Thresholds & Triage

This note covers what the monitor alerts on, what it means, and the recommended response for each situation.

---

## Threshold Summary

| Metric | ⚠️ Warn | 🔴 Critical | Notes |
|--------|--------|------------|-------|
| CPU % | > 70% | > 90% | OpenSearch process only |
| JVM Heap % | > 75% | > 90% | Most important metric |
| Disk % | > 80% | > 90% | Index data / filesystem |
| File Descriptors % | > 70% | > 90% | `/proc/<pid>/fd` |
| Write Thread Pool Queue | > 0 | > 50 | Backpressure signal |
| Write Rejected/s | any | any > 0 | Live user-facing errors |
| Data Stream Staleness | > 60 min | > 240 min | Upstream pipeline issue |
| GC Pause Rate (ms/s) | > 50 | > 100 | Memory/GC pressure |
| Unassigned Shards | any (yellow) | - | Depends on cluster size |

> [!note] System RAM
> **System RAM is informational only.** OpenSearch uses spare RAM as the page cache. High OS RAM usage is expected and healthy. Only JVM Heap triggers alarms.

---

## Triage Flowchart

```
Is cluster status RED?
  └─► Yes → Check unassigned shards first (data unavailable!)
             └─► Likely a disk watermark breach or node failure

Is cluster YELLOW?
  └─► Single-node setup → Replica shards can't be placed (normal)
                          Fix: set number_of_replicas: 0
  └─► Multi-node setup  → A node is down or shard relocation in progress

Is CPU > 90%?
  └─► Check: large aggregations, bulk indexing surge, merge storms
  └─► Look at thread pool write.queue — if non-zero, indexing is queuing

Is Heap > 90%?
  └─► Check GC pause rate — if > 100 ms/s, GC is thrashing
  └─► Immediate: reduce indexing rate or increase JVM heap (-Xmx)
  └─► Near-term: check for large segment merges consuming heap

Is Disk > 90%?
  └─► URGENT — disk watermarks may trigger read-only mode
  └─► Delete old indices or snapshots
  └─► Temporarily disable watermark checks (see Runbook)

Write Rejected/s > 0?
  └─► IMMEDIATE — users are getting write failures now
  └─► Back off ingestion rate
  └─► Check thread pool queue size
  └─► Consider adding nodes or increasing thread pool size

Data Stream stale > 240 min?
  └─► OpenSearch is healthy — issue is UPSTREAM
  └─► Check: Logstash, Kafka, Beats, Fluentd pipeline
  └─► maximum_timestamp shows when the last document arrived
```

---

## Alert Details

### 🟡 HIGH CPU (70–90%)

**Possible causes:**
- Bulk indexing surge
- Expensive query (large aggregations, wildcard searches)
- Segment merge storm after bulk load

**Triage:**
1. Check `thread_pool.write.queue` — is indexing queuing?
2. Check `thread_pool.search.queue` — is search queuing?
3. Look for slow logs: `GET /_nodes/stats/indices/search` for slow queries

**Response:**
- Throttle ingestion rate if write queue is growing
- Cancel rogue queries if search queue is growing

---

### 🔴 CRITICAL CPU (> 90%)

**Expect:** Latency spikes, timeouts, thread pool rejections.

**Immediate actions:**
1. Identify and kill expensive queries if possible
2. Pause bulk indexing temporarily
3. Look at GC pause rate — if > 100, heap issue may be causing CPU spike

---

### 🟡 HIGH HEAP (75–90%)

**Why it matters:** JVM GC runs more frequently at high heap, causing latency spikes and eventually OOM.

**Response:**
- Watch `gc_pause_rate_ms_per_s` — it will rise before heap becomes critical
- Force merge old indices to reduce segment count (frees heap)
- Consider reducing `indices.memory.index_buffer_size` temporarily

---

### 🔴 CRITICAL HEAP (> 90%)

**Expect:** Frequent long GC pauses, circuit breaker trips, possible OOM crash.

**Immediate actions:**
1. Reduce indexing rate (relieve buffer pressure)
2. Force merge heavily segmented indices to free heap
3. If `gc_pause_rate_ms_per_s > 200`, restart opensearch-service with increased `-Xmx`

---

### 🔴 CRITICAL DISK (> 90%)

**Why it matters:** OpenSearch has three disk watermarks:
| Watermark | Default | Effect |
|-----------|---------|--------|
| High (warn) | 90% | Stop allocating new shards to this node |
| Flood stage | 95% | Set all indices to read-only |

**Immediate actions:**
1. Delete unused indices: `DELETE /<index_name>`
2. Take snapshots and delete old data
3. Temporarily disable watermark (see [[08 - Runbook]])
4. Add disk capacity

---

### 🔴 THREAD POOL REJECTIONS

**Write rejections** → indexing requests dropped, data loss possible.  
**Search rejections** → queries returning errors to users.

**Immediate actions:**
1. Back off producers immediately (Logstash, Beats, application)
2. Allow queue to drain — watch `thread_pool.write.queue`
3. Check CPU and Heap — rejections are often a downstream symptom

**Root causes:**
- CPU saturated → workers slow → queue fills → rejections
- Heap high → GC pauses → workers stall → queue fills → rejections
- Ingestion spike exceeding cluster capacity

---

### 🟡 DATA STREAM STALE (> 60 min)

OpenSearch is healthy — no data is arriving from the upstream pipeline.

**Check upstream:**
```bash
# Check Logstash
systemctl status logstash

# Check Beats agent
systemctl status filebeat metricbeat

# Check Kafka consumer lag (if using Kafka)
kafka-consumer-groups.sh --describe --group <group>
```

---

### 🟡 UNASSIGNED SHARDS (single-node)

This is **normal for a single-node cluster** — replica shards cannot be placed without a second node.

**Fix:**
```bash
curl -u admin:admin -X PUT "http://localhost:9200/*/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index": {"number_of_replicas": 0}}'
```

---

## Root Cause Pattern Reference

From `monitor/config.py` — these strings in OpenSearch logs map to labels:

| Log Pattern | Root Cause Label |
|-------------|-----------------|
| `OutOfMemoryError` | 🔴 JVM OOM — heap exhausted |
| `GC overhead limit` | 🟠 GC overhead — excessive garbage collection |
| `disk usage exceeded` | 🔴 Disk full / watermark breached |
| `circuit_breaking_exception` | 🟠 Circuit breaker tripped — memory pressure |
| `flood stage` | 🔴 Disk flood-stage — index set read-only |
| `high disk watermark` | 🟡 High disk watermark crossed |
| `rejected execution` | 🟠 Thread pool rejection — queue full |
| `bulk rejected` | 🟠 Bulk indexing rejected — backpressure |
| `timeout` | 🟡 Operation timeout |
| `unassigned` | 🟡 Unassigned shards detected |

---

## Related Notes

- [[08 - Runbook]] — how to execute triage commands
- [[05 - Metrics Reference]] — what each metric measures
- [[06 - Configuration]] — how to tune thresholds
