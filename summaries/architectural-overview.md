<!-- anchor: cmd/server/main.go:L1-L100 sha:HEAD -->

# Nexus Trading Exchange Order Matching Engine Overview

This document provides a comprehensive architectural overview of the **Order Matching Engine (OME)**, part of the Nexus Trading Exchange (NTE) under the Global Financial Markets Group (GFMG).

For a high-level view of the application, refer to [[index]].

---

## 1. Main Components

The application is structured around a core microservice handling ultra-low-latency financial matching, asynchronous event broadcasting, and state management.

```
                  ┌────────────────────────────────────────┐
                  │           cmd/server/main.go           │
                  │  (Lifecycle Management & Simulation)   │
                  └───────────┬────────────────┬───────────┘
                              │                │
                              ▼                ▼
                 ┌──────────────────────┐ ┌──────────────────────┐
                 │ internal/matching    │ │ internal/events      │
                 │   - Engine           │ │   - KafkaPublisher   │
                 │   - OrderBook        │ └──────────┬───────────┘
                 └────────────┬─────────┘            │
                              │                      │ (Produce)
                              ▼                      ▼
                 ┌──────────────────────┐ ┌──────────────────────┐
                 │    models/*.go       │ │    Kafka Cluster     │
                 │  - Order             │ │  - nte.trades.matched│
                 │  - TradeExecution    │ │  - nte.orderbook...  │
                 └──────────────────────┘ └──────────────────────┘
```

*   **Server Lifecycle (`cmd/server/main.go`)**: 
    *   Initializes structured logging (`logrus` with JSON formatting).
    *   Connects to the event broker ([[entities/kafka-publisher|KafkaPublisher]]) with fallback handling if Kafka is unreachable.
    *   Instantiates the matching core and runs traffic simulation loops (handling features such as iceberg orders).
    *   Manages graceful shutdown via operating system signal trapping (`SIGINT`, `SIGTERM`).
*   **Matching Core (`internal/matching/`)**:
    *   `Engine`: Orchestrates multi-symbol order books concurrently, protecting state via fine-grained synchronization (`sync.RWMutex`). See [[entities/engine]] for details.
    *   `OrderBook`: Maintains price-time priority structures (`bids` and `asks`) per financial symbol, supporting specialized order handling (e.g., Iceberg orders with split visible/hidden quantities). Refer to [[entities/order-book]] and [[concepts/order-matching]] for detailed internal structure and matching mechanics.
*   **Event Publisher (`internal/events/kafka_publisher.go`)**:
    *   Wraps the `confluent-kafka-go` producer configured for low-latency delivery (`acks: all`, `linger.ms: 1`).
    *   Publishes matched trade reports and Level 2 (L2) order book snapshots. See [[entities/kafka-publisher]].
*   **Domain Models (`models/`)**:
    *   Defines core primitives: `Order`, `TradeExecution`, `Side` (`BUY`/`SELL`), and `OrderType` (`LIMIT`/`MARKET`). See [[entities/domain-models]].

---

## 2. Data Flows

1.  **Inbound Order Processing**:
    *   Orders are generated or ingested (simulated via `processTraffic` or consumed from `nte.orders.inbound`).
    *   The [[entities/engine|Engine]] intercepts the order, identifies or dynamically initializes the [[entities/order-book|OrderBook]] matching the target `Symbol` (e.g., `BTC/USD`), and dispatches it to `OrderBook.AddOrder`.
2.  **Matching & Trade Generation**:
    *   The [[entities/order-book|OrderBook]] validates order parameters (e.g., quantity > 0) and processes advanced execution features like **Iceberg orders** (evaluating display versus hidden quantities and managing automated slice-replenishment).
    *   Orders crossing the spread generate [[entities/domain-models|TradeExecution]] structures detailing matched pricing, volume, parties, and execution metadata using [[concepts/order-matching|Price-Time Priority]].
3.  **Outbound Event Dispatch**:
    *   Generated trades are asynchronously marshaled to JSON and published to the `nte.trades.matched` Kafka topic for downstream consumption by sister systems (refer to [[decisions/sister-system-integration|sister-system-integration]]).
    *   L2 book updates trigger snapshot publications to `nte.orderbook.snapshots` keyed by trading symbols for partition-safe distribution via [[concepts/event-driven-architecture|Kafka integration]].

---

## 3. Key Abstractions

*   **`Engine`**: Encapsulates the multi-symbol map of order books, abstracting the concurrency mechanics (`sync.RWMutex`) away from the underlying matching logic. Read more at [[entities/engine]].
*   **`OrderBook`**: Represents a single-instrument matching queue encapsulating price-time priority state and specialized trading rules (such as liquidity consumption and iceberg replenishment). See [[entities/order-book]].
*   **`KafkaPublisher`**: Provides an abstract messaging interface over Confluent Kafka, decoupling domain-specific trade execution and snapshot output from low-level broker configuration. Detailed in [[entities/kafka-publisher]].
*   **`Order` & `TradeExecution`**: Immutable-style structural models representing the core domain contracts exchanged between the gateway, engine, and downstream systems. See [[entities/domain-models]].

---

## 4. Design Decisions

*   **Memory-Driven / Concurrency Design**: The system relies on in-memory operations for performance, using localized `sync.RWMutex` locks per symbol to maximize parallelism across independent order books (e.g., trading pairs). Read more at [[decisions/memory-driven-concurrency]].
*   **Resilient Event Integration**: The Kafka publisher is initialized with a fallback mechanism, allowing the server to operate gracefully in standalone/development modes even if the Kafka broker cluster is unavailable. See [[decisions/resilient-event-integration]].
*   **Optimized Producer Settings**: Kafka producer parameters (`linger.ms: 1`, `acks: "all"`) are tuned to balance low-latency transaction throughput with durability guarantees.
*   **Sister-System Integration Architecture**: Decouples the OME from post-trade mechanics by pushing events onto specific Kafka topics (`nte.trades.matched`, `nte.orderbook.snapshots`), allowing independent scaling of the *Trade Settlement System*, *Market Data Gateway*, and *Compliance Surveillance Monitor*. Refer to [[decisions/sister-system-integration]] and [[concepts/event-driven-architecture]].