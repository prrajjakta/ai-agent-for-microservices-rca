# AI Agent for Automated Root Cause Analysis in Microservice-Based Cloud Systems

An evidence-grounded AI agent that investigates failures in a microservice-based cloud application using metrics, logs, and distributed traces. A classical graph-based algorithm ranks suspected root-cause services, while an LLM explains the ranking using structured, verifiable evidence.

The agent is **diagnosis-only**: it does not automatically fix failures or modify the Kubernetes cluster.

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [System Architecture](#system-architecture)
- [Technology Stack](#technology-stack)
- [Root Cause Analysis Approach](#root-cause-analysis-approach)
- [Fault Injection and Ground Truth](#fault-injection-and-ground-truth)
- [Data Pipeline](#data-pipeline)
- [Evaluation](#evaluation)
- [Project Constraints](#project-constraints)
- [Team Responsibilities and Deadlines](#team-responsibilities-and-deadlines)
- [Planned Repository Structure](#planned-repository-structure)
- [Getting Started](#getting-started)
- [References](#references)

## Overview

This project builds a controlled test environment for studying root cause analysis (RCA) in cloud-native microservices.

The target application is [Google's Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo), a demo e-commerce application composed of 11 microservices. We deliberately introduce controlled faults, observe their effects through the same telemetry sources an engineer would use, and evaluate whether the RCA system identifies the actual faulty service.

Because each injected fault is recorded, the project has **ground-truth labels** for evaluation. The system's diagnoses can therefore be measured against the known faulty service and fault interval.

### Core workflow

1. Deploy Online Boutique to a shared Kubernetes cluster.
2. Generate steady, repeatable application traffic.
3. Collect metrics, logs, and distributed traces.
4. Record a healthy baseline.
5. Inject controlled faults into selected services.
6. Collect telemetry for each fault window and construct per-service features and a dependency graph.
7. Rank suspected root-cause services with a classical graph-based algorithm.
8. Use an LLM to produce an evidence-backed explanation of the ranking.
9. Compare the results with ground truth and baseline methods.

## Objectives

- Build a repeatable, single-cluster microservice fault-injection environment.
- Investigate failures using metrics, logs, and traces.
- Implement a classical graph-based service-ranking method.
- Add an LLM explanation layer that is grounded in collected evidence.
- Evaluate localization accuracy, diagnosis time, and false positives.
- Measure the contribution of each telemetry source through ablation experiments.
- Keep the agent read-only with respect to the Kubernetes environment.

## System Architecture

```text
                         Shared Kubernetes Cluster
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌─────────────────────┐       ┌─────────────────────────────┐   │
│  │ Online Boutique     │◄──────│ Locust                      │   │
│  │ 11 microservices    │       │ Browsing, item views, carts │   │
│  └──────────┬──────────┘       └─────────────────────────────┘   │
│             │                                                    │
│             ▼                                                    │
│  ┌─────────────────────┐  ┌──────────────┐  ┌───────────────┐    │
│  │ Prometheus + Grafana│  │ Loki +       │  │ Jaeger        │    │
│  │ Metrics             │  │ Promtail     │  │ Distributed   │    │
│  │                     │  │ Logs         │  │ traces        │    │
│  └─────────────────────┘  └──────────────┘  └───────────────┘    │
│                                                                  │
│  ┌─────────────────────┐       ┌─────────────────────────────┐   │
│  │ Chaos Mesh          │◄──────│ inject.py                   │   │
│  │ Controlled faults   │       │ Fault orchestration +       │   │
│  │                     │       │ ground-truth recording      │   │
│  └─────────────────────┘       └─────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                                  │
                                  │ Query APIs for each fault window
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Python Data Pipeline                          │
│  Prometheus API + Loki API + Jaeger API                          │
│  Metrics features + Drain log templates + trace/call graph       │
└─────────────────────────────────┬────────────────────────────────┘
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                         RCA Agent                                │
│                                                                  │
│  ┌──────────────────────────┐   ┌─────────────────────────────┐  │
│  │ Classical graph-based    │──►│ LLM explanation layer       │  │
│  │ ranker                   │   │ Evidence-backed report      │  │
│  └──────────────────────────┘   └─────────────────────────────┘  │
└─────────────────────────────────┬────────────────────────────────┘
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Evaluation Harness                         │
│  Predictions vs. ground_truth.csv                                │
│  Accuracy, MAP, diagnosis time, false-positive rate, ablations   │
└──────────────────────────────────────────────────────────────────┘
```

### Component responsibilities

| Component | Responsibility |
|---|---|
| Online Boutique | Target application under investigation |
| Locust | Generates repeatable user-like traffic |
| Prometheus | Collects and serves time-series metrics |
| Grafana | Visualizes metrics and supports environment validation |
| Loki | Stores and serves application logs |
| Promtail | Ships logs to Loki |
| Jaeger | Stores and serves distributed traces |
| Chaos Mesh | Injects controlled faults into Kubernetes workloads or network paths |
| `inject.py` | Orchestrates fault scenarios and records ground truth |
| Python data pipeline | Retrieves and transforms telemetry into structured RCA inputs |
| Classical RCA ranker | Scores and ranks likely faulty services |
| LLM layer | Explains ranked results using supplied evidence |
| Evaluation harness | Compares predictions with ground-truth labels and baselines |

## Technology Stack

| Area | Technology | Purpose |
|---|---|---|
| Target application | [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) | 11-service demo e-commerce application |
| Orchestration | Kubernetes | Runs the shared environment |
| Local cluster | Minikube | Local development option |
| Cluster management | `kubectl`, Helm | Deploy and manage workloads and charts |
| Traffic generation | Locust (Python) | Repeatable browsing, item-view, and cart-addition traffic |
| Metrics | Prometheus | Time-series collection and query API |
| Metrics visualization | Grafana | Dashboards and environment validation |
| Metrics stack deployment | `kube-prometheus-stack` Helm chart | Installs Prometheus and Grafana components |
| Logs | Loki | Log storage and query API |
| Log shipping | Promtail | Ships application logs to Loki |
| Log stack deployment | `loki-stack` Helm chart | Planned Loki/Promtail deployment, subject to chart compatibility |
| Distributed tracing | Jaeger | Trace storage and query API |
| Fault injection | Chaos Mesh | Network, resource-stress, and pod faults |
| Data processing | Python | Telemetry collection, feature engineering, and evaluation |
| Log template extraction | Drain algorithm via `drain3` | Converts raw logs into structured templates |
| RCA method | Classical graph-based ranker | Ranks suspect services using anomalies and dependencies |
| Explanation | LLM | Generates structured, evidence-backed reports |
| Ground truth | CSV | Stores injected service, fault type, and time interval |

> **Compatibility note:** Verify chart versions and maintenance status before installation. Also confirm the telemetry format emitted by Online Boutique. Use Jaeger with Zipkin-format spans only if the application's instrumentation emits a compatible format; otherwise, configure a supported OpenTelemetry path. Do not assume trace collection works until it has been tested end to end.

## Root Cause Analysis Approach

The RCA system has two deliberately separated layers.

### 1. Classical graph-based ranker

The classical method performs the actual suspect-service ranking. Its planned workflow is:

1. Establish a steady-state baseline before injecting a fault.
2. Calculate anomaly scores for service-level telemetry, using z-scores against the baseline where appropriate.
3. Represent service dependencies using a call/dependency graph derived from application configuration and observed traces.
4. Propagate or combine anomaly evidence across the graph, following a graph-based approach inspired by systems such as MicroRCA and CloudRanger.
5. Produce a ranked list of suspect services.

The exact scoring and propagation rules should be documented alongside the implementation so the evaluation is reproducible.

### 2. LLM evidence-explanation layer

The LLM receives the ranked candidates and the structured evidence collected by the data pipeline. It explains the result and returns a structured report.

The LLM **does not choose or reorder the ranking**. This separation is intentional: it limits unsupported conclusions and enables a clean comparison between the classical-only and classical-plus-LLM configurations.

Planned report fields:

```json
{
  "faulty_service": "service-name",
  "confidence": 0.0,
  "evidence": [
    {
      "source": "metrics|logs|traces",
      "observation": "Observed signal supporting the diagnosis",
      "time_window": "start/end timestamp"
    }
  ],
  "suggested_next_step": "Read-only investigative recommendation"
}
```

This is an illustrative output schema. The implementation should define confidence semantics, validate output types, and ensure each evidence item points to data actually retrieved for the relevant fault window. The LLM may explain uncertainty; it must not invent missing evidence.

## Fault Injection and Ground Truth

Chaos Mesh will be used to inject controlled faults into the shared Kubernetes cluster.

### Planned fault families

| Fault family | Example | Purpose |
|---|---|---|
| `NetworkChaos` | Delay or packet loss | Simulates network degradation |
| `StressChaos` | CPU or memory stress | Simulates resource pressure |
| `PodChaos` | Kill a selected pod | Simulates pod disruption |

The custom Python script `inject.py` will:

1. Select a target service and fault template.
2. Record the intended scenario and start time.
3. Apply the fault through the approved Chaos Mesh/Kubernetes interface.
4. Allow a defined observation period while traffic continues.
5. Remove the fault and verify cleanup.
6. Record the end time and scenario outcome.
7. Write the ground-truth record to CSV.

### Ground-truth fields

At minimum, `ground_truth.csv` should record:

- Scenario/run ID
- Target faulty service
- Fault family and configuration
- Fault start and end timestamps
- Observation-window boundaries, if different from the fault interval
- Run status and any injection/cleanup errors

Use consistent timestamps and a documented time zone. Fault windows should account for propagation and recovery delays, while avoiding overlap with unrelated fault scenarios.

## Data Pipeline

The Python data pipeline runs outside the Kubernetes cluster and queries the telemetry APIs for each experiment's defined time window.

### Inputs and transformations

| Input | Source | Planned transformation |
|---|---|---|
| Metrics | Prometheus HTTP API | Per-service time series, baseline comparisons, and anomaly features |
| Logs | Loki API | Parse and normalize logs; extract templates using `drain3` |
| Traces | Jaeger API | Extract spans, service-to-service calls, and latency information |
| Ground truth | `ground_truth.csv` | Join telemetry windows with known fault labels |

### Expected outputs

The pipeline should create a structured, reproducible dataset containing:

- Scenario metadata and time windows
- Per-service metrics and anomaly scores
- Per-service log-template counts or related features
- Trace-derived call graph and latency features
- Ground-truth faulty service and fault type
- Data-quality indicators for missing or incomplete telemetry

Record query parameters and preprocessing choices so experiments can be reproduced. Handle missing telemetry explicitly instead of silently treating it as normal behavior.

## Evaluation

The evaluation harness compares predicted service rankings with the labels in `ground_truth.csv`.

### Metrics

| Metric | What it measures |
|---|---|
| Top-1 localization accuracy | Fraction of runs where the true faulty service ranks first |
| Top-3 localization accuracy | Fraction of runs where the true faulty service appears in the first three candidates |
| Mean Average Precision (MAP) | Quality of the ranked suspect lists across experiments |
| Time-to-diagnosis | Time from a defined fault/event reference point to a usable diagnosis |
| False-positive rate | Frequency of incorrectly flagging non-faulty services, using a clearly defined unit of analysis |

Define metric formulas and timing boundaries before running the final experiments. Report the number of runs and scenario breakdowns with every aggregate result.

### Comparisons

The project will compare:

1. **Threshold-alert baseline**: a simple rule-based baseline using defined telemetry thresholds.
2. **Classical-only RCA**: the graph-based ranking method without LLM explanations.
3. **Classical RCA + LLM**: the same ranking method with the evidence-grounded explanation layer.

### Ablation study

Evaluate the contribution of each telemetry source:

- Metrics only
- Metrics + logs
- Metrics + logs + traces

Keep fault scenarios, data windows, and evaluation procedures comparable across configurations. Report failures and missing telemetry rather than excluding them without explanation.

## Project Constraints

The following constraints are part of the project design.

- **Backend/infrastructure only:** No website or app-based project deliverable. The work remains focused on infrastructure, telemetry, diagnosis, and evaluation.
- **Read-only RCA agent:** The agent may query telemetry and analyze data but must not modify the cluster, inject faults, restart services, or apply fixes.
- **One shared cluster:** The team uses one shared Kubernetes cluster, not four disconnected environments.
- **Controlled scenarios:** Run single-fault and cascading-fault scenarios within the same cluster.
- **No unrelated simultaneous faults:** Do not inject multiple unrelated faults at the same time.
- **Ground-truth-driven evaluation:** Each experiment must have a recorded label and usable timestamps.
- **Diagnosis, not remediation:** The system reports the likely cause, evidence, confidence, and a suggested next investigative step. Any operational action remains a human decision.

## Team Responsibilities and Deadlines

Deadlines below are the planned milestones supplied for the project.

| Owner | Responsibilities | Deadline |
|---|---|---|
| You | Provision and configure the shared Kubernetes cluster (local/VM); plan resource allocation for Prometheus, Loki, Jaeger, and Chaos Mesh; set up secure multi-user access and SSH/network/port configuration; coordinate environment-level integration issues | 12 Oct |
| Person 2 | Deploy Online Boutique; verify service health; map the dependency graph; build the Locust traffic script | 14 Oct |
| Person 3 | Install and validate Prometheus, Grafana, Loki, and Jaeger; establish the steady-state baseline | 16 Oct |
| Person 4 | Install Chaos Mesh; build fault templates; automate fault injection; generate the first labelled dataset | 19 Oct |

Coordinate deployments and resource requests before installing the full stack, since telemetry and tracing components can be resource-intensive on a local cluster or small VM.

## Planned Repository Structure

The structure below is a suggested starting point and can be adjusted as implementation progresses.

```text
ai-agent-for-microservices-rca/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── experiment-protocol.md
│   └── evaluation-methodology.md
├── infrastructure/
│   ├── kubernetes/
│   ├── helm/
│   └── chaos-mesh/
├── traffic/
│   └── locustfile.py
├── fault-injection/
│   ├── inject.py
│   └── templates/
├── data-pipeline/
│   ├── collectors/
│   ├── processing/
│   └── schemas/
├── rca-agent/
│   ├── ranker/
│   ├── llm/
│   └── schemas/
├── evaluation/
│   ├── baselines/
│   ├── metrics/
│   └── run_evaluation.py
├── datasets/
│   └── README.md
└── tests/
```

Do not commit credentials, kubeconfig files, secrets, or sensitive cluster configuration. Keep large generated datasets and raw telemetry out of Git unless there is a deliberate, documented reason to version them.

## Getting Started

This repository describes the planned architecture. The following steps are the intended setup order; exact commands and configuration files should be added as the environment is validated.

### Prerequisites

- Kubernetes environment using Minikube or a suitable cloud VM
- `kubectl` configured for the shared cluster
- Helm
- Python environment for Locust, telemetry collection, fault orchestration, and evaluation
- Sufficient CPU, memory, and storage for the application and observability stack

### Setup sequence

1. Provision the shared cluster and establish secure team access.
2. Deploy Online Boutique and confirm that all expected services are healthy.
3. Start Locust traffic and verify that application behavior is repeatable.
4. Deploy Prometheus/Grafana, Loki/Promtail, and Jaeger or a validated OpenTelemetry-compatible tracing setup.
5. Confirm that metrics, logs, and traces can be queried for the same test requests and time intervals.
6. Record a steady-state baseline.
7. Install Chaos Mesh and validate each fault type in an isolated test run.
8. Implement `inject.py` and verify ground-truth timestamps and fault cleanup.
9. Build the telemetry collection and feature-generation pipeline.
10. Implement the classical ranker, then add the LLM explanation layer.
11. Run the evaluation baselines and ablation experiments.

**Safety and reliability:** Validate resource limits, access permissions, fault cleanup, and recovery procedures before running experiments on the shared cluster. Use only the designated test environment for fault injection.

## References

- [Google Online Boutique / microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)
- [Kubernetes documentation](https://kubernetes.io/docs/)
- [Minikube documentation](https://minikube.sigs.k8s.io/docs/)
- [Helm documentation](https://helm.sh/docs/)
- [Locust documentation](https://docs.locust.io/)
- [Prometheus documentation](https://prometheus.io/docs/)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Loki documentation](https://grafana.com/docs/loki/latest/)
- [Jaeger documentation](https://www.jaegertracing.io/docs/)
- [Chaos Mesh documentation](https://chaos-mesh.org/docs/)
- [Drain3](https://github.com/logpai/Drain3)

---

*Project status: architecture and implementation plan. Update this README as components are deployed, validated, and evaluated.*
