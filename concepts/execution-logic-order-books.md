# Execution Logic: Order Books

This page details the **[[concepts/execution-logic-order-books]]** layer of the system, which handles the granular mechanics of trade execution and the management of bid/ask queues. This component is the primary site where order matching occurs after a request has been routed by the `concepts/domain_orchestration_engine`.

## Overview
The Order Book is the core data structure responsible for maintaining the state of available liquidity for any given ticker symbol. While the `concepts/domain_orchestration_engine` handles multi-book management and routing, the **Execution Logic** layer focuses on:
- Maintaining price-time priority queues.
- Executing match logic between incoming orders and resting liquidity.
- Generating `TradeExecution` events for downstream systems.

## Bid/Ask Queue Management
Each `OrderBook` maintains two distinct sides of the market:
1.  **Bids (Buy Orders):** A queue of buy orders, typically ordered by price (descending) and time (ascending).
2.  **Asks (Sell Orders):** A queue of sell orders, typically ordered by price (ascending) and time (ascending).

In the current implementation (`internal/matching/order_book.go`), these are managed as slice-based structures for each ticker symbol. The system enforces a **Price-Time Priority** model to ensure that the highest-priced buy orders and lowest-priced sell orders are prioritized for execution.

## Matching Mechanics
When an order is submitted to the `concepts/execution_logic_order_books` layer, it undergoes a matching process based on its type:

### 1. Market Orders
Market orders are intended for immediate execution against available liquidity regardless of price. The system identifies these by the `OrderTypeMarket` flag. These orders "cross the spread" immediately to find a match.

### 2. Limit Orders
Limit orders include a specific price threshold (`OrderTypeLimit`). An order will only be executed if it can be filled at its specified price or better.

### Matching Logic Flow:
When `AddOrder` is called on an `OrderBook`:
- **Validation:** The system ensures the quantity is valid ($> 0$).
- **Execution Check:** If the incoming order is a Market Order, or a Limit Order with a price $> 0$, the engine attempts to find matching liquidity.
- **Trade Generation:** Upon a successful match, a `TradeExecution` object is created containing:
    - Unique Trade IDs and timestamps.
    - Matching details (Buyer/Seller IDs, Order IDs).
    - Status flags (e.g., "CONTINUOUS" match phase).
- **State Update:** Regardless of whether an immediate match occurred, the order is appended to its respective side of the book (Bid or Ask) if it remains unfilled.

## Integration & Data Flow
The transition from internal execution to external notification follows the `summaries/data_flow_pipeline`:

1.  **Order Entry:** The `Engine` receives an order and selects the correct `OrderBook`.
2.  **Match Processing:** The **Execution Logic** identifies matches and generates `TradeExecution` models.
3.  **Egress Handling:** Valid trades are passed to the `entities/kafka_publisher` via the `decisions/decoupling_strategy`, which abstracts the transport layer (Kafka) from the core matching logic.

## Technical Specifications
- **Concurrency:** The system utilizes a `sync.RWMutex` at the Engine level to ensure thread-safe access when selecting and updating specific order books.
- **Performance:** The engine is designed for high-throughput, aiming for microsecond-level execution during the matching phase of the lifecycle described in `summaries/order_matching_system`.

---
*See also:*
- [[index]] for general system overview.
- `summaries/order_matching_system` for architectural context.
- `decisions/decoupling_strategy` regarding how trade events are isolated from the core engine.