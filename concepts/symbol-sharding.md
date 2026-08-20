# Symbol Sharding and Lock Contention Mitigation

To achieve microsecond-level execution guarantees within the [[summaries/order-matching-engine]], the architecture implements **symbol sharding** to isolate state and eliminate global lock contention across distinct trading pairs (e.g., `BTC/USD` vs. `ETH/USD`).

Related architectural components and code implementations are detailed in [[entities/internal-matching-engine]], [[entities/internal-matching-order-book]], and [[concepts/data-pipeline]].

---

##  1. The Concurrency Challenge in Matching Engines

In a high-throughput financial exchange like the Nexus Trading Exchange (NTE), thousands of orders arrive concurrently from upstream gateways ([[entities/cmd-server-main]]). If all incoming orders for all symbols contended for a single global mutex lock over the entire exchange state, thread contention would introduce severe latency spikes and throttle throughput.

To prevent this bottleneck, the matching core isolates state per instrument.

---

## 2. Symbol Sharding Architecture

As implemented in [[entities/internal-matching-engine]], the `Engine` struct maintains a collection of isolated order books keyed directly by the trading symbol:

```go
type Engine struct {
	books map[string]*OrderBook
	mu    sync.RWMutex
	log   *logrus.Logger
}
```

### Routing and Initialization Flow

When an inbound [[entities/models-order]] is processed by `Engine.ProcessOrder`:
1. The engine acquires a read/write lock (`e.mu.Lock()`) solely to look up or lazily initialize the `OrderBook` instance for `order.Symbol`.
2. Once the specific `OrderBook` reference is retrieved, the global engine map lock is released immediately.
3. Subsequent matching algorithms, price-time priority evaluations, and state mutations execute entirely within the boundary of the isolated [[entities/internal-matching-order-book]].

```go
func (e *Engine) ProcessOrder(order models.Order) ([]models.TradeExecution, error) {
	e.mu.Lock()
	book, exists := e.books[order.Symbol]
	if !exists {
		book = NewOrderBook(order.Symbol)
		e.books[order.Symbol] = book
		e.log.Infof("Created new order book for symbol: %s", order.Symbol)
	}
	e.mu.Unlock()

	// Processing continues on the isolated OrderBook instance
	return book.AddOrder(order)
}
```

---

## 3. Benefits of Per-Symbol Isolation

* **Parallelism Across Markets:** Orders for `BTC/USD` can be matched concurrently with orders for `ETH/USD` or other liquidity pairs without blocking each other.
* **Granular Lock Contention:** Contention is restricted strictly to traders executing transactions within the exact same symbol book.
* **Predictable Latency Profiles:** Microsecond match durations (`match_uS`) remain stable under high load because processing threads are not queued behind unrelated market activities.
* **Streamlined Event Partitioning:** As seen in [[entities/internal-events-kafka-publisher]], downstream market data snapshots are partitioned explicitly by symbol keys (`Key: []byte(symbol)`).