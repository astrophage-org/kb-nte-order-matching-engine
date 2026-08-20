<!-- anchor: internal/events/kafka_publisher.go:L1-L100 sha:HEAD -->

# Trade Execution Domain Model (`models/trade.go`)

The `TradeExecution` domain model defines the canonical event schema for matched trades produced by the Nexus Trading Exchange (NTE) [[summaries/order-matching-engine]]. It acts as an immutable structural contract shared across the matching core ([[entities/internal-matching-engine]]), event publication layer ([[entities/internal-events-kafka-publisher]]), and downstream enterprise systems such as post-trade clearing and compliance monitoring.

The struct represents the exact outcome of an order matching event where a taker order crosses resting liquidity in an [[entities/internal-matching-order-book]].

---

## Responsibilities

*   **Immutable Transaction Representation:** Encapsulates all necessary metadata for a completed trade execution (pricing, volume, execution timestamp, and market identifiers).
*   **Downstream Integration Contract:** Serializes directly into JSON payloads emitted to Kafka topics (`nte.trades.matched`) for consumption by external systems like the *Trade Settlement System* and *Compliance Surveillance Monitor*.
*   **Auditability & Traceability:** Retains relational links back to the original buy and sell order identifiers (`BuyOrderID`, `SellOrderID`) alongside specific participant accounts (`BuyerID`, `SellerID`).

---

## Dependencies

*   `time`: Standard Go library utilized for precise execution timestamping (`ExecutedAt`).
*   [[entities/models-order]]: Operates alongside `models/order.go` primitives, translating resting and aggressive orders into matched execution outputs.
*   [[entities/internal-events-kafka-publisher]]: Consumed by `KafkaPublisher.PublishTrades` to marshal execution records into Kafka messages.

---

## Schema Definition

```go
package models

import "time"

// TradeExecution represents a matched trade.
// Used heavily by trade-settlement-system and compliance-surveillance-monitor.
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

### Field Descriptions

*   `trade_id`: Unique universally unique identifier (UUID) generated for the specific execution instance.
*   `symbol`: The asset pair or instrument identifier (e.g., `BTC/USD`) matching the underlying [[entities/internal-matching-order-book]].
*   `price`: The execution price agreed upon during the match, typically driven by the resting limit order's price level.
*   `quantity`: The matched volume executed.