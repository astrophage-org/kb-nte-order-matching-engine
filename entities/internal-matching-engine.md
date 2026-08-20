<!-- anchor: cmd/server/main.go:L1-L100 sha:HEAD -->

# Matching Engine Core (`internal/matching/engine.go`)

The Matching Engine Core (`internal/matching/engine.go`) serves as the central orchestrator for multi-symbol order matching within the Nexus Trading Exchange (NTE) ecosystem. It coordinates thread-safe access to symbol-specific order books and records execution latencies for telemetry and logging.

For an overview of the broader architecture, see [[index]] and [[summaries/order-matching-engine]]. For details regarding order book management and price-time priority execution, refer to [[entities/internal-matching-order-book]].

---

## Responsibilities

*   **Multi-Symbol Routing**: Maintains an in-memory map of trading symbols to their respective [[entities/internal-matching-order-book|OrderBooks]], dynamically instantiating new books upon receiving orders for unmapped symbols.
*   **Concurrency Control**: Utilizes a `sync.RWMutex` to manage thread-safe registration and access to symbol books, ensuring isolated lock contention per symbol in alignment with [[concepts/symbol-sharding]].
*   **Execution Telemetry**: Measures order processing latency in microseconds and logs structured diagnostic details (symbol, order ID, execution time, and trade quantities) using `sirupsen/logrus`.
*   **Error Handling & Propagation**: Captures errors returned during book mutation and order matching, wrapping them with context for upstream components like [[entities/cmd-server-main]].

---

## Dependencies

*   **[[entities/internal-matching-order-book]] (`internal/matching/order_book.go`)**: Delegates actual liquidity evaluation, price-time priority matching, and iceberg handling to the symbol-specific `OrderBook`.
*   **[[entities/models-order]] (`models/order.go`)**: Consumes `models.Order` payloads representing inbound trading requests.
*   **[[entities/models-trade]] (`models/trade.go`)**: Returns collections of `models.TradeExecution` records generated as a result of successful order matching.
*   **External Packages**: Uses `github.com/sirupsen/logrus` for structured logging.

---

## Core Structure & Implementation

The `Engine` type encapsulates the book registry and concurrency primitives:

```go
// Engine orchestrates the matching process across multiple symbols.
type Engine struct {
	books map[string]*OrderBook
	mu    sync.RWMutex
	log   *logrus.Logger
}
```

### Order Processing Workflow

When an inbound order is received via [[entities/cmd-server-main]] or an ingress pipeline (see [[concepts/data-pipeline]]), the engine performs the following steps:

1. Acquires a read-write lock (`e.mu.Lock()`) to check if an `OrderBook` exists for `order.Symbol`. If absent, it initializes a new book via `NewOrderBook(order.Symbol)`.
2. Measures execution duration using `time.Now()`.
3. Invokes `book.AddOrder(order)` to evaluate resting liquidity and produce trade executions.
4. Logs execution metrics and returns the resulting trades or any matching errors.