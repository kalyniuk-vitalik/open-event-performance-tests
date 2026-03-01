# Local Infrastructure Setup

Local pipeline runs JMeter tests through Jenkins CI with real-time metrics streaming to InfluxDB and Grafana.

## Components

| Tool | Version | Purpose |
|-----------|-------------|
| Apache JMeter | 5.5 | Test execution |
| InfluxDB | 1.x | Metrics storage |
| Grafana | 12.x | Dashboards |
| Telegraf | 1.x | System metrics agent |
| Jenkins | LTS | CI orchestration |

> InfluxDB 1.x is required. Version 2.x uses Flux — incompatible with JMeter Backend Listener and standard Grafana dashboards.

---

## How it works

Jenkins clones this repository via SSH, executes the selected `.jmx` file with build parameters, and archives results. During the test, JMeter streams metrics to InfluxDB via Backend Listener. Grafana reads from InfluxDB and displays live dashboards. Telegraf independently collects system metrics into a separate `telegraf` database.

```
Jenkins ──► JMeter ──► InfluxDB (db: jmeter)   ──► Grafana
Telegraf ──────────► InfluxDB (db: telegraf) ──► Grafana
```

---

## Directory Layout

Tools are stored outside the Jenkins workspace to avoid being wiped between builds. Jenkins build commands use absolute paths.

```
~/Tools/
  ├── apache-jmeter-5.5/
  ├── influxdb-1.x/
  ├── grafana-12.x/
  └── telegraf/
```

---

## InfluxDB

Start: `./influxd` (port 8086)

Two databases are required:

```sql
CREATE DATABASE jmeter    -- JMeter Backend Listener metrics
CREATE DATABASE telegraf  -- Telegraf system metrics
```

---

## Telegraf

Configuration (`telegraf.conf`):

```toml
[[outputs.influxdb]]
  urls     = ["http://127.0.0.1:8086"]
  database = "telegraf"

[[inputs.cpu]]
  percpu   = true
  totalcpu = true
[[inputs.mem]]
[[inputs.disk]]
[[inputs.net]]
[[inputs.system]]
```

Start: `./telegraf --config telegraf.conf`

---

## Grafana

Start: `./bin/grafana-server` (port 3000)

**Data sources:**

| Name | Query Language | URL | Database |
|-----------|-------------|----------|
| jmeter | InfluxQL | http://localhost:8086 | jmeter |
| telegraf | InfluxQL | http://localhost:8086 | telegraf |

**Dashboards** — import from [`/dashboards`](../dashboards/):

| Dashboard | Data Source |
|-----------|-------------|
| JMeter Load Test Monitoring | jmeter |
| JMeter Comparison Dashboard | jmeter |
| Linux System Overview | telegraf |

**Anonymous access** is enabled in `grafana.ini` so Jenkins build description links open dashboards directly without login:

```ini
[auth.anonymous]
enabled  = true
org_name = Main Org.
org_role = Viewer
```

---

## JMeter — Backend Listener

Each `.jmx` file contains a Backend Listener configured to write to InfluxDB:

| Parameter | Value |
|-----------|-------|
| influxdbUrl | `http://localhost:8086/write?db=jmeter` |
| summaryOnly | `false` |
| percentiles | `90;95;99` |

Each scenario uses a unique `application` field to allow filtering in Grafana:

| File | application |
|------|-------------|
| api_system_level_flow.jmx | system_level_flow |
| api_separate_endpoints.jmx | separate_endpoints |
| api_events_pagesize.jmx | events_pagesize |
| api_speakers.jmx | speakers_test |

`backend_metrics_window_mode=timed` is set in `jmeter.properties`. Without this, percentiles in Grafana diverge from the HTML Aggregate Report because the default `fixed` mode smooths metrics from test start rather than recalculating per time window.

---

## Jenkins

Start: `java -jar jenkins.war --httpPort=8080`

**Plugins:** HTML Publisher, Groovy Postbuild (Git is pre-installed)

**SSH credentials** — repository is cloned via SSH key stored in Jenkins credentials (`github-ssh`).

**Four Freestyle jobs**, one per scenario:

| Job | Test file |
|-----|-----------|
| OpenEvent_SystemLevelFlow | api_system_level_flow.jmx |
| OpenEvent_SeparateEndpoints | api_separate_endpoints.jmx |
| OpenEvent_EventsPageSize | api_events_pagesize.jmx |
| OpenEvent_SpeakersTest | api_speakers.jmx |

Each job is parameterized with USERS, RAMPUP, DURATION, LOOPS, TEST_TITLE. Build command example:

```bash
/Users/<user>/Tools/apache-jmeter-5.5/bin/jmeter.sh -n \
  -t open-event-api/api_system_level_flow.jmx \
  -l results.csv -j jmeter.log -e -o report -f \
  -JUSERS=$USERS -JRAMPUP=$RAMPUP \
  -JDURATION=$DURATION -JLOOPS=$LOOPS \
  -JTEST_TITLE="$TEST_TITLE"
```

**Post-build:**
- Archives `results.csv` and `jmeter.log`
- Publishes HTML report (requires CSP fix in Script Console after each restart)
- Groovy Postbuild generates a Grafana link with the exact test time range in the build description

```groovy
import hudson.model.*
def build      = Thread.currentThread().executable
def grafanaUrl = System.getenv("JENKINS_GRAFANA_URL") ?: "127.0.0.1:3000"
def start      = build.getStartTimeInMillis()
def end        = start + build.getExecutor().getElapsedTime()
def url = String.format(
    "http://%s/d/rxKcwpFmkk/load-test-monitoring-dashboard-last?from=%s&to=%s",
    grafanaUrl, start, end
)
build.setDescription("<a href='${url}'>Grafana Performance Result</a>")
```

---
