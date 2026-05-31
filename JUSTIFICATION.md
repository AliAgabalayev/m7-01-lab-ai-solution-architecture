# Justification — Real-time Fraud Scoring (Scenario A)

## Serving pattern

**Hybrid: streaming feature materialization + synchronous scoring.**

The scenario states p95 latency must stay under 80 ms end-to-end — the user is staring at a payment screen waiting for confirmation. That hard constraint rules out a fully async streaming path: publishing a transaction to Kafka and polling a reply topic adds two round-trip hops (10–40 ms each), making the budget fragile before the model has even run.

At the same time, account history — aggregate signals computed over days or weeks — cannot be joined on-the-fly at request time. Under 300 TPS, a live DB query would add 50–200 ms and saturate the transactional store. Computing it on the hot path is not viable.

The resolution: Kafka continuously materializes account history and device fingerprint signals into Redis. The Fraud API scoring path is synchronous and reads only pre-computed features (sub-millisecond from Redis). Streaming does the heavy lifting offline; the online path only reads.

## Where inference runs

**Cloud — centralized model serving cluster (Triton / TorchServe).**

Fraud patterns change quickly; models need to be swapped without redeploying the API. A dedicated serving cluster, updated via the Model Registry, enables zero-downtime model swaps. Capacity can also scale independently of the API tier during peak windows.

The cost of centralization is one additional internal network hop (~1–3 ms on the same VPC). That is acceptable within the 80 ms budget. An in-process model (embedded in the API) would eliminate the hop but couple model deployment to the service release cycle — unacceptable when new fraud vectors require a same-day model update.

Edge inference was rejected: the model requires features materialized from account history, which live in Redis. Replicating Redis to every edge node is operationally expensive and introduces consistency risk.

## Latency, throughput, and cost targets

**Optimize for: latency and throughput. Cost is the budget constraint.**

- **Latency:** p95 < 80 ms. Non-negotiable — the user is blocked on the decision. Budget breakdown: ~5 ms network/gateway, ~1 ms Redis read, ~20–30 ms model inference, ~10 ms API overhead, ~35 ms headroom.
- **Throughput:** 300 TPS sustained peak. The serving cluster and Redis must be provisioned for this; auto-scaling handles daily variance without paying for 300 TPS capacity 24/7.
- **Cost:** Over-provisioning for 300 TPS around the clock is the main waste risk. Horizontal auto-scaling on the model serving cluster and Redis read replicas keeps cost proportional to actual load.

## Fallback

When the model serving cluster is unavailable or the circuit breaker trips (timeout > 60 ms), the Fraud API falls back to a deterministic rules engine running in-process:

- Same card > 3 transactions in 60 seconds → **block**
- New device fingerprint + amount > threshold → **step-up auth**
- Otherwise → **allow**

Rules execute in < 2 ms with no external dependency. The fallback is lower accuracy than the model but retains the three-action contract the downstream consumers expect. A sustained model outage pages on-call; rules bridge the gap until recovery.
