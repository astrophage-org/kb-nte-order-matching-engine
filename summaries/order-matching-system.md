# Architectural Overview: Order Matching System

The [[Order Matching System]] is a high-performance, decoupled engine designed to process and execute financial trades with a primary focus on reliability and thread safety. The system utilizes a layered architecture to strictly separate core domain logic from infrastructure concerns, ensuring that the core matching algorithms remain isolated from external dependencies.

## Core Components

The system is structured into four primary architectural layers:

1.  **[[Orchestration & Lifecycle Layer]]**: This serves as the central entry point for the application. It manages the entire lifecycle of the [[Order Matching System]], including initialization, the loading of configuration data via `[[Configuration Management]]`, and graceful shutdown procedures. It ensures that all necessary dependencies, such as messaging clients, are fully initialized before the system begins processing orders.
2.  **[[Domain Orchestration Engine]]**: Functioning as a multi-book management system, this component acts as the primary router for the application. It maintains state for various trading symbols and routes incoming requests to the appropriate [[Order Books]] based on specific ticker metadata.\n3.  **[[Execution Logic (Order Books)]]**: This layer handles the granular specific mechanics of trade execution. Each order book manages its own bid/ask queues, processing orders that include advanced features such as **Iceberg Orders**. The matching logic determines fill quantities and handles automatic replenishment of hidden portions for Iceberg types.
4.  **[[Egress & Messaging Gateway]]**: To maintain a decoupled architecture, this gateway serves as an abstraction layer for message production. It encapsulates all logic related to the [[Kafka Publisher]], ensuring that internal match events are serialized and broadcasted without exposing the core engine to specific messaging broker protocols.

## Data Flow Pipeline

The system processes data through a linear, event-driven pipeline:

*   **Ingress**: Raw trading requests are captured and mapped into standardized internal models. This includes support for complex order types like **Iceberg Orders**, where only a portion of the total quantity is visible to the market.\n*   **Routing & Execution**: Orders are passed to the [[Domain Orchestration Engine]], which identifies the correct book. The order is then processed by the specific [[Execution Logic (Order Books)]]. During execution, Iceberg orders undergo replenishment logic: if a displayed portion is exhausted, a new slice from the hidden quantity is revealed.\n*   **Publication**: Upon a successful match, a "Trade" event is generated and handed off to the [[Egress & Messaging Gateway]], which broadcasts the data to external systems for downstream processing via Kafka.\n
## Architectural Principles

The design of the [[Order Matching System]] is guided by several key principles:

*   **Decoupling via Abstraction**: By utilizing an abstraction layer for the [[Kafka Publisher]], the core matching logic remains independent of downstream infrastructure changes.
*   **Thread-Safe Concurrency**: The system is engineered to handle concurrent requests across multiple symbols simultaneously while maintaining integrity in the order books.\n*   **Domain-Driven Design (DDD)**: A clear separation of concerns ensures that specific domain logic, like Iceberg matching mechanics, are encapsulated within the execution layer.",
