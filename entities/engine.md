<!-- anchor: internal/matching/engine.go:L1-L100 sha:HEAD -->

# Multi-Symbol Concurrency and Engine Abstraction

The `Engine` component (`internal/matching/engine.go`) serves as the central orchestration boundary for all matching operations within the [[index|Nexus Trading Exchange (NTE)]]. It acts as a concurrency-safe dispatcher that multiplexes inbound order streams across distinct [[entities/order-book|OrderBook]] instances partitioned by financial instruments (e.g., `BTC/USD`, `ETH/USD`).

For a broader view of the exchange's components, refer to [[summaries/architectural-overview]].

---

## Responsibilities

* **Multi-Symbol Multiplexing**: Maintains an in-memory mapping of trading symbols to their respective [[entities/order-book|OrderBook]] instances.
* **Dynamic Book Provisioning**: Automatically initializes a new `OrderBook` upon receiving the first inbound order for an unregistered symbol, logging the creation event via structured logging (`sirupsen/logrus`).
* **Concurrency Control**: Protects internal book map mutations using fine-grained synchronization primitives (`sync.RWMutex`), ensuring thread-safe access across concurrent traffic routines.
* **Execution Orchestration**: Routes validated orders to the targeted symbol's matching queue, records execution metrics (such as matching duration in microseconds), and returns resulting [[entities/domain-models|TradeExecution]] reports.

---

## Dependencies

* **`sync`**: Utilizes `sync.RWMutex` to manage safe concurrent access to the underlying `books` map.
* **`github.com/sirupsen/logrus`**: Employs structured JSON logging to record operational metrics, match durations, and book lifecycle events.
* **[[entities/order-book|OrderBook]]**: Interacts directly with individual symbol books (`internal/matching/order_book.go`) to execute price-time priority matching logic and manage order lifecycles.
* **[[entities/domain-models|Domain Models]]**: Consumes and processes [[entities/domain-models|Order]] inputs, returning slices of [[entities/domain-models|TradeExecution]] structures.

---

## Architecture & Concurrency Design

To achieve ultra-low latency, the engine implements a [[decisions/memory-driven-concurrency|memory-driven concurrency model]]. Rather than utilizing a single global lock across the entire exchange—which would create severe contention under high throughput—the `Engine` abstracts multi-symbol execution while delegating state management.

```
                  ┌────────────────────────────────────────┐
                  │            Engine.ProcessOrder         │
                  └───────────────────┬────────────────────┘
                                      │
                         [Acquire Lock (sync.RWMutex)]
                                      │
                                      ▼
                        Does `books[symbol]` exist?
                             /                \
                         (No)                  (Yes)
                         /                      \
        NewOrderBook(symbol)                     │
        Store in map                             │
             \                                   /
              └───► Release Map Lock ◄───────────┘
                               │
                               ▼
                   Call book.AddOrder(order)
                   (Independent Book Execution)
```

### Locking Strategy

1. **Map-Level Protection**: The `Engine` guards its internal `map[string]*OrderBook` with a `sync.RWMutex`. When `ProcessOrder` is invoked, it acquires the write lock only long enough to check for the symbol's existence and instantiate a new `OrderBook` if missing. 
2. **Book-Level Isolation**: Once the correct `OrderBook` reference is retrieved, map access is released. Subsequent matching logic and price-time priority queues operate independently per symbol. This design choice ensures that trading activity on `BTC/USD` does not block execution threads processing orders for `ETH/USD`.

---

## Code Implementation Reference

```go
package matching

import (
	"fmt"
	"sync"
	"time"

	"github.com/Astrophage/order-matching-engine/models"
	"github.com/sirupsen/logrus"
)

// Engine orchestrates the matching process across multiple symbols.
type Engine struct {
	books map[string]*OrderBook
	mu    sync.RWMutex
	log   *logrus.Logger
}

func NewEngine(log *logrus.Logger) *Engine {
	return &Engine{
		books: make(map[string]*OrderBook),
		log:   log,
	}
}

// ProcessOrder routes an order to the correct book and returns resulting trades.
func (e *Engine) ProcessOrder(order models.Order) ([]models.TradeExecution, error) {
	e.mu.Lock()
	book, exists := e.books[order.Symbol]
	if !exists {
		book = NewOrderBook(order.Symbol)
		e.books[order.Symbol] = book
		e.log.Infof("Created new order book for symbol: %s", order.Symbol)
	}
	e.mu.Unlock()

	start := time.Now()
	trades, err := book.AddOrder(order)
	duration := time.Since(start)

	e.log.WithFields(logrus.Fields{
		"symbol":    order.Symbol,
		"order_id":  order.ID,
		"match_uS":  duration.Microseconds(),
		"trades_qty": len(trades),
	}).Debug("Order processed")

	if err != nil {
		return nil, fmt.Errorf("failed to process order %s: %w", order.ID, err)
	}

	return trades, nil
}
```

---

## Related Documentation

* [[summaries/architectural-overview]] - High-level system architecture and data flow diagrams.
* [[concepts/order-matching]] - Core price-time priority rules and matching mechanics.
* [[entities/order-book]] - Detailed breakdown of LOB data structures and Iceberg order handling.
* [[decisions/memory-driven-concurrency]] - Rationale behind in-memory sharding and fine-grained locking.