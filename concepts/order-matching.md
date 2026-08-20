# Core Matching Logic and Price-Time Priority

The Nexus Trading Exchange (NTE) Order Matching Engine (OME) relies on a deterministic, low-latency execution model designed around **price-time priority** (also known as FIFO - First In, First Out matching). This document outlines the core algorithms, state management rules, and specialized order handling implemented across the matching core (`[[concepts/order-matching]]`).

---

## 1. Overview of Price-Time Priority

Price-time priority ensures fair and predictable execution across all participants. The matching engine evaluates inbound orders against existing liquidity residing in the Limit Order Book ([[entities/order-book]]) using two primary rules:
1. **Price Priority:** Better-priced orders take precedence over worse-priced orders. For buy orders (`models/order`), higher prices have priority; for sell orders, lower prices have priority.
2. **Time Priority:** Orders at the exact same price level are executed in the chronological order of their arrival (`Timestamp`).

---

## 2. Execution Flow and Spread Crossing

When an inbound order is received by the `[[entities/engine]]`, it is routed to the corresponding symbol's `[[entities/order-book]]` via `OrderBook.AddOrder()`. 

The lifecycle of an inbound order follows these programmatic steps:
1. **Validation:** Checks that the order `Quantity` is strictly greater than zero. Invalid orders are rejected.
2. **Iceberg Pre-Processing:** If `IsIceberg` is set to `true`, the engine splits the total order size into visible and hidden allocations (see [[#3. Iceberg Order Handling]]).
3. **Matching & Spread Evaluation:** If the order crosses the market spread (e.g., a `MARKET` order or a `LIMIT` order with a matching cross price), liquidity is consumed, generating one or more `[[entities/domain-models]]` (`TradeExecution`) objects.
4. **Book Insertion:** Unfilled portions of the order (or resting limit orders that do not immediately cross) are appended to the internal `bids` or `asks` queues, securing their time priority.

---

## 3. Iceberg Order Handling

To support institutional trading strategies that require large order sizing without immediate market impact disclosure, the OME supports **Iceberg orders** directly within the matching core (`[[entities/order-book]]`).

* **Visible vs. Hidden Slices:** An iceberg order contains a total `Quantity` alongside a `DisplayQuantity` (which defaults to 10% of the total order size if unassigned). The remainder is tracked as `HiddenQuantity`.
* **Replenishment Mechanism:** When visible liquidity on the order book is fully consumed by matching trades, the engine automatically replenishes the display quantity from the remaining `HiddenQuantity` pool, maintaining its original time priority timestamp or slotting it back according to exchange configuration.

---

## 4. Concurrency and Synchronization

Because the OME services high-throughput traffic across multiple trading symbols (e.g., `BTC/USD`), the architecture decouples execution state per instrument using fine-grained locking (`[[decisions/memory-driven-concurrency]]`). 

* The `[[entities/engine]]` manages a thread-safe map of order books protected by a `sync.RWMutex`.
* Individual order books handle localized matching queues, minimizing contention across distinct trading pairs and maximizing CPU cache locality.

---

## 5. Downstream Event Broadcast

Once trades are executed by the matching logic, they are immediately packaged into immutable execution reports and dispatched asynchronously via the `[[entities/kafka-publisher]]` to downstream sister systems like the *Trade Settlement System* and *Compliance Surveillance Monitor* ([[decisions/sister-system-integration]]).