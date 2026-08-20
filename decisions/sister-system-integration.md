# ADR: Decoupling OME from Post-Trade Mechanics via Kafka Topics

## Status
Approved

## Context
The Nexus Trading Exchange (NTE) Order Matching Engine (OME) is engineered for ultra-low latency execution, processing inbound orders and maintaining fine-grained [[entities/order-book]] states via the [[entities/engine]]. In a high-throughput financial exchange, tight coupling between core matching logic and downstream post-trade workflows—such as clearing, settlement, market data distribution, and regulatory surveillance—introduces severe bottlenecks, blocking risks, and tight failure domains.

To protect the microsecond-level performance goals of the matching core, the architecture must completely isolate the OME from the operational overhead of sister systems within the Global Financial Markets Group (GFMG) ecosystem.

## Decision
We decouple the Order Matching Engine from post-trade mechanics by implementing an asynchronous, event-driven integration layer using Apache Kafka topics via the [[entities/kafka-publisher]]. 

Specifically:
- **Trade Execution Broadcasting**: Matched trade reports (`TradeExecution`) are marshaled and published exclusively to the `nte.trades.matched` Kafka topic, allowing systems like the [[decisions/sister-system-integration|Trade Settlement System (TSS)]] and Compliance Surveillance monitors to ingest execution records independently.
- **Market Data Snapshots**: Level 2 (L2) book snapshots and state mutations are partitioned by symbol and published to the `nte.orderbook.snapshots` topic for downstream consumers such as the [[decisions/sister-system-integration|Market Data Gateway (MDG)]].
- **Asynchronous Producer Tuning**: Using optimized Confluent Kafka producer parameters (`acks: "all"`, `linger.ms: 1`), the [[entities/kafka-publisher]] balances transactional durability with ultra-low-latency event dispatch.
- **Graceful Fallback**: The server lifecycle (`cmd/server/main.go`) initializes event publishing with built-in resilience, ensuring that temporary Kafka cluster unavailability does not halt internal matching operations during standalone or development runs (see [[decisions/resilient-event-integration]]).

## Consequences
- **Positive**:
  - Eliminates direct blocking network calls from the core matching loop to downstream databases or sister systems.
  - Enables independent horizontal scaling of post-trade clearing, risk engines, and market data feeds.
  - Enhances fault isolation; downstream processing failures do not impact order matching availability.
- **Negative**:
  - Introduces eventual consistency models for downstream consumers requiring careful offset management and idempotent handling in sister systems.
  - Requires reliable message serialization and schema management across organizational boundaries.