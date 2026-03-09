# Performance Results — Open Event API

> 📄 [Full Performance Test Report (PDF)](docs/Performance_Test_Report_Kalyniuk.pdf)

---

## Summary

| Goal | Parameter | Value |
|------|-----------|-------|
| System Capacity | Max stable throughput | 5 RPS at 5 threads |
| Slowest endpoint | GET /v1/events | 2 RPS at 4 threads |
| Fastest endpoint | POST /v1/events | 9.2 RPS at 10 threads |
| Load Test stability | ~2 RPS, 0% errors | ✅ Stable |
| POST /v1/speakers degradation | 1 → 50 sessions | 782ms → 3,030ms (+287%) |
| GET /v1/events degradation | page[size] 10 → 50 | 935ms → 1,690ms (+81%) |

---

## 1. System Level Capacity Test

**Load model:** 20 threads, ramp-up 1800s, duration 1800s, Constant Throughput Timer 60 req/min

| Zone | Threads | Observation |
|------|---------|-------------|
| Comfort zone | 1 thread / 1 RPS | Minimal load, but already degrading — no stable performance |
| Degradation zone | 2–5 threads | RT grows, throughput unstable |
| Capacity point | **5 threads / 5 RPS** | Throughput plateaus, RT spikes |

**Bottleneck identified:** memory leak — heap grows continuously and is never released.

---

## 2. Separate Endpoints Capacity Test

**Load model:** 20 threads, ramp-up 300s, duration 300s, Constant Throughput Timer 60 req/min

| Endpoint | Max Throughput | Capacity Point |
|----------|---------------|----------------|
| POST /v1/auth/login | 8.4 RPS | 9 threads |
| GET /v1/users | 8.6 RPS | 9 threads |
| POST /v1/events | **9.2 RPS** | 10 threads |
| POST /v1/tickets | 8.0 RPS | 8 threads |
| GET /v1/events | **2.0 RPS** | 4 threads |

**Key finding:** gap between fastest (POST /v1/events) and slowest (GET /v1/events) is **4.6x** — points to uneven load distribution across system components.

---

## 3. Load Test

**Load model:** 2 threads, ramp-up 60s, duration 3600s, Constant Throughput Timer 60 req/min

| Endpoint | Avg RT | 90th pct | 95th pct | 99th pct | TPS | Errors |
|----------|--------|----------|----------|----------|-----|--------|
| POST /v1/auth/login | 464 ms | 527 ms | 527 ms | 527 ms | 0.20 req/s | 0% |
| GET /v1/users | 383 ms | 396 ms | 408 ms | 819 ms | 0.49 req/s | 0% |
| POST /v1/events | 447 ms | 520 ms | 792 ms | 871 ms | 0.49 req/s | 0% |
| POST /v1/tickets | 401 ms | 477 ms | 563 ms | 821 ms | 0.49 req/s | 0% |
| GET /v1/events | **1,040 ms** | 1,090 ms | 1,140 ms | 1,200 ms | 0.49 req/s | 0% |

**Key finding:** system is stable at ~2 RPS with 0% errors. GET /v1/events is a clear outlier — Avg RT is **2.7x higher** than the next slowest endpoint.

---

## 4. POST /v1/speakers — Sessions Performance

**Load model:** 1 thread, ramp-up 30s, duration 300s (750s for 50 sessions)

| Sessions | Avg RT | 90th pct | 95th pct | Errors |
|----------|--------|----------|----------|--------|
| 1 | 782 ms | 1,110 ms | 1,110 ms | 0% |
| 5 | 1,220 ms | 1,800 ms | 1,820 ms | 0% |
| 20 | 1,460 ms | 2,220 ms | 2,220 ms | 0% |
| 50 | **3,030 ms** | **4,770 ms** | **4,770 ms** | 0% |

**Key finding:** total degradation +287% from 1 to 50 sessions. Biggest jump between 20 and 50 sessions: +107% — suggests non-linear growth with large payloads.

---

## 5. GET /v1/events — page[size] Performance

**Load model:** 1 thread, ramp-up 30s, duration 300s

| page[size] | Avg RT | 90th pct | 95th pct | Errors |
|------------|--------|----------|----------|--------|
| 10 | 935 ms | 1,130 ms | 1,160 ms | 0% |
| 25 | 1,220 ms | 1,340 ms | 1,430 ms | 0% |
| 50 | **1,690 ms** | **1,870 ms** | **1,920 ms** | 0% |

**Key finding:** total degradation +81% from page[size]=10 to page[size]=50. Biggest relative jump between 10 and 25 (+30%), indicating non-linear growth as data volume increases.
