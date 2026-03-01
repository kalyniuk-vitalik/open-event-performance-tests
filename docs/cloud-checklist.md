# Cloud Setup Reference

Configuration reference for the cloud pipeline — GitHub Actions + GCP VM.

---

## GCP VM

**Specs:** e2-micro, us-central1, Ubuntu 22.04 LTS

**Firewall rules:**

| Port | Source | Purpose |
|-----------|-------------|
| tcp:8086 | 0.0.0.0/0 | InfluxDB — GitHub Actions uses dynamic IPs |
| tcp:3000 | your IP | Grafana — restrict to known IP |

Find your IP: `curl -s https://api.ipify.org`

---

## InfluxDB on GCP VM

```bash
sudo apt update && sudo apt -y install influxdb influxdb-client
sudo systemctl enable --now influxdb

# Verify
curl -i http://localhost:8086/ping  # → 204 No Content

# Create database
influx -execute 'CREATE DATABASE jmeter_gha'
```

External access test: `curl -i http://<VM_IP>:8086/ping`

---

## Grafana on GCP VM

```bash
sudo apt -y install adduser libfontconfig1 wget
wget -q -O - https://packages.grafana.com/gpg.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] \
  https://packages.grafana.com/oss/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt -y install grafana
sudo systemctl enable --now grafana-server
```

Access: `http://<VM_IP>:3000` (admin/admin)

**Data sources:**

| Name | Query Language | URL | Database |
|-----------|-------------|----------|
| jmeter | InfluxQL | http://localhost:8086 | jmeter_gha |
| telegraf | InfluxQL | http://localhost:8086 | telegraf |

Import dashboards from [`/dashboards`](../dashboards/):

| Dashboard | Data Source |
|-----------|-------------|
| JMeter Load Test Monitoring | jmeter |
| JMeter Comparison Dashboard | jmeter |
| Telegraf: System Dashboard | telegraf |

---

## Telegraf on GCP VM

```bash
sudo apt install -y telegraf
```

`/etc/telegraf/telegraf.conf`:
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
[[inputs.processes]]
[[inputs.swap]]
```

```bash
sudo systemctl enable --now telegraf
# Verify
influx -database telegraf -execute 'SHOW MEASUREMENTS'
```

---

## GitHub Actions

**Required secret** (Settings → Secrets and variables → Actions):

| Name | Value |
|------|-------|
| `INFLUX_WRITE_URL` | `http://<VM_IP>:8086/write?db=jmeter_gha` |

> VM IP changes on restart — update this secret when it does.

**Workflow trigger:** push to `main` or manual `workflow_dispatch`

**Key parameters passed to each JMeter run:**
```yaml
-JINFLUX_DB=${{ secrets.INFLUX_WRITE_URL }}
-Jbackend_metrics_window_mode=timed
-Jbackend_metrics_window=5000
-Jbackend_metrics_large_window=50000
-JTEST_TITLE="GHA - <scenario>"
```

**Artifact upload** (after all steps, including failures):
```yaml
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: jmeter-results
    path: |
      results-*.csv
      jmeter-*.log
      html-report-*/
```

---

## JMeter Install in GitHub Actions

```bash
wget -q https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-5.5.tgz
tar -xzf apache-jmeter-5.5.tgz

# Plugins Manager
wget -q ".../jmeter-plugins-manager-1.9.jar" -O apache-jmeter-5.5/lib/ext/jmeter-plugins-manager-1.9.jar
wget -q ".../cmdrunner-2.3.jar" -O apache-jmeter-5.5/lib/cmdrunner-2.3.jar
java -cp apache-jmeter-5.5/lib/ext/jmeter-plugins-manager-1.9.jar \
  org.jmeterplugins.repository.PluginManagerCMDInstaller
apache-jmeter-5.5/bin/PluginsManagerCMD.sh install \
  jpgc-graphs-basic,jpgc-tst,jpgc-casutg,jpgc-perfmon,jpgc-cmd,jpgc-csl
```

---

## Verifying Metrics

After a workflow run:
```bash
influx -database jmeter_gha -execute 'SHOW MEASUREMENTS'
# → jmeter, events

influx -database jmeter_gha -execute \
  'SELECT last("avg"), last("pct90.0") FROM jmeter WHERE time > now() - 2h'
```

In Grafana — set time range to match the Actions run time (UTC).

---
