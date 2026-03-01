# Grafana Dashboards

Pre-configured Grafana dashboards stored as JSON exports — ready to import.

## How to import

1. Grafana → Dashboards → **Import**
2. Upload JSON file
3. Select correct datasource (`jmeter` or `telegraf`)

## Dashboards

| File | Dashboard | Data Source |
|-----------|-------------|-------------|
| `jmeter-load-test.json` | JMeter Load Test Monitoring | jmeter |
| `jmeter-comparison.json` | JMeter Comparison Dashboard | jmeter |
| `server-monitoring-local.json` | Linux System Overview (local) | telegraf |
| `server-monitoring-cloud.json` | Telegraf: System Dashboard (cloud) | telegraf |

## How to export (to update)

1. Open dashboard in Grafana
2. Click **Share** (top right) → **Export** tab
3. **Save to file** → replace JSON here
