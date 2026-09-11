# End-to-End Benchmarking Guide

**Pipeline:** Devices → MN → MQTT → IN → Kafka → NiFi → Kafka → Druid (DB)

**Scope:** End-to-end only (device send → Druid queryable). Per-hop metrics are captured in the background only to explain E2E results, not reported separately.

---

## 1. Throughput Attributes

| Attribute                     | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| Messages sent                 | Total count and rate (msg/sec) from device simulators        |
| Messages received at Druid    | Total count and rate (rows/sec) queryable in Druid           |
| Data loss %                   | (sent − received) / sent                                     |
| Duplicate rate                | % of messages that arrived more than once (common with QoS 1 / at-least-once) |
| Bytes sent vs. bytes ingested | Throughput in MB/sec, not just message count                 |
| Average throughput            | Average rate held over full test duration                    |
| Peak throughput               | Max rate achieved before degradation/errors begin            |

## 2. Latency Attributes (device → Druid)

| Attribute           | Description     |
| ------------------- | --------------- |
| p50 latency         | Typical case    |
| p90 / p95 latency   | Upper-mid range |
| p99 / p99.9 latency | Tail latency    |
| Max latency         | Worst observed  |

## 3. Correctness / Completeness Attributes

| Attribute          | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| Data integrity     | Sample-check payload values at Druid match source            |
| Ordering           | Messages processed in sent order, or reordered (use timestamps/sequence numbers) |
| Schema consistency | No silently dropped/null fields at Druid vs. source          |

## 4. Load Test Parameters (independent variables)

| Parameter                              | Notes                                                  |
| --------------------------------------- | ------------------------------------------------------ |
| Number of concurrent simulated devices |                                                         |
| Message rate per device                | msg/sec per device                                     |
| Payload size                           | bytes                                                  |
| Test duration                          | Run both short burst and long sustained (hours)        |
| QoS level                              | MQTT QoS 0/1/2 — directly affects loss and duplication |

## 5. Background System Correlation

Capture these alongside E2E numbers so degradation at the breaking point can be explained:

- Aggregate CPU / memory usage across the pipeline
- Error / exception count during the test
- **Consumer lag at end of test** — critical. Always let the pipeline fully drain (lag = 0) before calculating final loss %, otherwise in-flight messages in Kafka/NiFi queues will be misread as "loss" when they are actually just delayed.

### 5a. Server-Side Network Correlation

Network-layer issues on the pipeline's own servers often explain loss/latency that would otherwise look like an "application" problem. Capture these per node (MN, Kafka brokers, NiFi nodes) alongside the background system stats above. 

**Scope:** these stats apply only to hops where the two components sit on different physical/virtual servers.

- **NIC throughput (RX/TX bytes/sec)** and **interface utilization %** relative to link speed, on both the sending and receiving server
- **RX/TX errors, drops, overruns** at the interface — nonzero values mean packets are being dropped before the application ever sees them

## 6. Repeatability & Statistical Validity

A single run per load level is not sufficient — pipeline behavior under load has run-to-run variance from JVM warm-up, GC pauses, network jitter, and scheduler noise.

- Run each scenario **at least 2–3 times** at a given load level.
- Report **mean and stddev (or min/max)** for loss %, duplicate %, and latency percentiles across repeated runs, not just a single sample.
- Exclude the **warm-up window** (first 30–60s, or until throughput/latency stabilizes) from steady-state calculations — cold starts, partition rebalances, and Druid task spin-up will skew results if included.

## 7. Environment & Config Baseline

Record the infrastructure and versions each set of results was produced against, since raw numbers are meaningless without this context:

- Broker count and partition count (Kafka)
- NiFi node/thread count and flow version
- Druid ingestion task count and cluster sizing
- Pipeline/component build or commit version
- Date/time of test run

---

## 8. Report Tables

Each **Scenario ID** may have multiple **Run #** rows (per Section 6). Tables are linked by ID + Run #.

### 8a. Throughput & Loss

| ID | Date | Scenario | Run # | Devices | Msg Rate/Device | Payload Size | QoS | Duration | Sent | Received | Loss % | Dup % | Avg Throughput | Peak Throughput |
| -- | ---- | -------- | ----- | ------- | ---------------- | ------------- | --- | -------- | ---- | -------- | ------ | ----- | --------------- | ---------------- |
| S1 | 2026-09-08 | 5k devices, steady load | 1 | 5,000 | 1 | 512 B | 1 | 30 min | 9,000,000 | 8,991,300 | 0.097% | 0.21% | 5,000 msg/s | 5,120 msg/s |
| S1 | 2026-09-08 | 5k devices, steady load | 2 | 5,000 | 1 | 512 B | 1 | 30 min | 9,000,000 | 8,993,800 | 0.069% | 0.18% | 5,000 msg/s | 5,105 msg/s |
| S2 | 2026-09-08 | 20k devices, burst | 1 | 20,000 | 2 | 1 KB | 0 | 10 min | 24,000,000 | 23,712,900 | 1.20% | 0.02% | 40,000 msg/s | 46,800 msg/s |

*Row(s) above are sample/example data — replace with actual measured results.*

### 8b. Latency

| ID | Run # | p50 | p90 | p95 | p99 | p99.9 | Max Latency |
| -- | ----- | --- | --- | --- | --- | ----- | ----------- |
| S1 | 1 | 420 ms | 780 ms | 910 ms | 1,450 ms | 2,100 ms | 3,320 ms |
| S1 | 2 | 410 ms | 765 ms | 895 ms | 1,390 ms | 2,050 ms | 3,180 ms |
| S2 | 1 | 610 ms | 1,150 ms | 1,480 ms | 2,900 ms | 4,600 ms | 6,750 ms |

*Row(s) above are sample/example data — replace with actual measured results.*

### 8c. Correctness & System Correlation

| ID | Run # | Node | Role | Data Integrity (**Pass/Fail**) | Ordering (**Pass/Fail**) | Schema Consistency (**Pass/Fail**) | CPU % (avg/peak) | Mem % (avg/peak) | Error/Exception Count | Consumer Lag @ End | Notes |
| -- | ----- | ---------------------- | ---------------- | -------------------------- | ------------------ | ------------------ | ------------------------ | -------------------- | ----- | ----- | ----- |
| S1 | 1 | nifi-01 | Nifi | Pass | Pass | Pass | 55% / 78% | 50% / 70% | 3 | 0 | Drained fully before final count |
| S1 | 2 | kafka-01 | Nifi | Pass | Pass | Pass | 71% / 94% | 40% / 54% | 1 | 0 |  |
| S2 | 1 | kafka-01 | Kafka | Pass | Fail (0.4% out-of-order) | Pass | 71% / 94% | 68% / 88% | 27 | 0                  |                                  |

*Row(s) above are sample/example data — replace with actual measured results.*

### 8d. Server-Side Network Stats

| ID | Run # | Node | Role | RX Throughput (avg/peak) | TX Throughput (avg/peak) | NIC Util % (avg/peak) | RX/TX Errors/Drops | Notes |
|----|-------|------|------|---------------------------|---------------------------|------------------------|----------------------|-------|
| S1 | 1 | mn-01 | MQTT Broker | 42 MB/s / 61 MB/s | 8 MB/s / 12 MB/s | 34% / 49% | 0 / 0 | |
| S1 | 1 | kafka-01 | Kafka Broker | 55 MB/s / 79 MB/s | 55 MB/s / 80 MB/s | 44% / 63% | 0 / 0 | Incl. replication traffic |
| S1 | 1 | nifi-01 | NiFi | 30 MB/s / 45 MB/s | 30 MB/s / 46 MB/s | 24% / 36% | 0 / 0 | |
| S2 | 1 | mn-01 | MQTT Broker | 180 MB/s / 240 MB/s | 20 MB/s / 28 MB/s | 82% / 96% | 0 / 142 | Drops correlate w/ backlog full events |
| S2 | 1 | kafka-01 | Kafka Broker | 210 MB/s / 260 MB/s | 210 MB/s / 265 MB/s | 78% / 91% | 0 / 0 |  |

*Row(s) above are sample/example data — replace with actual measured results.*

