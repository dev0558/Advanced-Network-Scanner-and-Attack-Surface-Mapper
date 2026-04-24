# Architecture

## System Components

```
+----------------------------------------------------------+
|                        CLI Layer                          |
|  Typer + Rich                                            |
|  Commands: scan, modules, export                         |
+----------------------------------------------------------+
                              |
+----------------------------------------------------------+
|                     REST API Layer                        |
|  FastAPI + HTML dashboard                                |
|  Endpoints: /scans, /modules, /assets, /export           |
+-----------------------------+----------------------------+
                              |
                              v
+----------------------------------------------------------+
|              Scan Orchestration Engine                    |
|                                                          |
|  - Lifecycle: PENDING -> RUNNING -> COMPLETED/FAILED     |
|  - Concurrency control (asyncio.Semaphore)               |
|  - State persistence at every transition                 |
|  - Scope enforcement before every module                 |
+-------+------------------+------------------+-----------+
        |                  |                  |
        v                  v                  v
+---------------+  +---------------+  +----------------+
| Module        |  | Scope         |  | Event Bus      |
| Registry      |  | Manager       |  |                |
|               |  |               |  | - Pub/sub      |
| - Auto-       |  | - IP/CIDR     |  | - Sync/async   |
|   discovery   |  | - Domain      |  |   handlers     |
| - importlib   |  | - Exclusions  |  | - Lifecycle    |
| - pkgutil     |  | - Private     |  |   events       |
+-------+-------+  |   range ctrl  |  +----------------+
        |          +---------------+
        v
+----------------------------------------------------------+
|                    Scan Modules                           |
|                                                          |
|  Passive:              Active:                           |
|  - WHOIS Lookup        - Port Scanner                    |
|  - DNS Enum            - Service Enum                    |
|  - Subdomain Enum      - Vuln Correlator                 |
|  - Geo/ASN Enrich                                        |
+-----------------------------+----------------------------+
                              |
                              v
+----------------------------------------------------------+
|                   Persistence Layer                       |
|  SQLAlchemy 2.0 async + aiosqlite (dev) / PostgreSQL      |
|                                                          |
|  Tables: scan_jobs, assets, ports, vulnerabilities,      |
|          subdomains, scan_module_records                  |
+----------------------------------------------------------+
```

## Data Flow

1. **Input** — Operator supplies targets, profile, and options via CLI or REST API.
2. **Scope validation** — Targets parsed, CIDRs expanded, exclusions applied, private ranges gated.
3. **Authorization gate** — Operator confirms written authorization; gate is bypassable only via a signed config flag for CI.
4. **Scan creation** — `ScanJob` row persisted with `PENDING` status.
5. **Module selection** — Profile determines which modules run and with what budget.
6. **Module execution** — Engine runs modules with an `asyncio.Semaphore`-bounded concurrency pool.
7. **Result storage** — Modules persist findings as `Asset`, `Port`, `Vulnerability`, `Subdomain` rows linked to the scan.
8. **State updates** — Engine transitions `ScanJob` through lifecycle states atomically.
9. **Event emission** — `EventBus` publishes lifecycle events to subscribed handlers (logging, alerting, webhook forwarding).
10. **Export** — Results exported as JSON (pipeline-friendly) or Markdown (human-friendly).

## Module Contract

Every scan module extends `BaseModule` and implements:

- `metadata` — Name, category (`passive` / `active`), version, declared dependencies.
- `initialize(config)` — Validate prerequisites, check tool availability, return a readiness flag.
- `run(targets, options)` — Perform reconnaissance, yield findings.
- `cleanup()` — Release sockets, temp files, child processes.

Modules are auto-discovered at startup via `pkgutil.walk_packages` + `importlib.import_module` — dropping a file into `modules/passive/` or `modules/active/` registers it with zero wiring.

## Scope Manager

The scope manager is the last line of defense against out-of-bounds scanning. It:

- Parses inputs as IPs, CIDRs, or domains.
- Expands CIDRs into discrete host lists with configurable size caps.
- Enforces exclusion lists.
- Blocks private ranges (RFC1918, loopback, link-local) unless explicitly allowed.
- Is consulted before **every** module invocation, not just at scan start — so a module that resolves a domain mid-scan can't pivot onto an IP outside the agreed scope.

## State Machine

```
PENDING ──> RUNNING ──> COMPLETED
   |           |
   |           ├──> PAUSED ──> RUNNING (resume)
   |           |
   |           └──> FAILED
   |
   └──> CANCELLED
```

Transitions are implemented as conditional SQL `UPDATE`s that check the current state in the `WHERE` clause. If the row is not in the expected prior state, the update affects zero rows and the engine aborts the transition — making concurrent orchestrators safe.

## Event Bus

A lightweight in-process pub/sub. Publishers emit lifecycle events (`scan.started`, `module.completed`, `vulnerability.found`, `scan.failed`); subscribers register sync or async handlers. Used internally for logging and externally for webhook/alert fan-out.

## Configuration

Layered TOML config with explicit precedence: `config/default.toml` -> scan-profile overrides (`config/scan_profiles/*.toml`) -> environment variables -> CLI flags. Secrets live exclusively in `.env` and are surfaced through Pydantic settings — never logged, never serialized into reports.
