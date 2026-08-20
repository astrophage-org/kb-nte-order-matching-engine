# Architectural Overview: Order Matching Engine (OME)

The **Order Matching Engine (OME)** is a core component of the Global Financial Markets Group (GFMG) Nexus Trading Exchange ecosystem, engineered for high-throughput, microsecond-level performance. 

For related architectural concepts and component specifications, refer to [[entities/cmd-server-main]], [[entities/internal-matching-engine]], [[entities/internal-matching-order-book]], [[entities/internal-events-kafka-publisher]], [[entities/models-order]], and [[entities/models-trade]].

---

## Navigation Table

| Category | Document | Description |
| :--- | :--- | :--- |
| **Summaries** | [[summaries/order-matching-engine]] | High-level OME architectural overview |
| **Entities** | [[entities/cmd-server-main]] | Server core and process orchestrator |
| **Entities** | [[entities/internal-matching-engine]] | Matching engine core implementation |
| **Entities** | [[entities/internal-matching-order-book]] | Order book and price-time priority logic |
| **Entities** | [[entities/internal-events-kafka-publisher]] | Kafka event publisher for egress |
| **Entities** | [[entities/models-order]] | Order domain model structure |
| **Entities** | [[entities/models-trade]] | Trade execution domain model schema |
| **Concepts** | [[concepts/data-pipeline]] | End-to-end data flow and ingress/egress routing |
| **Concepts** | [[concepts/iceberg-orders]] | Iceberg order handling and hidden liquidity |
| **Concepts** | [[concepts/symbol-sharding]] | Symbol sharding and lock contention mitigation |
| **Decisions** | [[decisions/in-memory-queues]] | In-memory priority queues for microsecond latency |
| **Decisions** | [[decisions/kafka-durability]] | Resiliency and Kafka-based event log durability |

---

## 1. Main Components

The system is structured as an in-memory, event-driven matching engine:

*   **Server Core & Process Orchestrator (`[[entities/cmd-server-main]]`)**: Initializes logging (`logrus`), establishes Kafka connectivity via [[entities/internal-events-kafka-publisher]], instantiates the matching engine, and manages the lifecycle/graceful shutdown of the exchange node via OS signals. In the current iteration, it also includes a traffic simulation loop for inbound orders (e.g., handling limit and [[concepts/iceberg-orders|iceberg orders]]).
*   **Matching Core (`[[entities/internal-matching-engine]]` & `[[entities/internal-matching-order-book]]`)**: 
    *   `Engine`: Orchestrates multi-symbol order matching, maintaining thread-safe, sharded access to symbol-specific order books using a `sync.RWMutex` (see [[concepts/symbol-sharding]]).
    *   `OrderBook`: Manages [[decisions/in-memory-queues|in-memory priority queues]] for individual symbols (`bids` and `asks`). It evaluates incoming orders against resting liquidity, handles matching logic, and computes partial fills.
    *   **Iceberg Order Handling**: Automatically fragments large orders, maintaining visible (`DisplayQuantity`) and hidden (`HiddenQuantity`) states, and dynamically replenishing the display tranche as visible segments are filled (see [[concepts/iceberg-orders]]).
*   **Event Publisher (`[[entities/internal-events-kafka-publisher]]`)**: Interfaces with Apache Kafka using `confluent-kafka-go`. Optimized with `acks=all` and minimal linger times (`linger.ms=1`) to balance durability with ultra-low latency (see [[decisions/kafka-durability]]). Publishes matched trades and order book snapshots.
*   **Domain Models (`[[entities/models-order]]` & `[[entities/models-trade]]`)**: Define core primitives (`Order`, `TradeExecution`, `Side`, `OrderType`) utilized across ingress, matching, and egress layers.

---

## 2. Data Flows

For the end-to-end data pipeline trace, see [[concepts/data-pipeline]].

1.  **Inbound Ingress**: Orders originate from upstream gateways (simulated via traffic generator or Kafka consumers) containing parameters such as symbol, trader ID, side, type, price, quantity, and iceberg flags.
2.  **State Routing & Matching**:
    *   The `Engine` receives the `models.Order`, locking on the specific symbol to retrieve or initialize its `OrderBook`.
    *   The `OrderBook` evaluates the order against existing book state, executing matching algorithms against available liquidity (generating `models.TradeExecution` records).
    *   Unfilled portions or resting orders are appended to the appropriate side (`bids` or `asks`).
3.  **Egress & Downstream Distribution**:
    *   **Matched Trades**: Emitted to the `nte.trades.matched` Kafka topic for consumption by the Trade Settlement System.