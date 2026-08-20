# Decision Record: Memory-Driven Concurrency & Fine-Grained Locking

## Status
**Accepted**

## Context
The [[index|Nexus Trading Exchange (NTE)]] Order Matching Engine (OME) requires ultra-low latency order execution and state management to meet the performance SLAs of the [[summaries/architectural-overview|Global Financial Markets Group (GFMG)]]. Traditional database-backed state storage introduces unacceptable I/O latency overhead. Therefore, the matching engine relies entirely on in-memory operations.

However, operating multiple financial symbols (such as `BTC/USD`, `ETH/USD`, etc.) concurrently introduces synchronization challenges. If a single global lock protects the entire collection of order books, high-throughput traffic on one trading pair (e.g., during market volatility) would block execution and matching for completely independent trading pairs.

## Decision
We implement a **memory-driven concurrency model with fine-grained sharding per symbol** inside the [[entities/engine|Engine]] and [[entities/order-book|OrderBook]] abstractions:

1. **Engine-Level Sharding (`sync.RWMutex`)**: The `Engine` maintains a map of symbol-specific order books (`map[string]*OrderBook`) protected by a localized reader-writer mutex (`sync.RWMutex`).
2. **Concurrent Independent Routing**: When `Engine.ProcessOrder` receives an incoming [[entities/domain-models|Order]], it acquires a write lock briefly to check or dynamically instantiate the target symbol's `OrderBook`. Subsequent matching logic executes isolated within that specific instrument's book context.
3. **Optimized Lock Contention**: By partitioning locking logic per symbol, trading activity on `BTC/USD` does not contend with or block threads processing `ETH/USD`, maximizing CPU utilization and parallelism across independent markets.

### Implementation Reference
```go
// internal/matching/engine.go
type Engine struct {
	books map[string]*OrderBook
	mu    sync.RWMutex
	log   *logrus.Logger
}

func (e *Engine) ProcessOrder(order models.Order) ([]models.TradeExecution, error) {
	e.mu.Lock()
	book, exists := e.books[order.Symbol]
	if !exists {
		book = NewOrderBook(order.Symbol)
		e.books[order.Symbol] = book
		e.log.Infof("Created new order book for symbol: %s", order.Symbol)
	}
	e.mu.Unlock()

	// Execution happens isolated per order book instance
	return book.AddOrder(order)
}
```

## Consequences
* **Positive**: Achieves microsecond-level matching latencies while allowing distinct trading symbols to scale horizontally across multi-core processors without cross-instrument locking penalties.
* **Positive**: Cleanly abstracts concurrency primitives away from core price-time priority [[concepts/order-matching|matching logic]].
* **Negative / Trade-off**: In-memory state requires robust recovery and persistence mechanisms (e.g., replaying Kafka transaction logs from [[decisions/resilient-event-integration|inbound event streams]] or restoring end-of-day snapshots from object storage) in the event of a host failure or unexpected restart.