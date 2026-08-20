# In-Memory Priority Queues for Microsecond Latency

## Status
**Accepted**

## Context
The Nexus Trading Exchange (NTE) Order Matching Engine (OME) requires microsecond-level execution guarantees to remain competitive in high-frequency global financial markets managed under the Global Financial Markets Group (GFMG). Traditional relational databases or persistent storage layers introduce unacceptable disk and network I/O latency, making them unviable for real-time limit order book (LOB) matching.

Furthermore, complex native capabilities such as [[concepts/iceberg-orders]] must be processed deterministically without incurring external round-trip overhead. To isolate lock contention and allow parallel processing of independent markets (e.g., `BTC/USD` vs. `ETH/USD`), the system implements [[concepts/symbol-sharding]].

## Decision
1. **Fully In-Memory State**: All active order books, bids, and asks reside entirely in memory as structured data primitives within the Go runtime, managed by [[entities/internal-matching-order-book]].
2. **Dedicated Slice Queues**: Dedicated slice queues are maintained per price level for bids and asks, implementing strict price-time priority semantics as specified in [[entities/internal-matching-engine]].
3. **Fine-Grained Sharding**: State access is sharded per trading symbol using `sync.RWMutex` constructs managed by the [[entities/internal-matching-engine]] to prevent cross-symbol blocking and maintain high throughput.
4. **Asynchronous Durability Offloading**: While memory structures handle hot-path matching, event streams and state changes are asynchronously published to Apache Kafka via [[entities/internal-events-kafka-publisher]] to ensure durability and feed downstream components like the Trade Settlement System and Market Data Gateway, as detailed in [[decisions/kafka-durability]] and [[concepts/data-pipeline]].