# Grafana Dashboards

Real-time dashboards used during performance test sessions.

---

## Dashboard Overview

| Dashboard | Base ID | Data Source | Purpose |
|-----------|---------|-------------|---------|
| JMeter Load Test Monitoring | 1152 (customized) | InfluxDB `jmeter` / `jmeter_gha` | Active threads, RPS, response time, error rate |
| JMeter Comparison Dashboard | 5496 (customized) | InfluxDB `jmeter` / `jmeter_gha` | Side-by-side comparison across test runs |
| Linux System Overview | 928 (customized) | InfluxDB `telegraf` | CPU/RAM/Disk/Network during test |

---

## Setup

Dashboards are not exported as JSON - they contain local InfluxDB data source references that are environment-specific.

To reproduce:
1. Install Grafana and import community dashboards by ID (1152, 5496, 928)
2. Configure data sources per [../docs/local-setup.md](../docs/local-setup.md) or [../docs/cloud-setup.md](../docs/cloud-setup.md)
3. Set `backend_metrics_window_mode=timed` in `jmeter.properties` for accurate real-time percentiles
