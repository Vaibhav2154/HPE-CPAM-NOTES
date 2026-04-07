# Project Overview

**HPE-Monitor** is a Python-based monitoring system built for OpenSearch clusters, developed as part of an HPE internship/project. It replaces the need for heavy dashboards (Grafana + Prometheus) with a lightweight, self-contained toolset that runs directly from a terminal.

---

## Problem Statement

OpenSearch clusters can silently degrade — CPU spikes, heap pressure, shard failures, and thread pool rejections all cause user-visible slowdowns before anyone notices. Standard monitoring tools require significant infrastructure (Prometheus, Grafana, alerting pipelines). This project provides:

- **Immediate visibility** without external tools
- **Historical context** via a minimal local JSONL store
- **Actionable diagnostics** — not just numbers, but explanations

---

## Design Goals

1. **Zero external dependencies for monitoring** — just Python + OpenSearch's own REST API
2. **Two-tier architecture** — passive collector (`poller`) + interactive viewer (`monitor`)
3. **Poller-first, Prometheus fallback** — use local data when available, fall back gracefully
4. **Small but high-signal metrics** — only collect what matters for OpenSearch ops health
5. **Plain-English diagnostics** — bottleneck summaries instead of raw counters

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.11+ |
| Terminal UI | [Rich](https://github.com/Textualize/rich) |
| CLI parsing | [Click](https://click.palletsprojects.com/) |
| Interactive menus | [simple-term-menu](https://github.com/IngoMeyer441/simple-term-menu) |
| OpenSearch client | `opensearch-py` |
| Config | `.env` via `python-dotenv` |
| Storage | JSONL flat files (daily rotation) |
| History fallback | Prometheus HTTP API |

---

## Repository Layout

```
HPE-Monitor/
├── .env                    # Secrets (not committed)
├── .env.example            # Template
├── requirements.txt        # Python dependencies
├── opensearch.py           # (legacy/scratch file)
├── monitor/                # Interactive CLI tool
│   ├── __main__.py
│   ├── cli.py
│   ├── client.py
│   ├── config.py
│   ├── menus.py
│   ├── metrics_service.py
│   ├── poller_history.py
│   ├── utils.py
│   └── Opensearch/
│       └── views/          # 7 terminal view modules
├── poller/                 # Background metrics daemon
│   ├── __main__.py
│   ├── poller.py
│   ├── config.py
│   ├── collectors/
│   │   ├── opensearch_api.py
│   │   └── system.py
│   ├── storage/
│   ├── data/               # JSONL output (daily files)
│   └── docs/
└── obsidian/               # 📓 This vault
```

---

## Related Notes

- [[02 - Architecture]] — component interaction diagram
- [[03 - Monitor CLI]] — CLI flags and views
- [[04 - Poller Daemon]] — how the poller works
- [[06 - Configuration]] — connection and threshold setup
