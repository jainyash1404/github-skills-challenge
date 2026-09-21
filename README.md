# AIOps Payment-Service Assessment

## Scenario

This repository monitors a synthetic `payment-service`. Each operational record contains a timestamp, service name, response-time and resource-utilization metrics, plus a log level and message. The service normally processes payment requests quickly, but a short incident occurs when requests time out and database connectivity degrades.

The operational problem is detecting that incident promptly from both telemetry and logs. High latency, CPU or memory utilization, and concerning log events should be turned into an actionable anomaly event instead of being discovered only after users report failed or slow payments.

AIOps in this assessment connects those steps: it analyzes service telemetry, identifies abnormal observations, publishes anomaly events, and passes them through a simulated event-streaming workflow for downstream processing.

## Repository Components

- [data/service_data.json](data/service_data.json) is the operational dataset. Its numeric fields are metrics; `timestamp`, `log_level`, and `message` provide log and event context.
- [src/anomaly_detector.py](src/anomaly_detector.py) applies threshold-based rules and creates an anomaly event containing the original record and detection reasons.
- [src/event_producer.py](src/event_producer.py) publishes anomaly events to an in-memory topic.
- [src/event_topic.py](src/event_topic.py) is the in-memory event topic that stores published messages.
- [src/event_consumer.py](src/event_consumer.py) reads messages from the topic for downstream handling.
- [src/aiops_pipeline.py](src/aiops_pipeline.py) loads the operational data, runs detection, sends detected events through the producer, and collects consumer output.
- [src/calculations.py](src/calculations.py) contains unrelated example calculation functions covered by the existing unit tests.
- [tests/test_aiops_pipeline.py](tests/test_aiops_pipeline.py) covers the detector and event-flow components; [tests/calculations_test.py](tests/calculations_test.py) covers the calculation examples.

The remaining sections record the observations, corrections, and reproducible execution results from the assessment workflow.

## Operational Data Analysis

The dataset contains ten records for `payment-service`, sampled at one-minute intervals from `2026-09-20T10:00:00` through `2026-09-20T10:09:00`. The ISO-like timestamps provide event ordering and make the incident timeline visible.

- **Metrics:** `response_time_ms`, `cpu_percent`, and `memory_percent` are numeric service metrics. The detector compares them with thresholds of 500 ms, 80%, and 80%, respectively.
- **Log information:** `log_level` and `message` describe the corresponding service log event. `timestamp` identifies when both the metrics and log entry were observed; `service` identifies their source.
- **Normal observations:** 10:00–10:04 and 10:07–10:09 have `INFO` logs, response times from 120–150 ms, CPU from 42–50%, and memory from 51–57%. These values are stable and below the configured anomaly thresholds.
- **Unusual observations:** 10:05 reports a 610 ms response time and an `ERROR` message, `Payment service timeout`. At 10:06, response time increases to 640 ms, CPU to 94%, memory to 91%, and the `ERROR` message is `Database connection timeout`. Together these records indicate a short payment-service/database incident.

The expected anomaly set is therefore the two records at 10:05 and 10:06. The 10:05 record is anomalous because of latency and its error log; the 10:06 record is anomalous because of latency, CPU, memory, and its error log.

## Anomaly-Detection Findings

Running the provided `AnomalyDetector` against all ten records produced two readable anomaly events and did not flag any normal observation:

| Timestamp | Log information | Detected reasons |
| --- | --- | --- |
| `2026-09-20T10:05:00` | `ERROR` - Payment service timeout | High response time |
| `2026-09-20T10:06:00` | `ERROR` - Database connection timeout | High response time; high CPU utilization; high memory utilization |

The detector correctly identified the abnormal metric behavior and retained the complete source record in each event. It missed the expected log-based anomaly signal for both records: the implementation currently checks for `WARNING`, while the concerning records use `ERROR`. No normal event was incorrectly flagged. This is a limitation of the static rule set; a correction should recognize the log levels present in the operational data, and a future improvement could combine configurable log-severity rules with adaptive or time-window-based thresholds.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

