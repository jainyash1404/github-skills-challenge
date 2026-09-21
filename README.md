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

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

