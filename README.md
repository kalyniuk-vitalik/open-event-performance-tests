# Open Event API - Performance Tests

[![JMeter CI](https://github.com/kalyniuk-vitalik/open-event-performance-tests/actions/workflows/jmeter.yml/badge.svg)](https://github.com/kalyniuk-vitalik/open-event-performance-tests/actions/workflows/jmeter.yml)

Performance testing suite for [Open Event API](https://github.com/fossasia/open-event-server) built with Apache JMeter. Full CI/CD pipeline with real-time metrics visualization: locally via Jenkins and in the cloud via GitHub Actions + Google Cloud Platform.

---

## Stack

| Tool | Role |
|------|------|
| Apache JMeter 5.5 | Load test execution, metrics via Backend Listener |
| InfluxDB 1.x | Time-series storage for JMeter and system metrics |
| Grafana | Real-time dashboards: Load Test + Comparison + Server Monitoring |
| Telegraf | System-level metrics agent (CPU/RAM/Disk/Network) |
| Jenkins | Local CI: parameterized builds, HTML reports, Grafana links |
| GitHub Actions | Cloud CI: automated test runs on push/manual trigger |
| Google Cloud Platform | Hosts InfluxDB + Grafana + Telegraf for cloud pipeline |

---

## Architecture

**Local Pipeline**
```
Jenkins
  └── JMeter ──► InfluxDB (db: jmeter) ──► Grafana
Telegraf ────► InfluxDB (db: telegraf) ──► Grafana
```

**Cloud Pipeline**
```
git push/workflow_dispatch
  └── GitHub Actions
        └── JMeter ──► GCP VM: InfluxDB (db: jmeter_gha) ──► Grafana
              GCP VM: Telegraf ────────────────────────────► Grafana
```

> Local and cloud pipelines are fully isolated.  
> Same `.jmx` files, two environments, via `INFLUX_DB` parameter with default fallback.

---

## Test Scenarios

| File | Scenario | Key Parameters |
|------|----------|----------------|
| `api_system_level_flow.jmx` | Full user journey: login → create event → buy ticket → list events | USERS, RAMPUP, DURATION |
| `api_separate_endpoints.jmx` | Isolated load per endpoint with individual thread control | POST_LOGIN, GET_USERS, POST_EVENTS, POST_TICKETS, GET_EVENTS |
| `api_events_pagesize.jmx` | Pagination performance across different page sizes | USERS, PAGES |
| `api_speakers.jmx` | Speaker endpoint load with session-based concurrency | USERS, SESSION_COUNT |

All tests are fully parameterized via `${__P(VARIABLE, default)}` - no hardcoded values.

---

## Performance Results

📊 [View Results Summary](PERFORMANCE_RESULTS.md)  
📄 [Full Test Report (PDF)](docs/Performance_Test_Report_Open_Event_API.pdf)

**Key findings:**
- System capacity: **5 RPS at 5 threads** - memory leak identified as main bottleneck
- Endpoint gap: **4.6x** between fastest (POST /v1/events, 9.2 RPS) and slowest (GET /v1/events, 2 RPS)
- Load test: stable at **~2 RPS, 0% errors** over 1 hour

---

## Grafana Dashboards

Real-time metrics visualized during test execution across 3 dashboards.

| Dashboard | Purpose | Data Source |
|-----------|---------|-------------|
| JMeter Load Test Monitoring | Active threads, RPS, response time, error rate | InfluxDB `jmeter` |
| JMeter Comparison Dashboard | Side-by-side comparison across test runs | InfluxDB `jmeter` |
| Linux System Overview | CPU/RAM/Disk/Network during test | InfluxDB `telegraf` |

Dashboard screenshots from live test runs:

| Screenshot | Description |
|-----------|-------------|
| [System Capacity Test](dashboards/screenshots/load-test-system-capacity.png) | Comfort zone → degradation → capacity point |
| [Separate Endpoints](dashboards/screenshots/load-test-separate-endpoints.png) | Per-endpoint capacity comparison |
| [Load Test (1hr)](dashboards/screenshots/load-test-load-test.png) | Stable throughput over 1 hour |
| [Server Monitoring](dashboards/screenshots/server-monitoring.png) | CPU/RAM during test execution |
| [Comparison Dashboard](dashboards/screenshots/comparison-dashboard.png) | Two runs side-by-side |

> Dashboards are not exported as JSON - they contain local data source references. To reproduce, import community dashboards by ID (1152, 5496, 928) and configure data sources per [docs/local-setup.md](docs/local-setup.md).

---

## CI/CD

### GitHub Actions (Cloud)
Runs all 4 scenarios sequentially on every push to `main`. Results uploaded as artifacts.  
→ [View workflow runs](../../actions/workflows/jmeter.yml)

### Jenkins (Local)
4 parameterized jobs, one per scenario. Each build archives `results.csv`, `jmeter.log`, HTML report and generates a Grafana link with exact test time range in the build description.

| Screenshot | Description |
|-----------|-------------|
| [Jobs Overview](docs/screenshots/jenkins/jenkins-jobs-overview.png) | 4 jobs, all green |
| [Build History](docs/screenshots/jenkins/jenkins-build-history.png) | OpenEvent_SystemLevelFlow - 26 builds |
| [Parameterized Build](docs/screenshots/jenkins/jenkins-parameterized-build.png) | Per-endpoint thread control |
| [Console Output](docs/screenshots/jenkins/jenkins-console-output.png) | JMeter execution with parameters |

---

## Quick Start

### Local (Jenkins)
→ [docs/local-setup.md](docs/local-setup.md)

### Cloud (GitHub Actions + GCP)
→ [docs/cloud-setup.md](docs/cloud-setup.md)

### Manual CLI run
```bash
/path/to/jmeter.sh -n \
  -t test-plans/api_system_level_flow.jmx \
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
│       └── jmeter.yml                  # Cloud CI pipeline (GitHub Actions)
├── test-plans/
│   ├── api_system_level_flow.jmx       # Full user journey + load test
│   ├── api_separate_endpoints.jmx      # Isolated capacity per endpoint
│   ├── api_events_pagesize.jmx         # page[size] parameter impact
│   └── api_speakers.jmx                # Sessions count impact
├── dashboards/
│   ├── screenshots/                    # Grafana dashboard screenshots
│   └── README.md                       # Dashboard overview
├── docs/
│   ├── screenshots/
│   │   └── jenkins/                    # Jenkins CI screenshots
│   ├── local-setup.md                  # Local pipeline setup
│   ├── cloud-setup.md                  # Cloud pipeline setup
│   └── Performance_Test_Report_Open_Event_API.pdf
├── .gitignore
├── PERFORMANCE_RESULTS.md              # Test results summary
└── README.md
```

---

## Key Design Decisions

**Why InfluxDB 1.x?**  
JMeter Backend Listener uses the v1 write API (`/write?db=...`). InfluxDB 2.x requires Flux queries - incompatible with standard Grafana JMeter dashboards.

**Why sequential tests in GitHub Actions (not matrix)?**  
Parallel execution puts concurrent load on the API server, making results unreliable. Sequential runs give clean, isolated measurements per scenario.

**Why separate Jenkins jobs per JMX?**  
Independent build history, separate Grafana time-range links per test, and the ability to run specific scenarios without triggering the full suite.

**Why `backend_metrics_window_mode=timed`?**  
Default `fixed` mode accumulates metrics from test start - percentiles become increasingly smoothed and diverge from HTML Aggregate Report. `timed` mode recalculates every 5s window for accurate real-time percentiles.

**Why SSH instead of HTTPS for Git?**  
macOS Keychain stores old HTTPS credentials and returns 403. SSH key bypasses this completely.

**Why two separate pipelines (Jenkins + GitHub Actions)?**  
GitHub Actions runs smoke tests automatically on every push for fast CI feedback. Jenkins is used for full test sessions with detailed analytics in Grafana - load generator and target are on the same network, giving more stable and reliable results.
