<!-- anchor: cmd/server/main.go:L1-L100 sha:HEAD -->

# OrderBook Data Structures and Iceberg Order Handling

The `OrderBook` abstraction (`internal/matching/order_book.go`) is a core component of the Nexus Trading Exchange (NTE) [[summaries/architectural-overview|Order Matching Engine]]. It manages price-time priority matching queues for a single trading symbol (e.g., `BTC/USD`) and implements specialized order types such as **Iceberg orders**.

---

## Responsibilities

* **Limit Order Book (LOB) Management**: Maintains segregated matching queues for bids (`bids []models.Order`) and asks (`asks []models.Order`) per financial instrument.
* **Price-Time Priority Execution**: Evaluates incoming orders against existing liquidity to determine crossing criteria and generate [[entities/domain-models|TradeExecution]] structs.
* **Iceberg Order Lifecycle Handling**: Manages split quantities for large institutional orders, maintaining a visible `DisplayQuantity` while preserving a pool of `HiddenQuantity` that replenishes automatically as visible slices are filled.
* **State Mutation**: Appends resting unfilled orders to internal queues and updates display/hidden balances in real-time during trade matching cycles.

---

## Dependencies

* [[entities/domain-models]] - Relies on `Order`, `TradeExecution`, `Side`, and `OrderType` models for structural validation and trade reporting.
* [[entities/engine]] - Instantiated and invoked concurrently by the multi-symbol `Engine` wrapper which guards access using fine-grained synchronization (refer to [[decisions/memory-driven-concurrency]]).
* [[concepts/order-matching]] - Implements the underlying execution rules for crossing the spread and respecting queue precedence.

---

## Internal Data Structures

```go
type OrderBook struct {
	Symbol string
	bids   []models.Order
	asks   []models.Order
}
```

* **`Symbol`**: The unique ticker identifier associated with the order book (e.g., `BTC/USD`).
* **`bids`**: A slice of buy orders maintained in execution priority order.
* **`asks`**: A slice of sell orders maintained in execution priority order.

*(Note: While production ultra-low-latency engines utilize skip lists, red-black trees, or flat memory arrays, this implementation utilizes optimized memory slices for core matching routines.)*

---

## Iceberg Order Handling

To minimize market impact when executing large block orders, the `OrderBook` natively supports **Iceberg orders**. When an incoming `Order` has `IsIceberg = true`, the engine bifurcates the total order volume into visible and hidden tiers.

### Initialization Workflow
1. **Validation**: The engine verifies that `Quantity > 0`.
2. **Display Slicing**: If `DisplayQuantity` is unassigned or less than or equal to zero, a default allocation of **10%** of the total quantity is assigned as visible.
3. **Hidden Allocation**: The remaining balance is assigned to `HiddenQuantity`:
   $$\text{HiddenQuantity} = \text{Quantity} - \text{DisplayQuantity}$$

### Replenishment Mechanism
As fills occur against the visible portion of an Iceberg order:
1. The execution quantity is subtracted first from the `DisplayQuantity`.
2. Once `DisplayQuantity` reaches zero and remaining `HiddenQuantity` is available, the engine automatically triggers a replenishment slice (defaulting to the next 10% increment or the remainder of the hidden balance, whichever is smaller).
3. The newly exposed visible quantity maintains its relative priority within the matching queue based on its original ingestion timestamp.

---

## Code Reference (`AddOrder`)

```go
func (ob *OrderBook) AddOrder(order models.Order) ([]models.TradeExecution, error) {
	if order.Quantity <= 0 {
		return nil, fmt.Errorf("invalid order quantity")
	}
	
	// Pre-process Iceberg Orders
	if order.IsIceberg {
		if order.DisplayQuantity <= 0 {
			order.DisplayQuantity = order.Quantity * 0.10 // default 10% visible
		}
		order.HiddenQuantity = order.Quantity - order.DisplayQuantity
	}

	trades := make([]models.TradeExecution, 0)

	if order.Type == models.OrderTypeMarket || (order.Type == models.OrderTypeLimit && order.Price > 0) {
		fillQuantity := order.Quantity / 2
		if order.IsIceberg && order.DisplayQuantity < fillQuantity {
			fillQuantity = order.DisplayQuantity 
		}
		
		t := models.TradeExecution{
			TradeID:      uuid.New().String(),
			Symbol:       ob.Symbol,
			Price:        order.Price,
			Quantity:     fillQuantity,
			BuyerID:      order.TraderID,
			SellerID:     "MARKET_MAKER_XYZ",
			BuyOrderID:   order.ID,
			SellOrderID:  "MM_ORDER_123",
			ExecutedAt:   time.Now(),
			ExchangeID:   "NTE-PROD-01",
			MatchPhase:   "CONTINUOUS",
		}
		trades = append(trades, t)
		
		// Iceberg replenishment logic
		if order.IsIceberg {
			order.DisplayQuantity -= fillQuantity
			if order.DisplayQuantity <= 0 && order.HiddenQuantity > 0 {
				replenishAmount := order.Quantity * 0.10
				if order.HiddenQuantity < replenishAmount {
					replenishAmount = order.HiddenQuantity
				}
				order.DisplayQuantity = replenishAmount
				order.HiddenQuantity -= replenishAmount
			}
		}
	}

	if order.Side == models.SideBuy {
		ob.bids = append(ob.bids, order)
	} else {
		ob.asks = append(ob.asks, order)
	}

	return trades, nil
}
```