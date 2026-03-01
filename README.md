# Open Event API — Performance Tests

[![JMeter CI](https://github.com/kalyniuk-vitalik/open-event-performance-tests/actions/workflows/jmeter.yml/badge.svg)](https://github.com/kalyniuk-vitalik/open-event-performance-tests/actions/workflows/jmeter.yml)

Performance testing suite for [Open Event API](https://github.com/fossasia/open-event-server) built with Apache JMeter. Full CI/CD pipeline with real-time metrics visualization — locally via Jenkins and in the cloud via GitHub Actions + Google Cloud Platform.

---

## Stack

| Tool | Role |
|------|------|
| Apache JMeter 5.5 | Load test execution, metrics via Backend Listener |
| InfluxDB 1.x | Time-series storage for JMeter and system metrics |
| Grafana | Real-time dashboards — Load Test + Comparison + Server Monitoring |
| Telegraf | System-level metrics agent (CPU / RAM / Disk / Network) |
| Jenkins | Local CI: parameterized builds, HTML reports, Grafana links |
| GitHub Actions | Cloud CI: automated test runs on push / manual trigger |
| Google Cloud Platform | Hosts InfluxDB + Grafana + Telegraf for cloud pipeline |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    LOCAL PIPELINE                        │
│                                                         │
│  Jenkins (localhost:8080)                               │
│    └─ clones .jmx from GitHub (SSH)                    │
│    └─ runs JMeter with parameters                       │
│         ├─ Backend Listener ──► InfluxDB:jmeter         │
│         │                           └─► Grafana :3000   │
│         └─ artifacts: results.csv, jmeter.log, report/  │
│                                                         │
│  Telegraf ──────────────────► InfluxDB:telegraf         │
│                                    └─► Grafana :3000    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    CLOUD PIPELINE                        │
│                                                         │
│  git push / workflow_dispatch                           │
│    └─ GitHub Actions (ubuntu-latest)                    │
│         └─ installs JMeter + plugins                    │
│         └─ runs 4 tests sequentially                    │
│              ├─ Backend Listener ──► InfluxDB:jmeter_gha│
│              │              (GCP VM) └─► Grafana :3000  │
│              └─ uploads artifacts to GitHub             │
│                                                         │
│  GCP VM (e2-micro, us-central1)                        │
│    ├─ InfluxDB  — DB: jmeter_gha                       │
│    ├─ Grafana   — dashboards for cloud runs            │
│    └─ Telegraf  — VM system metrics                    │
└─────────────────────────────────────────────────────────┘
```

> Local and cloud pipelines are fully isolated.  
> Same `.jmx` files — two environments — via `INFLUX_DB` parameter with default fallback.

---

## Test Scenarios

| File | Scenario | Key Parameters |
|-----------|-------------|
| `api_system_level_flow.jmx` | Full user journey: login → create event → buy ticket → list events | USERS, RAMPUP, DURATION |
| `api_separate_endpoints.jmx` | Isolated load per endpoint with individual thread control | POST_LOGIN, GET_USERS, POST_EVENTS, POST_TICKETS, GET_EVENTS |
| `api_events_pagesize.jmx` | Pagination performance across different page sizes | USERS, PAGES |
| `api_speakers.jmx` | Speaker endpoint load with session-based concurrency | USERS, SESSION_COUNT |

All tests are fully parameterized via `${__P(VARIABLE, default)}` — no hardcoded values.

---

## Quick Start

### Local (Jenkins)
→ Overview: [docs/local-setup.md](docs/local-setup.md)  
→ Configuration reference: [docs/local-checklist.md](docs/local-checklist.md)

### Cloud (GitHub Actions + GCP)
→ Overview: [docs/cloud-setup.md](docs/cloud-setup.md)  
→ Configuration reference: [docs/cloud-checklist.md](docs/cloud-checklist.md)

### Manual CLI run
```bash
/path/to/jmeter.sh -n \
  -t open-event-api/api_system_level_flow.jmx \
  -l results.csv -e -o report \
  -JUSERS=10 -JRAMPUP=60 -JDURATION=300 \
  -JTEST_TITLE="smoke_test"
```

---

## Repository Structure

```
open-event-performance-tests/
├── .github/
│   └── workflows/
│       └── jmeter.yml          # Cloud CI pipeline
├── open-event-api/
│   ├── api_system_level_flow.jmx
│   ├── api_separate_endpoints.jmx
│   ├── api_events_pagesize.jmx
│   └── api_speakers.jmx
├── dashboards/
│   ├── jmeter-load-test.json          # JMeter Load Test Monitoring
│   ├── jmeter-comparison.json         # JMeter Comparison Dashboard
│   ├── server-monitoring-local.json   # Linux System Overview (local)
│   └── server-monitoring-cloud.json   # Telegraf: System Dashboard (cloud)
├── docs/
│   ├── local-setup.md          # Local pipeline overview
│   ├── local-checklist.md      # Local configuration reference
│   ├── cloud-setup.md          # Cloud pipeline overview
│   └── cloud-checklist.md      # Cloud configuration reference
├── .gitignore
└── README.md
```

---

## Grafana Dashboards

Pre-configured dashboards are stored in [`/dashboards`](dashboards/) as JSON exports — ready to import.

| File | Dashboard | Data Source |
|-----------|-------------|
| `jmeter-load-test.json` | JMeter Load Test Monitoring | jmeter |
| `jmeter-comparison.json` | JMeter Comparison Dashboard | jmeter |
| `server-monitoring-local.json` | Linux System Overview (local) | telegraf |
| `server-monitoring-cloud.json` | Telegraf: System Dashboard (cloud) | telegraf |

Import: Dashboards → Import → Upload JSON file.

---

## Key Design Decisions

**Why InfluxDB 1.x?**  
JMeter Backend Listener uses the v1 write API (`/write?db=...`). InfluxDB 2.x requires Flux queries — incompatible with standard Grafana JMeter dashboards.

**Why sequential tests in GitHub Actions (not matrix)?**  
Parallel execution puts concurrent load on the API server, making results unreliable. Sequential runs give clean, isolated measurements per scenario.

**Why separate Jenkins jobs per JMX?**  
Independent build history, separate Grafana time-range links per test, and the ability to run specific scenarios without triggering the full suite.

**Why `backend_metrics_window_mode=timed`?**  
Default `fixed` mode accumulates metrics from test start — percentiles become increasingly smoothed and diverge from HTML Aggregate Report. `timed` mode recalculates every 5s window for accurate real-time percentiles.

**Why SSH instead of HTTPS for Git?**  
macOS Keychain stores old HTTPS credentials and returns 403. SSH key bypasses this completely.
