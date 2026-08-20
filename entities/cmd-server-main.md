<!-- anchor: cmd/server/main.go:L1-L100 sha:HEAD -->

# Server Core and Process Orchestrator (`cmd/server/main.go`)

The Server Core and Process Orchestrator is the main entry point and lifecycle manager for the Nexus Trading Exchange (NTE) Order Matching Engine. Implemented in `cmd/server/main.go`, it initializes foundational infrastructure—including structured logging (`logrus`) and Kafka event publishing—instantiates the core matching engine, and manages graceful shutdowns during system termination signals.

For an overarching view of how this component fits into the wider architecture, refer to [[index]], [[summaries/order-matching-engine]], and [[entities/cmd-server-main]].

---

## Responsibilities

*   **Initialization & Configuration**: Establishes global parameters, configures structured JSON logging via `logrus`, and prepares connection strings for downstream infrastructure.
*   **Infrastructure Wiring**: Initializes the [[entities/internal-events-kafka-publisher|Kafka Event Publisher]] (`internal/events/kafka_publisher.go`) and the multi-symbol [[entities/internal-matching-engine|Matching Core Engine]] (`internal/matching/engine.go`).
*   **Traffic Simulation Loop**: Runs a concurrent background worker (`processTraffic`) that feeds synthetic limit and [[concepts/iceberg-orders|iceberg orders]] into the matching engine for testing and validation workflows.
*   **Lifecycle Management & Graceful Shutdown**: Listens for operating system interrupts (`SIGINT`, `SIGTERM`) via a buffered signal channel to ensure active workers cleanly terminate before process exit.

---

## Dependencies

*   `github.com/sirupsen/logrus`: Provides structured JSON logging across the orchestration and traffic generation lifecycle.
*   `github.com/google/uuid`: Generates unique identifiers for simulated traffic orders.
*   [[entities/internal-matching-engine]] (`internal/matching/engine.go`): The central processing unit for routing and matching orders against symbol-specific books.
*   [[entities/internal-events-kafka-publisher]] (`internal/events/kafka_publisher.go`): Interfaces with the Apache Kafka cluster to emit matched trades and order book snapshots.
*   [[entities/models-order]] & [[entities/models-trade]] (`models/order.go`, `models/trade.go`): Domain models utilized when generating and processing traffic simulation batches.

---

## Process Flow

1. **Bootstrap**: The application launches, initializes the `logrus` logger with JSON formatting, and outputs startup signatures for the Global Financial Markets Group (GFMG) Nexus Trading Exchange.
2. **Producer & Engine Setup**: Connects to the Kafka broker (`kafka-cluster.nte.gfmg.internal:9092`) for the `nte.trades.matched` and `nte.orderbook.snapshots` topics. If connection fails, it falls back gracefully with a warning log while instantiating the [[entities/internal-matching-engine|Matching Engine]].
3. **Traffic Loop**: Spawns a background goroutine executing on a 500ms ticker. This loop builds standard limit orders or [[concepts/iceberg-orders|iceberg orders]] and routes them through [[concepts/data-pipeline]].