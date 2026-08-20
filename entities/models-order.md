<!-- anchor: cmd/server/main.go:L1-L100 sha:HEAD -->

# Order Domain Model (`models/order.go`)

The Order domain model represents the core primitive for inbound trading requests processed by the [[entities/internal-matching-engine]] and maintained within the [[entities/internal-matching-order-book]]. It defines the structural contract for transactions originating from upstream gateways ([[entities/cmd-server-main]]) before being matched against available liquidity or resting in the order book queues.

## Responsibilities

*   **Ingress Schema Definition**: Encapsulates all attributes required to ingest, validate, and route an order through the matching lifecycle (such as symbol, side, order type, price, and quantity).
*   **Iceberg Order State Management**: Maintains native fields (`IsIceberg`, `DisplayQuantity`, `HiddenQuantity`) to support [[concepts/iceberg-orders]], allowing the engine to track visible tranches versus hidden liquidity without external round-trips.
*   **JSON Serialization**: Provides standard JSON field tags for serializing and deserializing order payloads across Kafka event topics and internal APIs.

## Dependencies

*   `time`: Utilized by the `Timestamp` field to record the exact arrival or generation time of the order for price-time priority sorting.
*   [[entities/models-trade]]: Operates alongside the `TradeExecution` domain model, where inbound orders generate matched trade records during continuous trading or auction phases.

## Structure and Fields

```go
type Side string
type OrderType string

const (
	SideBuy  Side = "BUY"
	SideSell Side = "SELL"

	OrderTypeLimit  OrderType = "LIMIT"
	OrderTypeMarket OrderType = "MARKET"
)

type Order struct {
	ID              string    `json:"id"`
	Symbol          string    `json:"symbol"`
	TraderID        string    `json:"trader_id"`
	Side            Side      `json:"side"`
	Type            OrderType `json:"type"`
	Price           float64   `json:"price,omitempty"`
	Quantity        float64   `json:"quantity"`
	IsIceberg       bool      `json:"is_iceberg"`                 
	DisplayQuantity float64   `json: "display_quantity,omitempty"`
	HiddenQuantity  float64   `json:"hidden_quantity,omitempty"`  
	Timestamp       time.Time `json:"timestamp"`
}
```

*   **`ID`**: Unique string identifier (typically a UUID) assigned to the order upon ingestion.
*   **`Symbol`**: The trading instrument identifier (e.g., `BTC/USD`), used by the engine for [[concepts/symbol-sharding]] and lock contention mitigation.
*   **`Side`**: Specifies whether the order is a `BUY` (`bids`) or `SELL` (`asks`) instruction.
*   **`Type`**: Determines execution behavior (`LIMIT` or `MARKET`).
*   **`Price`**: Limit price for the order. Omitted for market orders.
*   **`Quantity`**: Total aggregate size of the order.
*   **`IsIceberg`**, **`DisplayQuantity`**, **`HiddenQuantity`**: Flags and sizing parameters governing hidden liquidity handling as described in [[concepts/iceberg-orders]].