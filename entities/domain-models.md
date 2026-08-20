<!-- anchor: internal/matching/order_book.go:L1-L100 sha:HEAD -->

# Domain Models

The Nexus Trading Exchange (NTE) Order Matching Engine uses clean, immutable-style structural models located in the `models/` package to represent core financial contracts exchanged between inbound gateways, the [[entities/engine]], the [[entities/order-book]], and downstream sister systems (such as the Trade Settlement System and Market Data Gateway).

These models are designed for high-performance memory serialization, utilizing JSON tags for event publishing via [[entities/kafka-publisher]].

---

## Core Types and Enums

### `Side`
Defines the market direction of an order or trade.
*   `SideBuy` (`"BUY"`): Represents an order bidding to purchase an asset.
*   `SideSell` (`"SELL"`): Represents an order offering to dispose of an asset.

### `OrderType`
Defines the execution instruction profile for an inbound order.
*   `OrderTypeLimit` (`"LIMIT"`): Requires execution at a specific limit price or better.
*   `OrderTypeMarket` (`"MARKET"`): Executes immediately against available liquidity at the best available counter-price.

---

## Entity Specifications

### `Order`
The `Order` struct represents an inbound execution request ingested from the gateway or simulation loops. It supports advanced features like **Iceberg orders** by tracking total, display, and hidden quantities.

```go
type Order struct {
    ID              string    `json:"id"`
    Symbol          string    `json:"symbol"`
    TraderID        string    `json:"trader_id"`
    Side            Side      `json:"side"`
    Type            OrderType `json:"type"`
    Price           float64   `json:"price,omitempty"`
    Quantity        float64   `json:"quantity"`
    IsIceberg       bool      `json:"is_iceberg"`                 // Indicates if this is an Iceberg order
    DisplayQuantity float64   `json:"display_quantity,omitempty"` // Visible portion shown to market
    HiddenQuantity  float64   `json:"hidden_quantity,omitempty"`  // Remaining hidden portion
    Timestamp       time.Time `json:"timestamp"`
}
```

### `TradeExecution`
The `TradeExecution` struct represents a successfully matched transaction generated when orders cross the spread within an [[entities/order-book]]. These reports are serialized and published to Kafka topics to decouple OME core execution from downstream processing (refer to [[decisions/sister-system-integration]]).

```go
type TradeExecution struct {
    TradeID      string    `json:"trade_id"`
    Symbol       string    `json:"symbol"`
    Price        float64   `json:"price"`
    Quantity     float64   `json:"quantity"`
    BuyerID      string    `json:"buyer_id"`
    SellerID     string    `json:"seller_id"`
    BuyOrderID   string    `json:"buy_order_id"`
    SellOrderID  string    `json:"sell_order_id"`
    ExecutedAt   time.Time `json:"executed_at"`
    ExchangeID   string    `json:"exchange_id"`
    MatchPhase   string    `json:"match_phase"` // e.g., "CONTINUOUS", "AUCTION"
}
```

---

## Responsibilities

*   **Contract Definition**: Establish strongly-typed Go representations for orders, trade reports, trading sides, and execution types.
*   **Serialization Support**: Provide standard JSON struct tags to facilitate rapid marshaling for event streaming through [[entities/kafka-publisher]].
*   **Iceberg State Tracking**: Encapsulate display versus hidden quantity metrics directly within the `Order` structure to support advanced slice replenishment mechanics in the [[entities/order-book]].

---

## Dependencies

*   **Standard Library (`time`)**: Utilized for precise timestamping of order ingress (`Order.Timestamp`) and trade execution events (`TradeExecution.ExecutedAt`).
*   **Downstream Consumers**: Directly structures payloads produced to Kafka topics (`nte.trades.matched`, `nte.orderbook.snapshots`), serving as the definitive contract for systems described in [[summaries/architectural-overview]].