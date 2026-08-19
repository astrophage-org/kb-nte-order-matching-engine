# Data Flow Pipeline

The data flow within the system is designed as a linear, event-driven pipeline to ensure high throughput and low latency while maintaining strict separation between core domain logic and infrastructure concerns. This architecture adheres to the principles of [[decisions/decoupling-strategy]], ensuring that any changes in downstream messaging technologies do not impact the execution core.

The lifecycle of an order from entry to publication follows these four primary stages:

## 1. Ingress & Normalization
Incoming trade requests enter the system through the ingress layer via the `nte.orders.inbound` Kafka topic. During this phase, raw data is validated against expected parameters and mapped into internal standardized models (e.g., `models.Order`). This process ensures that the core engine only processes well-formed, valid trading instructions as defined in the [[summaries/order-matching-system]] overview.

## 2. Routing & Orchestration
Once an order is normalized, it is passed to the `concepts/domain_orchestration_engine`. This component serves as the system's traffic controller:
*   **Symbol Resolution**: It identifies the correct ticker from the metadata.
*   **Book Mapping**: It routes the request to the specific matching instance corresponding to that symbol.
*   **State Management**: It ensures that new order books are instantiated and managed correctly within the multi-book environment.

## 3. Execution Logic
The order is then processed by the `concepts/execution_logic_order_books`. This layer handles the "heavy lifting" of the matching algorithm:
*   **Price-Time Priority**: The system matches orders based on price priority and time arrival.
*   **Trade Generation**: If an incoming order crosses a spread, a `TradeExecution` event is generated immediately.
*   **State Update**: The internal state of the bid/ask queues for that specific symbol is updated in memory to reflect the new market state.

## 4. Egress & Publication
After execution, successful match events are passed to the **Egress & Messaging Gateway**. This layer serves as a buffer between the inner matching logic and external infrastructure:
*   **Decoupling**: By utilizing the `entities/kafka_publisher`, the system abstracts away specific Kafka producer configurations (like retry policies or broker lists).
*   **Multi-Channel Broadcasting**: 
    *   Matched trades are published to `nte.trades.matched` for consumption by the Trade Settlement System and Compliance tools.
    *   Order book snapshots are published to `nte.orderbook.snapshots` for distribution via the Market Data Gateway.

This pipeline ensures that while the data flows linearly, each stage is strictly decoupled, allowing for scalable development as outlined in [[index]].