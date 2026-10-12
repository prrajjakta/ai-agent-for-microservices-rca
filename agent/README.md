# Root Cause Analysis Agent

An evidence-grounded AI agent for diagnosing failures
in microservice-based cloud systems.

## Architecture
1. Classical graph-based ranker identifies and ranks
   suspected root-cause services.
2. LLM reasoning layer explains the ranking using
   structured telemetry evidence.
3. Evaluation harness compares predictions against
   ground-truth fault labels.

## Responsibilities
- Implement baseline comparison and anomaly scoring.
- Rank suspected services using dependency information.
- Generate structured, evidence-backed diagnostic reports.
- Include confidence, supporting evidence, and suggested
  next investigative steps.
- Keep the agent read-only: it must never modify
  the cluster or automatically fix failures.
