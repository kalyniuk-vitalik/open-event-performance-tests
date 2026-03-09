# Cloud Infrastructure Setup

Cloud pipeline runs the same JMeter scenarios automatically on every push to `main`, streaming metrics to a persistent GCP VM.

## Components

| Tool | Where | Purpose |
|------|-------|---------|
| GitHub Actions | GitHub-hosted runner | Test execution |
| GCP VM (e2-micro, us-central1) | Google Cloud | Hosts InfluxDB + Grafana + Telegraf |
| InfluxDB 1.x | GCP VM | Metrics storage (db: jmeter_gha) |
| Grafana | GCP VM | Dashboards |
| Telegraf | GCP VM | VM system metrics |

---

## How it works

On push to `main` (or manual trigger via `workflow_dispatch`), GitHub Actions installs JMeter 5.5 with required plugins and runs all four scenarios sequentially. JMeter streams metrics to InfluxDB on the GCP VM during each run. After all tests complete, results are uploaded as GitHub Actions artifacts.

```
GitHub Actions ──► JMeter ──► GCP VM: InfluxDB (db: jmeter_gha) ──► Grafana
                              GCP VM: Telegraf ──────────────────► Grafana
```

---

## Pipeline Isolation

Local and cloud pipelines write to separate InfluxDB databases. The `INFLUX_DB` parameter defaults to `localhost` for local runs and is overridden via GitHub Secret for cloud runs - same `.jmx` files work in both environments.

| Pipeline | Host | Database |
|---------|------|---------|
| Jenkins (local) | localhost:8086 | jmeter |
| GitHub Actions | GCP VM:8086 | jmeter_gha |

---

## GCP VM

**Specs:** e2-micro, us-central1, Ubuntu 22.04 LTS

**Firewall rules:**

| Port | Source | Purpose |
|------|--------|---------|
| 8086 | 0.0.0.0/0 | InfluxDB - GitHub Actions uses dynamic IPs |
| 3000 | your IP | Grafana - restrict access |

---

## InfluxDB on GCP VM

```bash
sudo apt install -y influxdb influxdb-client
sudo systemctl enable --now influxdb
influx -execute 'CREATE DATABASE jmeter_gha'
```

---

## Grafana on GCP VM

Installed via apt. Data sources:

| Name | Query Language | URL | Database |
|------|---------------|-----|---------|
| jmeter | InfluxQL | http://localhost:8086 | jmeter_gha |
| telegraf | InfluxQL | http://localhost:8086 | telegraf |

Import dashboards from [`/dashboards`](../dashboards/):

| Dashboard | Data Source |
|-----------|------------|
| JMeter Load Test Monitoring | jmeter |
| JMeter Comparison Dashboard | jmeter |
| Telegraf: System Dashboard | telegraf |

---

## Telegraf on GCP VM

```toml
[[outputs.influxdb]]
  urls     = ["http://127.0.0.1:8086"]
  database = "telegraf"

[[inputs.cpu]]
[[inputs.mem]]
[[inputs.disk]]
[[inputs.net]]
[[inputs.system]]
[[inputs.processes]]
[[inputs.swap]]
```

---

## GitHub Actions Workflow

**Secret required:**

| Name | Value |
|------|-------|
| `INFLUX_WRITE_URL` | `http://<VM_IP>:8086/write?db=jmeter_gha` |

> GCP VM IP changes on restart - update this secret accordingly.

**Key workflow parameters per step:**

```yaml
-JINFLUX_DB=${{ secrets.INFLUX_WRITE_URL }}
-Jbackend_metrics_window_mode=timed
-Jbackend_metrics_window=5000
-JTEST_TITLE="GHA - <scenario_name>"
```

**Artifacts** uploaded after run (even on failure via `if: always()`):
- `results-*.csv`
- `jmeter-*.log`
- `html-report-*/`

---

## Why Sequential Tests

Running scenarios in parallel via matrix strategy loads the API server simultaneously, producing unreliable results. Sequential execution gives each scenario an isolated measurement window.

---
