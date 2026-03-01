# Local Setup Reference

Configuration reference for the local performance testing stack.

---

## Prerequisites

- Java 8+
- `~/Tools/` directory containing: `apache-jmeter-5.5/`, `influxdb-1.x/`, `grafana-12.x/`, `telegraf/`
- GitHub account with SSH key configured: `~/.ssh/jenkins_github`

> Tools are stored outside the Jenkins workspace. Jenkins build commands must use absolute paths.

---

## InfluxDB

**Start:** `cd ~/Tools/influxdb-1.x/ && ./influxd`  
**Verify:** `curl -i http://localhost:8086/ping` → `204 No Content`

**Required databases:**
```sql
CREATE DATABASE jmeter    -- JMeter metrics
CREATE DATABASE telegraf  -- System metrics
```

**Version note:** InfluxDB 1.x only. Version 2.x uses Flux — incompatible with JMeter Backend Listener and standard Grafana dashboards.

---

## Telegraf

**Config** (`~/Tools/telegraf/telegraf.conf`):
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

**Start:** `./telegraf --config telegraf.conf`  
**Verify:** `USE telegraf` → `SHOW MEASUREMENTS` → cpu, mem, disk, net, system

---

## Grafana

**Start:** `cd ~/Tools/grafana-12.x/ && ./bin/grafana-server` → `http://localhost:3000`

**Data sources** (Administration → Data Sources → InfluxDB):

| Name | Query Language | URL | Database |
|-----------|-------------|----------|
| jmeter | InfluxQL | http://localhost:8086 | jmeter |
| telegraf | InfluxQL | http://localhost:8086 | telegraf |

**Dashboards** (Dashboards → Import):

| Dashboard | Data Source |
|-----------|-------------|
| JMeter Load Test Monitoring | jmeter |
| JMeter Comparison Dashboard | jmeter |
| Linux System Overview | telegraf |
| Linux System Overview | telegraf |

**Anonymous access** — required for Jenkins Grafana links to work without login.  
Edit `conf/grafana.ini`:
```ini
[auth.anonymous]
enabled  = true
org_name = Main Org.
org_role = Viewer
```
Restart Grafana after change.

---

## JMeter

**Backend Listener** (required in each `.jmx`):

| Parameter | Value |
|-----------|-------|
| influxdbUrl | `http://localhost:8086/write?db=jmeter` |
| application | unique per scenario (see table below) |
| summaryOnly | `false` |
| percentiles | `90;95;99` |
| testTitle | `${__P(TEST_TITLE, local_test)}` |

| File | application |
|------|-------------|
| api_system_level_flow.jmx | system_level_flow |
| api_separate_endpoints.jmx | separate_endpoints |
| api_events_pagesize.jmx | events_pagesize |
| api_speakers.jmx | speakers_test |

**Test parameters** (all via `${__P()}`):

| Parameter | Default | Description |
|-----------|-------------|
| USERS | 1 | Thread count |
| RAMPUP | 60 | Ramp-up period (seconds) |
| DURATION | 120 | Test duration (seconds) |
| LOOPS | -1 | -1 = duration-controlled |
| TEST_TITLE | local_test | Displayed in Grafana |

**jmeter.properties fix** (`~/Tools/apache-jmeter-5.5/bin/jmeter.properties`):
```properties
backend_metrics_window_mode=timed
backend_metrics_window=5000
backend_metrics_large_window=50000
```
Without this, percentiles in Grafana diverge from the HTML Aggregate Report.

---

## Git

Repository cloned via SSH. Key location: `~/.ssh/jenkins_github`

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/jenkins_github
# Add ~/.ssh/jenkins_github.pub to GitHub → Settings → SSH keys
```

HTTPS authentication is not used — macOS Keychain stores stale credentials and returns 403.

`.gitignore` excludes: `results.csv`, `*.log`, `report/`, `apache-jmeter*/`, `.DS_Store`

---

## Jenkins

**Start:** `java -jar jenkins.war --httpPort=8080`

**Required plugins:** HTML Publisher, Groovy Postbuild (Git is pre-installed)

**SSH credentials** in Jenkins (Manage Jenkins → Credentials):

| Field | Value |
|-------|-------|
| Kind | SSH Username with private key |
| ID | github-ssh |
| Username | git |
| Private Key | contents of `~/.ssh/jenkins_github` |

**SCM configuration** (each job):

| Field | Value |
|-------|-------|
| Repository URL | `git@github.com:kalyniuk-vitalik/open-event-performance-tests.git` |
| Credentials | github-ssh |
| Branch | `*/main` |

**Build parameters** (each job):

| Type | Name | Default |
|-----------|-------------|
| String | USERS | 1 |
| String | RAMPUP | 60 |
| String | DURATION | 120 |
| String | LOOPS | -1 |
| Validating String | TEST_TITLE | — (regex `.+`, required) |

**Build command** (absolute path required):
```bash
/Users/<user>/Tools/apache-jmeter-5.5/bin/jmeter.sh -n \
  -t open-event-api/api_system_level_flow.jmx \
  -l results.csv -j jmeter.log -e -o report -f \
  -JUSERS=$USERS -JRAMPUP=$RAMPUP \
  -JDURATION=$DURATION -JLOOPS=$LOOPS \
  -JTEST_TITLE="$TEST_TITLE"
```

**Post-build actions:**
- Archive artifacts: `jmeter.log, results.csv`
- Publish HTML reports: dir=`report`, index=`index.html`, keep past reports ON

**CSP fix** (run after each Jenkins restart via Script Console):
```groovy
System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")
```

**Markup Formatter:** Manage Jenkins → Configure Global Security → Safe HTML

**Groovy Postbuild** — generates Grafana link with exact test time range:
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
