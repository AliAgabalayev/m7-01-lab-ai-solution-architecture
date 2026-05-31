# Architecture — Real-time Fraud Scoring

```mermaid
flowchart TD
    subgraph ONLINE["⚡ Online — p95 < 80 ms"]
        PA[Payment App]
        FAPI[Fraud API]
        MSC["Model Serving Cluster\nTriton / TorchServe"]
        FS["Feature Store\nRedis"]
        RE["Rules Engine\n(fallback)"]
    end

    subgraph STREAM["🔄 Streaming — continuous enrichment"]
        KAFKA["Kafka\nTransaction Events"]
        FEP["Feature Enrichment Pipeline\n(account history, device fingerprint)"]
    end

    subgraph OFFLINE["📦 Offline — batch"]
        LABELS["Labeled Events Store\n(fraud outcomes)"]
        TRAIN["Training Pipeline"]
        MR["Model Registry\nMLflow"]
        MON["Monitoring & Drift Detection"]
    end

    DS["Downstream Actions\nblock / allow / step-up auth"]

    PA -->|"HTTP request\n(transaction)"| FAPI
    PA -->|async event| KAFKA
    KAFKA -->|event stream| FEP
    FEP -->|"feature write\n(< 30 s lag)"| FS
    FAPI -->|feature read| FS
    FAPI -->|score request| MSC
    MSC -->|fraud score| FAPI
    FAPI -.->|"fallback\n(circuit breaker)"| RE
    FAPI -->|decision| PA
    PA -->|action| DS
    FAPI -->|decision event| KAFKA
    KAFKA -->|decision + outcome stream| MON
    KAFKA -->|labeled outcomes| LABELS
    MON -->|"drift signal /\nretrain trigger"| TRAIN
    LABELS -->|training data| TRAIN
    TRAIN -->|new model version| MR
    MR -->|"model deployment\n(zero-downtime swap)"| MSC
```

## Serving boundary

| Zone | Components | Latency target |
|---|---|---|
| **Online** | Payment App → Fraud API → Feature Store + Model Serving Cluster | p95 < 80 ms |
| **Streaming** | Kafka → Feature Enrichment Pipeline → Redis | Features stale < 30 s |
| **Offline** | Training Pipeline, Model Registry, Monitoring | Hours to days |

## Latency budget (80 ms)

| Hop | Allocation |
|---|---|
| Network + API gateway | ~5 ms |
| Redis feature read | ~1 ms |
| Model Serving Cluster inference | ~20–30 ms |
| API overhead + serialization | ~10 ms |
| Headroom / queuing | ~35 ms |
