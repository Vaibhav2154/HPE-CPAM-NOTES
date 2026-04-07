# HPE-Monitor Vault 🏠

> [!tip] Quick Links
> - [[01 - Project Overview]] — what this project does and why
> - [[02 - Architecture]] — component map and data flow
> - [[03 - Monitor CLI]] — the interactive terminal UI
> - [[04 - Poller Daemon]] — the background metrics collector
> - [[05 - Metrics Reference]] — every metric, explained
> - [[06 - Configuration]] — .env, thresholds, and flags
> - [[07 - OpenSearch API Reference]] — all endpoints used
> - [[08 - Runbook]] — day-to-day ops commands
> - [[09 - Alert Thresholds & Triage]] — when to worry and what to do
> - [[10 - Development Notes]] — code structure, patterns, how to extend

---

## Project at a Glance

**HPE-Monitor** is a self-contained OpenSearch observability stack consisting of two cooperating Python processes:

| Component | Role |
|-----------|------|
| `poller/` | Background daemon — polls OpenSearch API + Linux `/proc` every N seconds and appends JSONL snapshots |
| `monitor/` | Interactive CLI — reads live API + poller JSONL history to render 7 terminal views |

```
┌─────────────────────┐       HTTP        ┌─────────────────┐
│   OpenSearch :9200  │◄──────────────────│  poller daemon  │──► poller/data/metrics_YYYY-MM-DD.jsonl
└─────────────────────┘                   └─────────────────┘
         │                                        │
         │ HTTP                              JSONL files
         ▼                                        ▼
┌─────────────────────────────────────────────────────────┐
│                  monitor CLI (Rich UI)                   │
│  Quick Summary │ Trends │ Health │ Indices │ Nodes │ ... │
└─────────────────────────────────────────────────────────┘
```

---

## Status

- [x] Poller daemon — collecting all metrics ✅ 2026-04-06
- [x] Monitor CLI — all 7 views implemented
- [x] Historical trends — poller-first, Prometheus fallback
- [ ] Alerting / notifications
- [ ] Multi-cluster support

---
*Last updated: 2026-04-06*
