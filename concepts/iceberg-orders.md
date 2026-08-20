# Iceberg Order Handling and Hidden Liquidity Management

## Overview
Within the Nexus Trading Exchange (NTE) [[summaries/order-matching-engine]], **Iceberg Orders** are a native complex order type designed to allow institutional participants to execute large-size orders without revealing their entire volume to the broader market. This mitigates adverse price impact and signaling risks. 

Iceberg order functionality is implemented directly inside [[entities/internal-matching-order-book]] (`internal/matching/order_book.go`), ensuring that hidden liquidity rules, fragmentation mechanics, and tranche replenishment execute deterministically within the critical matching loop alongside standard limit and market orders defined in [[entities/models-order]].

---

## State Model and Attributes

An iceberg order is characterized by a segregation of its total volume into two distinct quantities:
1. **Display Quantity (`DisplayQuantity`)**: The visible tranche exposed to the limit order book and broadcasted via L2/L3 market data feeds to systems like the *Market Data Gateway* (as outlined in [[concepts/data-pipeline]]).
2. **Hidden Quantity (`HiddenQuantity`)**: The unexposed reserve volume held internally within memory.

As detailed in [[entities/models-order]], the base `Order` domain model supports these flags natively:
* `IsIceberg` (`bool`): Flag indicating whether the order requires iceberg fragmentation.
* `DisplayQuantity` (`float64`): The current visible slice size.
* `HiddenQuantity` (`float64`): The remaining pool awaiting disclosure.

---

## Processing and Lifecycle Mechanics

When an inbound order is received by [[entities/internal-matching-engine]] and routed to its respective symbol's [[entities/internal-matching-order-book]], the following execution lifecycle takes place:

### 1. Initialization and Tranche Allocation
If `IsIceberg` is set to `true`, the matching core validates the initial visibility parameters. If no explicit `DisplayQuantity` is provided or if it is misconfigured, the engine applies a default sizing rule (typically 10% of the total order quantity):
```go
if order.IsIceberg {
    if order.DisplayQuantity <= 0 {
        order.DisplayQuantity = order.Quantity * 0.10 // default 10% visible
    }
    order.HiddenQuantity = order.Quantity - order.DisplayQuantity
}
```

### 2. Liquidity Matching and Tranche Depletion
During the matching cycle, incoming or resting iceberg orders interact with opposing liquidity up to the bounds of their currently visible tranche. When a fill occurs that consumes a portion of the `DisplayQuantity`:
* The fill quantity is subtracted directly from the `DisplayQuantity`.
* Executed trades are generated via [[entities/models-trade]] (`TradeExecution`) and emitted asynchronously through [[entities/internal-events-kafka-publisher]] to the `nte.trades.matched` topic for downstream consumption by the *Trade Settlement System* and *Compliance Surveillance Monitor*.

### 3. Automatic Hidden Tranche Replenishment
Once the active display tranche is exhausted, the engine replenishes it from the `HiddenQuantity` pool.