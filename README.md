# Open Event Performance Tests

JMeter performance test scripts for Open Event API.

## Test Scripts

| File | Description |
|------|-------------|
| api_separate_endpoints.jmx | Performance test for separate API endpoints |
| api_system_level_flow.jmx | System level flow test |
| api_events_pagesize.jmx | Events page size performance test |
| api_speakers.jmx | Speakers endpoint performance test |

## Requirements
- Apache JMeter 5.5+
- InfluxDB 1.x
- Grafana

## Running Tests
Tests are executed via Jenkins CI with parameterized builds.
