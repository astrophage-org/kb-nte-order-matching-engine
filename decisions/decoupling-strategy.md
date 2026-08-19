# Decisions: Infrastructure Decoupling Strategy

This document outlines the architectural decisions made to decouple the core trading logic from underlying infrastructure concerns within the `order-matching-system`. The primary goal of this strategy is to ensure that the high-performance matching engine remains resilient, maintainable, and independent of external messaging systems or hardware specifications.

## Core Philosophy: Separation of Concerns

A fundamental principle of this system's design, as outlined in the [[index]], is the strict isolation of domain logic from infrastructure implementation. This ensures that changes to the transport layer (e.g., switching from Kafka to another message broker) do not require modifications to the core matching algorithms.

## Key Decoupling Decisions

### 1. Abstracted Messaging Gateway
To prevent "leaky abstractions" where messaging protocols bleed into business logic, all external communications are handled by the **[[entities/kafka-publisher]]**. 

*   **Decision**: The `Engine` and `OrderBook` components never interact with the Kafka client library directly. Instead, they produce internal domain events (e.g., `models.TradeExecution`).
*   **Implementation**: The system utilizes an Egress & Messaging Gateway. This layer encapsulates all logic regarding serialization, producer configurations (`linger.ms`, `acks`), and connection management.
*   **Impact**: This allows the **`execution-logic-order-books`** to focus solely on bid/ask queue management without needing knowledge of Kafka topics or partitions.

### 2. Domain vs. Infrastructure Layers
The system is organized into distinct layers to isolate the logic described in [[summaries/order-matching-system]]:

*   **Orchestration Layer**: Handles lifecycle, configuration, and initialization.
*   **Domain Orchestration Engine**: Manages multi-book routing ([[concepts/domain-orchestration-engine]]). This layer acts as a buffer; it knows *what* a trade is, but not *how* it is transmitted to the outside world.
*   **Execution Logic**: The [[concepts/execution-logic-order-books]] focuses purely on matching mechanics and price-time priority.

### 3. Event-Driven Data Flow
The integration of **[[summaries/data-flow-pipeline]]** ensures that components are connected via events rather than direct dependencies. 

*   When an order is processed by the `OrderBook`, it produces a "Trade" event.
*   This event is handed off to the gateway for publication.
*   Because the system follows this linear, event-driven pipeline, the core engine can remain agnostic of downstream consumers such as the `trade-settlement-system` or `market-data-gateway`.

## Benefits Summary

| Feature | Decision | Impact on System |
| :--- | :--- | :--- |
| **Broker Independence** | Abstracting via [[entities/kafka-publisher]] | Allows swapping transport layers without touching matching logic. |
| **Scalability** | Decoupling via [[concepts/domain-orchestration-engine]] | Enables independent scaling of routing and execution logic. |
| **Maintainability** | Defined `summaries/data_flow_pipeline` | Simplifies debugging by isolating failures to specific layers (e.g., a network issue in the Gateway won't stall the internal Matching Engine). |

## Related Documents
*   Overview: [[index]]
*   Architecture Summary: [[summaries/order-matching-system]]
*   Flow Logic: [[summaries/data-flow-pipeline]]