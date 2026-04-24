# Advanced Network Scanner and Attack Surface Mapper

An offensive security reconnaissance platform that automates network discovery, service enumeration, vulnerability correlation, and attack surface visualization.

> **Disclaimer:** This tool is intended for **authorized security assessments only**. Always obtain written permission before scanning any targets. Unauthorized scanning is illegal and unethical.

> **Note:** Source code is not published in this repository. This repo documents the design, capabilities, and architecture of the project.

## Overview

Advanced Network Scanner and Attack Surface Mapper (ANS-ASM) is a modular reconnaissance framework designed for authorized penetration testing, red-team engagements, and continuous attack-surface monitoring. It combines passive OSINT collection with active network probing and persists every finding into a queryable history so operators can diff an asset's exposure across time.

## Features

- **Modular Architecture** — Plugin-based scan modules with auto-discovery from `modules/passive/` and `modules/active/`.
- **Async Engine** — Non-blocking scan orchestration with concurrent module execution via `asyncio.Semaphore`.
- **Scan Profiles** — Stealth, balanced, and aggressive profiles tuned for different engagement needs.
- **Scope Management** — Strict target validation, CIDR expansion, exclusion lists, and private-range controls so scans can't leak beyond authorization.
- **Rich CLI** — Typer + Rich terminal UI with tables, progress bars, and panels.
- **REST API** — FastAPI service exposing scan control and results, with an HTML dashboard for real-time visualization.
- **Database Backed** — SQLAlchemy 2.0 async ORM (SQLite in dev, PostgreSQL in prod) with Alembic migrations and full scan history.
- **Export Support** — JSON and Markdown report generation.
- **Authorization Gate** — Mandatory written-authorization confirmation before any active scan.
- **Event Bus** — Pub/sub lifecycle events for integration with alerting, ticketing, and SIEM pipelines.

## Capabilities

### Passive reconnaissance
- WHOIS lookup and registrar metadata correlation.
- DNS enumeration (A, AAAA, MX, NS, TXT, SOA, CAA).
- Subdomain discovery via certificate transparency and public sources.
- Geo/ASN enrichment for discovered IPs.

### Active reconnaissance
- TCP / UDP port scanning with configurable port sets per profile.
- Service fingerprinting and banner grabbing.
- Lightweight vulnerability correlation against known CVE signatures.

### Reporting
- JSON export for pipeline ingestion.
- Markdown reports for human consumption.
- Scan diff across historical runs.

## Technology Stack

| Layer | Technology |
|-------|-----------|
| CLI framework | Typer |
| Terminal UI | Rich |
| REST API | FastAPI |
| ORM | SQLAlchemy 2.0 (async) |
| Migrations | Alembic |
| Dev database | SQLite via aiosqlite |
| Prod database | PostgreSQL |
| HTTP client | httpx |
| Validation | Pydantic 2 |
| Logging | loguru |
| Config | TOML (stdlib `tomllib`) |
| Runtime | Python 3.11+ |
| Packaging | Hatchling |
| Container | Docker / docker compose |

## Project Layout

```
src/
├── core/        # Engine, module registry, scope manager, config, event bus
├── models/      # SQLAlchemy ORM models (scan_jobs, assets, ports, vulns, subdomains)
├── modules/     # Scan modules — passive/ and active/ subpackages, auto-discovered
├── cli/         # Typer CLI application (scan, modules, export commands)
├── api/         # FastAPI REST API + HTML dashboard
└── utils/       # Logging setup, input validation helpers

alembic/         # Database migrations
config/          # Default config + scan profiles (stealth / balanced / aggressive)
docs/            # Architecture and module-development docs
tests/           # Pytest suite
```

## Scan Lifecycle

```
PENDING ──▶ RUNNING ──▶ COMPLETED
   │           │
   │           ├──▶ PAUSED ──▶ RUNNING (resume)
   │           │
   │           └──▶ FAILED
   │
   └──▶ CANCELLED
```

State transitions are atomic — implemented as conditional SQL `UPDATE`s that check the current state before applying the new one, so two orchestrators can never race a scan into an inconsistent state.

## Architecture

A high-level diagram and component description is in [ARCHITECTURE.md](ARCHITECTURE.md).

## Status

Private development build. Source code is withheld because the running deployment contains API keys and environment secrets that have not been rotated. This repository serves as the public design record.

## Contributors

- [BHARGAV RAJ DUTTA (@dev0558)](https://github.com/dev0558)
- [@techtrail42](https://github.com/techtrail42)

## License

MIT License — applies to design, documentation, and any source that is published separately.
