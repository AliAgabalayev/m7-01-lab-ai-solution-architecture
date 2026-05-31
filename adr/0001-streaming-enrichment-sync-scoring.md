# ADR 0001: Streaming Feature Materialization with Synchronous Scoring

## Context

The fraud scorer must return a block/allow/step-up decision before the user sees payment confirmation, imposing a hard p95 latency budget of 80 ms. Scoring accuracy depends on account history — aggregate signals computed over days or weeks — that cannot be joined on-the-fly from transactional storage under 300 TPS without blowing the budget.

## Decision

Account history and device fingerprint signals are continuously materialized into Redis by a Kafka-driven feature enrichment pipeline. The Fraud API scoring path is synchronous: it reads pre-computed features from Redis and calls the model serving cluster. No joins against transactional storage occur on the hot path.

## Alternatives rejected

- **Pure async streaming (Kafka request-reply):** Publishing the transaction and polling a reply topic introduces two Kafka round-trip hops (~10–40 ms each). The 80 ms budget becomes fragile, and the operational overhead of correlation-ID matching adds complexity with no accuracy gain.
- **Synchronous scoring with live DB joins:** Computing account history on-the-fly at request time would add 50–200 ms under 300 TPS load, exceeding the latency budget before the model even runs.
- **Batch pre-scoring (score accounts ahead of time):** Impossible — the fraud score must incorporate the transaction itself. A pre-computed account risk score cannot substitute for per-transaction scoring.

## Consequences

- The feature enrichment pipeline becomes a first-class production service. It must maintain low lag (target: features stale by < 30 seconds) and be monitored for pipeline backpressure and lag.
- Feature staleness is an accepted trade-off: account history reflects the state as of the last pipeline write, not the exact millisecond of the transaction. A burst of fraudulent transactions can partially evade detection until the enrichment pipeline catches up.
- Redis must be treated as critical infrastructure: replication, persistence policy (RDB or AOF), and eviction policy must be configured and monitored. It is not a cache — data loss means degraded scoring until the pipeline backfills.

## Revisit if

The feature staleness window (seconds-level lag) causes measurable fraud leakage that exceeds business tolerance — at that point, consider passing the raw current transaction alongside the materialized features directly to the model, so it can reason about the present event without depending on the pipeline having already processed prior events.
