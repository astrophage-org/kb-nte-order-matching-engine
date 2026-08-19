# Kafka Publisher

The **[[entities/kafka-publisher]]** serves as the core implementation of the **`Egress & Messaging Gateway`** layer within the [[summaries/order-matching-system]]. It provides a high-level abstraction for broadcasting internal system events to external consumers while isolating the core matching logic from infrastructure-specific details.

## Role in Architecture
In accordance with the [[decisions/decoupling-strategy]], the Kafka Publisher ensures that the **`concepts/execution_logic_order_books`** and the **`concepts/domain-orchestration_engine`** remain "pure." These components do not interact with Kafka directly; instead, they produce internal events which are then passed to the publisher.

This abstraction allows the system to:
1.  Isolate infrastructure failures (e.g., network issues with a broker) from the execution of the matching engine.
2.  Facilitate easier testing by allowing for mock implementations of the publisher during CI/CD.
3.  Provide a centralized point for event serialization and mapping to external schemas.

## Data Flow & Pipeline
The component is the final stage in the **`summaries/data_flow_pipeline`**. Once an order is successfully processed through the `concepts/domain-orchestration_engine`, any resulting trade executions or state changes are passed to this entity for publication.

## Technical Specifications

### Broker Configuration
The producer is initialized with specific configurations optimized for high-performance financial data:
*   **Acknowledge (acks):** Set to `all` to ensure maximum reliability for trade settlement.
*   **Linger.ms:** Optimized for low latency while allowing for slight batching of events.
*   **Partitioning:** Snapshots are partitioned by the asset symbol to maintain order and locality for downstream consumers.

### Broadcast Methods

#### Trade Publication (`PublishTrades`)
This method broadcasts successful match results to the `nte.trades.matched` topic. 
- **Internal Source:** `models.TradeExecution`
- **Downstream Consumers:** 
    - `trade-settlement-system` (for post-trade clearing)
    - `compliance-surveillance-monitor` (for fraud and wash-trading detection)

#### Snapshot Publication (`PublishSnapshot`)
This method broadcasts L2 order book snapshots to the `nte.orderbook.snapshots` topic.
- **Internal Source:** Generic snapshot object/map.
- **Downstream Consumers:** 
    - `market-data-gateway` (for broadcasting market depth to clients).

## Implementation Details
The `KafkaPublisher` struct encapsulates the Confluent Kafka Go client, handling the translation of internal domain models into JSON payloads. By wrapping the producer logic here, the system maintains a clean separation between business rules and messaging protocols.

See **[[index]]** for more information on the overall high-level architecture and how this entity fits within the broader ecosystem.