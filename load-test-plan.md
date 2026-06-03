# Load Test Plan

## Tool Choice

**k6** will be used for load testing because it is easy to script, provides detailed latency metrics (p50, p95, p99), and is widely used for HTTP API performance testing. It also supports gradual traffic ramp-up and realistic workload simulation.

---

## Test Phases

| Phase          | Duration | Description                                                               |
| -------------- | -------- | ------------------------------------------------------------------------- |
| Warmup         | 5 min    | Allow caches, Redis connections, and model replicas to reach steady state |
| Ramp Up        | 10 min   | Increase traffic from 50 RPS to 500 RPS gradually                         |
| Sustained Peak | 15 min   | Hold constant load at 500 RPS                                             |
| Soak Test      | 60 min   | Run at 300 RPS sustained traffic to detect memory leaks and degradation   |

---

## Traffic Shape

* 90% synchronous requests (`POST /v1/predict`)
* 10% batch requests (`POST /v1/predict-batch`)
* Image sizes:

  * 70% between 100 KB and 1 MB
  * 20% between 1 MB and 3 MB
  * 10% between 3 MB and 5 MB
* Concurrency model:

  * 300 concurrent virtual users during normal load
  * Up to 500 concurrent virtual users during peak testing

---

## Pass/Fail Criteria

| Metric              | Threshold |
| ------------------- | --------- |
| p95 latency         | ≤ 250 ms  |
| p99 latency         | ≤ 500 ms  |
| Error rate          | < 1%      |
| Availability        | ≥ 99.9%   |
| Successful requests | ≥ 99%     |
| Redis latency p95   | ≤ 15 ms   |

The test fails if any threshold is exceeded during the sustained peak phase.

---

## Bottleneck Checklist

During testing, monitor the following metrics on every replica:

* CPU utilization (%)
* Memory utilization (%)
* GPU utilization (%)
* GPU memory usage
* Request queue depth
* Redis latency (p95 and p99)
* Network throughput
* Error rate (4xx and 5xx)
* Pod/container restarts
* Disk I/O
* Autoscaling events

The goal is to identify whether bottlenecks occur in model inference, Redis feature lookup, networking, or resource exhaustion before production deployment.
