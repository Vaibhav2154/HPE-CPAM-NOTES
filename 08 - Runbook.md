# Runbook

Day-to-day operational commands for keeping HPE-Monitor running and interpreting its output.

---

## Installation

```bash
cd /path/to/HPE-Monitor

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Configure
cp .env.example .env
nano .env    # set OPENSEARCH_HOST, OPENSEARCH_PASS, etc.
```

---

## Starting the Poller

```bash
# Activate venv first
source venv/bin/activate

# Standard (API metrics only)
python -m poller --interval 15

# Full metrics (FD + I/O) — requires root or same user as OpenSearch
sudo venv/bin/python -m poller --interval 15

# Fast + verbose (see each record printed)
python -m poller --interval 5 --verbose

# Background with logging
nohup python -m poller --interval 15 > /tmp/poller.log 2>&1 &
echo $! > /tmp/poller.pid
```

### Stopping the Poller

```bash
# If running in foreground: Ctrl+C

# If running in background
kill $(cat /tmp/poller.pid)
```

---

## Starting the Monitor

```bash
source venv/bin/activate

# Interactive menus
python -m monitor

# Direct to OpenSearch menu
python -m monitor --service opensearch

# Quick health snapshot (non-interactive)
python -m monitor --summary

# Auto-refresh every 15 seconds
python -m monitor --summary --watch 15

# Historical — last 6 hours
python -m monitor --timeframe 6h

# Historical — last 4 days (requires Prometheus)
python -m monitor --timeframe 4d
```

---

## Checking Poller Health

```bash
# Is the poller running?
pgrep -fl "python.*poller"

# How many records today?
wc -l poller/data/metrics_$(date +%F).jsonl

# What does the latest record look like?
tail -1 poller/data/metrics_$(date +%F).jsonl | python3 -m json.tool

# Extract just the CPU from each record for today
cat poller/data/metrics_$(date +%F).jsonl | \
  python3 -c "
import sys, json
for line in sys.stdin:
    r = json.loads(line)
    for n, v in r['nodes'].items():
        print(r['timestamp'], n, 'cpu=', v['cpu_pct'], 'heap=', v['heap_pct'])
  "
```

---

## Common Tasks

### Check Cluster Status

```bash
python -m monitor --summary
```

Or via raw API:
```bash
curl -s -u admin:admin http://localhost:9200/_cluster/health | python3 -m json.tool
```

### List Unassigned Shards

```bash
curl -s -u admin:admin "http://localhost:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason" | grep UNASSIGNED
```

### Fix Unassigned Replicas (Single Node)

```bash
# Set replica count to 0 for a specific index
curl -s -u admin:admin -X PUT "http://localhost:9200/<index>/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index": {"number_of_replicas": 0}}'

# For all indices at once
curl -s -u admin:admin -X PUT "http://localhost:9200/*/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index": {"number_of_replicas": 0}}'
```

### Check Thread Pool Rejections

```bash
curl -s -u admin:admin "http://localhost:9200/_cat/thread_pool/write,search?v&h=name,queue,active,rejected"
```

### Clear Disk Watermark (Temporary)

```bash
curl -s -u admin:admin -X PUT "http://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" \
  -d '{
    "transient": {
      "cluster.routing.allocation.disk.threshold_enabled": false
    }
  }'
```

> [!caution] Temporary Only
> Always re-enable disk watermarks after clearing disk space. Disabling permanently risks data loss.

### View Today's JSONL Data

```bash
# Count records
wc -l poller/data/metrics_$(date +%F).jsonl

# View last record pretty-printed
tail -1 poller/data/metrics_$(date +%F).jsonl | jq .

# Find records where cpu > 50%
python3 -c "
import json, sys
for line in open('poller/data/metrics_$(date +%F).jsonl'):
    r = json.loads(line)
    for n, v in r['nodes'].items():
        if v.get('cpu_pct', 0) > 50:
            print(r['timestamp'], n, 'cpu=', v['cpu_pct'])
"
```

---

## Systemd Service (Optional)

Create `/etc/systemd/system/hpe-poller.service`:

```ini
[Unit]
Description=HPE OpenSearch Poller
After=network.target

[Service]
Type=simple
User=opensearch
WorkingDirectory=/path/to/HPE-Monitor
ExecStart=/path/to/HPE-Monitor/venv/bin/python -m poller --interval 15
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable hpe-poller
sudo systemctl start hpe-poller
sudo systemctl status hpe-poller
```

---

## Related Notes

- [[09 - Alert Thresholds & Triage]] — what to do when things go wrong
- [[06 - Configuration]] — environment variables and thresholds
- [[04 - Poller Daemon]] — poller internals
