# Capacity Plan – Vision Moderation Service

## 1. Latency Budget Breakdown

| Stage                  | Budget (ms) | Notes                                     |
| ---------------------- | ----------: | ----------------------------------------- |
| Network In             |           5 | Incoming request transmission             |
| Auth + Routing         |           5 | API key validation and routing            |
| Payload Parsing        |          10 | Decode and validate image payload         |
| Feature Lookup (Redis) |           8 | Given in scenario                         |
| Pre-processing         |           8 | Resize and normalization                  |
| Model Inference        |          22 | T4 GPU inference time                     |
| Post-processing        |           7 | Label ranking and filtering               |
| Serialization          |           5 | JSON response generation                  |
| Network Out            |           5 | Response transmission                     |
| Headroom               |         175 | Buffer for traffic spikes and variability |
| **Total**              |     **250** | p95 latency budget                        |

The latency budget totals exactly 250 ms. The measured processing path consumes approximately 75 ms, leaving 175 ms of positive headroom for transient load increases, queueing delays, and infrastructure variability.

---

## 2. CPU vs GPU Decision

We choose GPU-based serving using NVIDIA T4 instances.

The model requires approximately 75 ms inference time on a CPU core and 22 ms on a T4 GPU. This provides roughly a 3.4× reduction in inference latency. Assuming 22 ms inference time, a single GPU replica can process approximately 45 requests per second before accounting for batching improvements.

A cloud T4 instance typically costs around $0.50–$0.70 per hour, while a CPU-only instance costs around $0.10–$0.15 per hour. Although GPUs are more expensive per instance, the significantly higher throughput reduces the number of replicas required and helps satisfy the p95 latency target of 250 ms during peak traffic periods.

For a latency-sensitive vision moderation workload, GPU serving provides a better balance of performance and operational simplicity.

---

## 3. Replica Sizing

| Scenario          | Target RPS | Per-Replica Throughput | Replicas Required | +30% Headroom | Estimated Monthly Cost |
| ----------------- | ---------: | ---------------------: | ----------------: | ------------: | ---------------------: |
| Sustained Traffic |        300 |                 45 RPS |                 7 |            10 |                ~$3,600 |
| Peak Traffic      |        500 |                 45 RPS |                12 |            16 |                ~$5,800 |

Traffic spikes will be handled through autoscaling. The service will maintain 10 warm replicas during normal operation and scale to 16 replicas during peak demand periods. This approach minimizes latency while avoiding excessive idle infrastructure costs.

---

## 4. Batching Decision

Dynamic batching will be enabled.

The service will use a maximum batch size of 8 requests and a maximum wait window of 5 milliseconds. The wait window is small compared with the 250 ms latency budget and therefore has minimal impact on user-facing latency.

Dynamic batching improves GPU utilization by combining multiple requests into a single inference pass. Under sustained traffic levels, this increases throughput and lowers the cost per prediction. A batch size of 8 is conservative enough to avoid excessive memory usage while still delivering meaningful efficiency gains.

The 5 ms batching window ensures that requests are not delayed significantly. Even under moderate traffic, the system can form batches quickly, improving throughput while remaining comfortably within the p95 latency target. This configuration provides a good balance between latency and infrastructure efficiency.
