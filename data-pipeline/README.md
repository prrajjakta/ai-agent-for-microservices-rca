# Telemetry Data Pipeline

Python-based collection and processing of metrics, logs,
and distributed traces for root cause analysis.

## Data Sources
- Prometheus HTTP API: Metrics.
- Loki API: Application logs.
- Jaeger API: Distributed traces.
- ground_truth.csv: Fault labels and timestamps.

## Responsibilities
- Collect telemetry for each fault observation window.
- Calculate per-service metrics and anomaly features.
- Extract log templates using the Drain algorithm
  and the drain3 library.
- Build service dependency graphs and latency features.
- Produce structured datasets for the RCA agent.
