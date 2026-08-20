<!-- anchor: internal/matching/order_book.go:L1-L100 sha:HEAD -->

# Order Book and Price-Time Priority Logic (`internal/matching/order_book.go`)

The `OrderBook` component is responsible for maintaining the limit order book (LOB) state for an individual trading symbol within the [[entities/internal-matching-engine]]. It manages price-time priority queues for both [[entities/models-order|bids and asks]], evaluates inbound liquidity, executes matches, and handles complex execution primitives such as [[concepts/iceberg-orders|Iceberg Orders]].

---

## Responsibilities

*   **Order Book State Management**: Maintains separate internal collections (`bids` and `asks`) of [[entities/models-order]] structs for a specific trading symbol, structured around price-time priority.
*   **Liquidity Crossing & Matching**: Evaluates incoming limit and market orders against resting liquidity, computing fills and generating immutable [[entities/models-trade|TradeExecution]] records.
*   **Iceberg Order Processing**: Intrinsic handling of hidden liquidity parameters (`DisplayQuantity` and `HiddenQuantity`), ensuring the visible display tranche is dynamically replenished as fills occur without external round-trip overhead (see [[concepts/iceberg-orders]]).
*   **Order Book Mutability**: Appends unfilled order quantities or resting passive orders to the appropriate side of the book.

---

## Dependencies

*   `github.com/Astrophage/order-matching-engine/models`: Imports core domain primitives including [[entities/models-order]] and [[entities/models-trade]].
*   `github.com/google/uuid`: Generates unique identifiers for executed trades (`TradeID`).
*   [[entities/internal-matching-engine]]: Instantiated and managed dynamically by the parent `Engine` via symbol sharding (see [[concepts/symbol-sharding]]).

---

## Core Data Structures

```go
// OrderBook represents a price-time priority matching queue for a single symbol.
type OrderBook struct {
	Symbol string
	bids   []models.Order
	asks   []models.Order
}
```

While production-grade ultra-low latency books utilize advanced structures like skip lists, red-black trees, or flat arrays, this implementation utilizes optimized execution slices to track resting liquidity per symbol channel.

---

## Execution Flow & Algorithm

1. **Validation**: Validates that incoming orders possess a positive quantity.
2. **Iceberg Initialization**: If `IsIceberg` is set:
   * Defaults `DisplayQuantity` to 10% of the total order quantity if unassigned.
   * Calculates the remaining `HiddenQuantity` (`Quantity - DisplayQuantity`).
3. **Matching & Fill Computation**: Crosses the order against current book liquidity, enforcing visible-first matching rules for iceberg structures.
4. **Replenishment**: For iceberg orders, once the `DisplayQuantity` is exhausted, subsequent slices are automatically replenished from the `HiddenQuantity`.
5. **State Persist**: Appends residual or passive orders to the appropriate `bids` or `asks` slice queue.

---

## Related Components & Concepts

*   [[concepts/data-pipeline]]
*   [[decisions/in-memory-queues]]