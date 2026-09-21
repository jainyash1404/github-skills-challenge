# AIOps Payment-Service Assessment

## What This Project Is About

This small project watches a fictional `payment-service`. Most requests are handled normally, but the sample data includes a brief incident where payment requests slow down and the database starts timing out.

The practical problem is spotting that incident from the service's metrics and logs before it becomes a larger user-facing issue. In this exercise, AIOps is the glue between those steps: it finds unusual telemetry, turns it into an event, and passes that event through a simple streaming pipeline for further handling.

## Where Things Live

- [data/service_data.json](data/service_data.json) contains the ten sample service records.
- [src/anomaly_detector.py](src/anomaly_detector.py) applies the response-time, CPU, memory, and log-level rules.
- [src/event_producer.py](src/event_producer.py) publishes anomaly events.
- [src/event_topic.py](src/event_topic.py) provides the in-memory topic used by the simulation.
- [src/event_consumer.py](src/event_consumer.py) reads events from that topic.
- [src/aiops_pipeline.py](src/aiops_pipeline.py) connects data loading, detection, publishing, and consumption.
- [tests/test_aiops_pipeline.py](tests/test_aiops_pipeline.py) checks the detector and event flow.
- [src/calculations.py](src/calculations.py) and [tests/calculations_test.py](tests/calculations_test.py) are the original calculation example and its tests.

## What the Data Shows

The records cover `payment-service` from `2026-09-20T10:00:00` to `2026-09-20T10:09:00`, one record per minute. The timestamps make it possible to follow the incident in order.

The metric fields are `response_time_ms`, `cpu_percent`, and `memory_percent`. The log fields are `log_level` and `message`; `service` identifies the source of each record. The detector uses thresholds of 500 ms for response time and 80% for both CPU and memory.

The normal baseline is visible from 10:00 to 10:04 and again from 10:07 to 10:09. Those records have `INFO` logs, response times between 120 and 150 ms, CPU between 42% and 50%, and memory between 51% and 57%.

The two unusual records are:

- **10:05:** response time reaches 610 ms and the service logs `ERROR: Payment service timeout`.
- **10:06:** response time reaches 640 ms, CPU reaches 94%, memory reaches 91%, and the service logs `ERROR: Database connection timeout`.

These are the two records the analysis should identify as the incident.

## Detection Results

Before the correction, the detector found both incident records from their metrics and did not flag any normal records:

| Time | Log entry | Initial reasons |
| --- | --- | --- |
| `10:05` | `ERROR: Payment service timeout` | High response time |
| `10:06` | `ERROR: Database connection timeout` | High response time; high CPU utilization; high memory utilization |

The original log rule looked for `WARNING`, while the supplied incident records use `ERROR`. As a result, the first run missed the log-based signal even though it correctly found the metric problems. The rule was changed to recognize `ERROR`. No normal event was incorrectly flagged.

The detector is deliberately simple and uses fixed thresholds. A useful next step would be configurable log-severity rules and thresholds that adapt to the service's normal baseline.

## Event Flow

The event path is intentionally small:

1. The detector creates an `ANOMALY` event and keeps the original record with it.
2. `EventProducer` publishes the event to an `EventTopic`.
3. The in-memory topic stores the message.
4. `EventConsumer` reads the message and returns it to the pipeline as downstream AIOps input.

The first pipeline run produced two events but consumed none. The producer was writing to `service-events`, while the consumer was reading from a different `anomaly-events` topic. The pipeline was corrected to share one `anomaly-events` instance.

There was also an import issue: the modules used top-level sibling imports, so importing the pipeline as `src.aiops_pipeline` failed. Package-compatible imports were added while keeping the original direct-script entry point working.

## Final Run

After those corrections, the complete flow was:

`Operational data -> anomaly detection -> event -> producer -> topic -> consumer -> AIOps output`

The final run processed 10 records, found 2 anomalies, published 2 events, and consumed both events. The output identified the payment timeout at 10:05 and the database connection timeout at 10:06, including their metric breaches and `Error log detected` as reasons.

## Reproduce It

From the repository root:

```bash
python3 src/aiops_pipeline.py
python3 -m pytest -q
```

The expected pipeline summary is 10 records processed, 2 anomalies detected, and 2 events consumed. The test suite passes with 8 tests.

The bare `pytest -q` launcher may fail to import `src` in this container because of its executable path. `python3 -m pytest -q` uses the active interpreter and is the reliable validation command here.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

