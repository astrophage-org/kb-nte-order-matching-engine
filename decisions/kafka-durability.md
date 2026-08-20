# Resiliency and Kafka-Based Event Log Durability

## Status
Accepted

## Context
The [[summaries/order-matching-engine|Order Matching Engine (OME)]] processes high volumes of financial transactions with microsecond-level performance requirements. To achieve this, [[entities/internal-matching-engine]] and [[entities/internal-matching-order-book]] maintain the state of Limit Order Books (LOBs) entirely in memory using price-time priority queues ([[decisions/in-memory-queues|In-Memory Priority Queues for Microsecond Latency]]). 

While in-memory storage provides optimal execution speed, it introduces a vulnerability to node crashes, hardware failures, or network partitions. Because financial markets demand strict durability, fault tolerance, and deterministic recoverability, the architecture requires an explicit resiliency strategy. State must be preserved without compromising the low-latency execution path of the matching loop managed by [[entities/cmd-server-main]].

## Decision
We utilize Apache Kafka as an immutable, append-only event log to guarantee message durability and state recovery. The resiliency architecture is structured around the following core mechanisms:

1. **Write-Ahead Event Sourcing via Inbound Logs**: Upstream ingress routes flow through durable Kafka topics (such as `nte.orders.inbound`), ensuring all orders are safely committed to disk across broker clusters before or concurrent with matching operations.
2. **Producer Durability Tuning**: As implemented in [[entities/internal-events-kafka-publisher]], the Kafka producer is configured with maximum durability parameters:
   * `"acks": "all"`: Ensures that messages are fully acknowledged by all in-sync replicas (ISRs) before writes are considered successful.
   * `"linger.ms": 1`: Introduces a minimal batching window to optimize network throughput while maintaining near-zero latency overhead for outbound events like matched trades (`nte.trades.matched`) and snapshots (`nte.orderbook.snapshots`).
3. **Offset Replay Recovery**: In the event of an OME node failure or restart, system state is reconstructed deterministically by replaying Kafka offsets from the beginning of the active trading session.
4. **End-of-Day (EOD) Object Storage Snapshots**: To prevent unbounded offset replay times during startup, periodic and EOD snapshots of the order books are persisted to cloud object storage (S3/GCS). Recovery proceeds by loading the latest daily snapshot and replaying only subsequent intraday events.

This design decouples transient memory state from ultimate ledger durability, allowing the trading engine to maintain microsecond execution speeds while satisfying regulatory compliance and disaster recovery standards.