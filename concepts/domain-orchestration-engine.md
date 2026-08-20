# Domain Orchestration Engine

The **`concepts/domain-orchestration_engine`** serves as the core routing layer within the [[summaries/order-matching-system]]. It functions as a multi-book management system, acting as the primary gateway that translates incoming high-level orders into specific execution tasks.

## Overview
As part of the broader {{index}}, the Domain Orchestration Engine is responsible for isolating and organizing trading activity by symbol (ticker). Instead of processing all orders in a single monolithic queue, this engine ensures that each ticker has its own dedicated environment, ensuring scalability and thread-safety across multiple concurrent markets.

The core responsibilities include:
- **Multi-book Management**: Maintaining a registry of individual order books for various symbols.
- **Routing Logic**: Identifying the correct book based on symbol metadata provided in the ingress data.
- **Dynamic Lifecycle Management**: Automatically instantiating new `OrderBook` instances when a previously unseen ticker is encountered.

## Routing & Execution Flow
The engine sits at the heart of the `summaries/data_flow_pipeline`. When an order enters the system, the Orchestration Engine performs the following steps:

1. **Symbol Identification**: Extracts the symbol from the incoming request.
2. **Route Selection**: Queries the internal map to find the corresponding `OrderBook`. 
3. **Just-in-Time Provisioning**: If a book for a specific symbol does not exist, the engine initializes it dynamically.
4. **Handoff**: Once the correct book is identified, the order is passed to the `concepts/execution_logic_order_books` layer for matching and trade generation.

## Architectural Significance
By decoupling the routing logic from the execution logic, the system achieves several key goals outlined in the `decisions/decoupling_strategy`:

*   **Concurrency**: By sharding data by symbol at the orchestration level, the system minimizes lock contention, allowing multiple orders for different symbols to be processed simultaneously.
*   **Isolation**: Failures or bottlenecks in one `OrderBook` (e.g., a high-volume ticker) do not impact the performance of other tickers managed by the same engine.
*   **Abstraction**: The engine ensures that the outer layers of the system do not need to know the internal mechanics of trade matching; they only need to interact with the orchestration layer to route an order.

## Data Integration
Once the `concepts/execution_logic_order_books` completes a match, the resulting trades are passed toward the **`entities/kafka_publisher`**. The Orchestration Engine’s role ends at successful routing and execution, ensuring that the core logic remains focused on matching rather than downstream message broadcasting.

## Technical Summary
- **Core Component**: `Engine` struct in `internal/matching/engine.go`.
- **Concurrency Model**: Uses `sync.RWMutex` to manage thread-safe access to the map of order books.
- **Scaling Mechanism**: Multi-book sharding by ticker symbol.