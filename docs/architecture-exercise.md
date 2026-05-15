# Low-Latency Crypto Trading Platform – Architecture Exercise

## Overview

The current trading system runs the entire pipeline on a single server using file-based communication between stages:

* `get_data.py` → fetches market data
* `data_analysis.py` → feature engineering
* `signal_calc.py` → trading signal generation
* `live_trading.py` → order execution

Each stage communicates through CSV/Parquet files (File A → B → C → D).

While this approach is simple and easy to debug, it introduces significant latency, scalability, reliability, and operational limitations for a real-time trading system.

The goal of this proposal is to evolve the architecture into a low-latency, observable, and operationally safe trading platform while remaining pragmatic for a two-engineer team.

---
# Target Architecture
```text
Binance WebSocket
        ↓
Market Data Service
        ↓
NATS
        ↓
Feature Engineering
        ↓
Signal Engine
        ↓
Risk Engine
        ↓
Execution Engine (Rust)
        ↓
Binance Exchange

Monitoring:
Prometheus + Grafana + PagerDuty


---

# 1. Latency Optimization

## Current Latency Bottlenecks

### 1. File-Based Communication (Largest Bottleneck)

Current flow:

```text
get_data.py → writes File A
↓
data_analysis.py → reads File A → writes File B
↓
signal_calc.py → reads File B → writes File C
↓
live_trading.py → reads File C
```

Problems:

* Disk I/O latency
* Serialization/deserialization overhead
* Blocking between stages
* File locking and synchronization overhead
* Increased garbage collection pressure in Python
* Sequential processing pipeline

Estimated improvement:

* File I/O → in-memory messaging can reduce latency from milliseconds to microseconds.

---

### 2. Sequential Pipeline Execution

Each stage waits for the previous stage to complete.

This creates cumulative latency:

```text
Market data → feature engineering → signal generation → order execution
```

Instead of streaming continuously.

---

### 3. Single Server Resource Contention

All workloads share:

* CPU
* memory
* disk I/O
* network bandwidth

This creates unpredictable latency spikes during:

* market volatility
* backfills
* logging spikes
* GC pauses

---

### 4. Geographic Distance from Binance

Binance Spot and USD-M Futures matching engine runs in:

```text
AWS Tokyo (ap-northeast-1)
```

If infrastructure is deployed outside Tokyo:

* additional network RTT directly impacts execution quality
* slippage increases
* order execution worsen during volatility

---

### 5. REST Polling

Polling market data via REST introduces:

* additional latency
* unnecessary bandwidth
* rate limiting risk

WebSocket streams should be preferred.

---

## Recommended Low-Latency Improvements

### Immediate Improvements (Highest ROI)

### Replace Flat Files with In-Memory Messaging

Recommended options:

* Redis Streams
* Replace File I/O with NATS
* shared memory ring buffers

Recommendation:

```text
NATS for event transport
```

Reason:

* lightweight
* very low latency
* operationally simple for a small team
* easier than Kafka initially

Replacing file-based communication with in-memory event streaming using NATS significantly reduces latency, removes disk I/O overhead, and provides a lightweight low-operational-overhead messaging system suitable for a small trading engineering team.
---

### Move Infrastructure to AWS Tokyo

Deploy trading infrastructure in:

```text
ap-northeast-1
```

Expected gain:

* Significant RTT reduction to Binance
* Lower order latency
* Better fill quality

Deploying infrastructure in AWS Tokyo reduces network distance to Binance’s matching engine, which lowers communication latency, improves execution speed, and helps achieve better trade prices during fast market movement.

---

### Use WebSocket Market Streams

Replace:

```text
REST polling
```

With:

```text
Binance WebSocket feeds
```

Benefits:

* real-time streaming
* lower latency
* reduced API usage

---

### Separate Critical Trading Path

Split architecture into:

#### Hot Path (Low Latency)

* market data
* signal generation
* execution

#### Cold Path

* analytics
* dashboards
* reporting
* historical storage

This prevents analytics workloads from impacting trading latency.

---

## Advanced Optimizations (Later Phases)

### Rewrite Execution Engine in Rust

Current Python execution logic can introduce:

* GC pauses
* unpredictable latency
* lower concurrency efficiency

Recommendation:

```text
Keep research stack in Python
Rewrite execution path in Rust
```

Reason:

* memory safety
* low latency
* predictable performance
* excellent concurrency

---

### Kernel and Networking Optimizations

Not required in phase 1, but valuable later:

* CPU pinning
* isolated cores
* IRQ affinity
* hugepages
* PTP time sync
* AF_XDP / DPDK for ultra-low-latency networking

These should only be introduced after eliminating larger bottlenecks. These are advanced kernel and networking optimizations used to reduce latency jitter and improve determinism in low-latency trading systems, but they should only be introduced after eliminating larger architectural bottlenecks such as file I/O and inefficient networking patterns.

---

### FIX Protocol

If supported operationally:

* FIX sessions can provide lower latency and more deterministic order flow compared to REST.

However, REST/WebSocket APIs are acceptable initially given team size.

---

# 2. Tech Stack Migration

## Recommended Architecture

### Market Data Layer

Current:

```text
get_data.py
```

Proposed:

* Python ingestion service
* Binance WebSocket feeds
* Publish normalized events into NATS

Reason:

* Python is sufficient for ingestion
* easier iteration
* lower engineering overhead

---

### Feature Engineering

Keep in:

```text
Python
```

Reason:

* excellent ecosystem for quant analysis
* pandas/numpy ecosystem
* fast iteration for strategy development

Move communication away from files.

---

### Signal Engine

Phase 1:

* Python acceptable

Later:

* move latency-sensitive strategies to Rust

---

### Execution Engine (Critical Path)

Recommended:

```text
Rust
```

Reason:

* deterministic latency
* strong concurrency
* memory safety
* lower runtime overhead than Python

Execution engine responsibilities:

* order placement
* risk checks
* retries
* idempotency
* kill switch handling

---

## Inter-Process Communication

### Recommendation: NATS

Why NATS:

* low operational overhead
* simpler than Kafka
* very low latency
* excellent for event-driven systems
* suitable for small teams

Use cases:

* market events
* signal events
* execution acknowledgements
* risk alerts

---

## Time-Series Storage

### Recommendation

#### Hot Data

* Redis

#### Historical Analytics

* ClickHouse or TimescaleDB

Reason:

Trading systems generate high-volume time-series data.

Need:

* fast writes
* efficient aggregations
* historical backtesting

ClickHouse is especially strong for analytical workloads.

---

## Message Bus

### Phase 1

```text
NATS
```

### Phase 2+

Potentially Kafka if:

* event volume increases significantly
* replay guarantees become critical
* multiple downstream consumers emerge

Avoid Kafka initially due to operational complexity.

---

## Containers vs Bare Metal

### Recommendation

#### Hot Path

* Dedicated EC2 instances
* Avoid Kubernetes initially for execution path

Reason:

* lower jitter
* simpler debugging
* fewer networking layers
* more deterministic latency

#### Supporting Services

Can run in containers:

* dashboards
* monitoring
* analytics
* CI/CD tooling

---

## Networking Recommendations

### Phase 1

* optimized EC2 placement
* enhanced networking enabled
* Tokyo deployment

### Later Phases

* CPU pinning
* PTP clock synchronization
* AF_XDP
* DPDK

These should only be introduced after higher ROI improvements.

---

# 3. CI/CD Design

## Goals

The trading platform manages real capital.

The deployment pipeline must prioritize:

* safety
* rollback capability
* observability
* risk control

Over deployment speed.

---

## CI/CD Pipeline

```text
GitHub
↓
Build
↓
Unit Tests
↓
Integration Tests
↓
Backtesting
↓
Paper Trading
↓
Canary Deployment
↓
Production
```

---

## Source Control

Recommended:

* GitHub
* protected main branch
* mandatory PR reviews
* signed commits

---

## Testing Pyramid

### 1. Unit Tests

Validate:

* indicator logic
* signal calculations
* risk rules
* utility functions

---

### 2. Integration Tests

Validate:

* exchange connectivity
* order lifecycle
* event flow
* retries and reconnections

---

### 3. Backtesting

Run strategy against:

* historical Binance data

Validate:

* Sharpe ratio
* drawdown
* win rate
* slippage assumptions

---

### 4. Paper Trading

Run strategies without real capital.

Critical before production rollout.

---

### 5. Canary Deployment

Deploy strategy to:

* small capital allocation
* limited symbols

Monitor:

* PnL
* latency
* error rates
* risk exposure

Before full rollout.

---

## Deployment Strategy

### Blue/Green or Canary

Avoid deploying directly to all trading instances.

Safer approach:

```text
Old strategy stays active
New strategy receives limited traffic/capital
```

---

## Rollback Strategy

Rollback triggers:

* abnormal losses
* increased latency
* order rejection spikes
* strategy divergence

Rollback should be:

* automated where possible
* executable within seconds

---

## Kill Switches (Critical)

System must support:

* emergency global stop
* symbol-level disable
* strategy-level disable
* exchange disconnect protection

Examples:

* max daily loss exceeded
* excessive leverage
* exchange instability
* abnormal order volume

---

## Risk Controls

Mandatory controls:

* max position size
* max leverage
* max daily drawdown
* max open orders
* rate limiting
* duplicate order prevention

Risk checks should execute BEFORE order submission.

---

## Observability

Critical metrics:

* order latency
* exchange RTT
* rejected orders
* PnL
* strategy performance
* queue lag
* websocket disconnects

Recommended stack:

* Prometheus
* Grafana
* Loki/OpenSearch
* PagerDuty alerts

---

# 4. Deployment Roadmap

## Guiding Principle

Avoid rewriting everything immediately.

The highest-value improvements should be delivered first while minimizing operational risk.

---

# Phase 1 (Weeks 1–4)

## Goals

* remove biggest latency bottlenecks
* improve reliability
* improve observability

## Deliverables

### Replace File-Based Communication

Move:

```text
CSV/Parquet
```

To:

```text
NATS or Redis Streams
```

---

### Deploy Infrastructure in Tokyo

Move workloads to:

```text
AWS ap-northeast-1
```

---

### Introduce Monitoring

Deploy:

* Prometheus
* Grafana
* centralized logging
* PagerDuty alerts

---

### Implement Risk Controls

Add:

* kill switches
* max position checks
* daily loss limits

---

## What NOT to do in Phase 1

Avoid:

* Kubernetes migration
* Rust rewrite
* DPDK
* microservice explosion
* Kafka cluster

Reason:

The largest wins come from eliminating file I/O and improving deployment location.

---

# Phase 2 (Weeks 5–8)

## Goals

* separate services
* improve scalability
* improve deployment safety

## Deliverables

* separate ingestion/execution services
* canary deployments
* paper trading environment
* integration testing pipeline
* Redis hot cache

---

# Phase 3 (Weeks 9–12)

## Goals

* optimize critical execution path
* reduce latency jitter

## Deliverables

* rewrite execution engine in Rust
* CPU pinning
* dedicated execution hosts
* isolated cores for execution process

---

# Phase 4 (Weeks 13–16)

## Goals

* advanced optimization
* long-term scalability

## Deliverables

* AF_XDP / DPDK experimentation
* advanced replay systems
* strategy isolation
* multi-region DR planning
* more advanced event streaming if needed

---

# Final Recommendation

The largest practical improvements for this trading system are:

1. Eliminate file-based communication
2. Move infrastructure to Tokyo
3. Introduce strong observability and risk controls
4. Separate the execution path from analytics workloads
5. Gradually migrate latency-sensitive components to Rust

The proposed approach intentionally prioritizes:

* operational simplicity
* low engineering overhead
* reliability
* measurable latency gains

over premature ultra-HFT optimizations.

This provides a realistic path toward a production-grade low-latency trading platform operable by a small engineering team.
