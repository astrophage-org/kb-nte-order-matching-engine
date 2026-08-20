# Asynchronous Event Broadcasting and Kafka Integration

The Nexus Trading Exchange (NTE) Order Matching Engine (OME) utilizes an event-driven architecture built on top of **Apache Kafka** to decouple core matching operations from downstream sister systems. This approach ensures that ultra-low-latency matching loops in the [[concepts/order-matching|matching core]] are not blocked by downstream persistence, market data broadcasting, or compliance checks.

Refer to [[summaries/architectural-overview]] for the high-level system topology and [[decisions/sister-system-integration]] for details on OME-to-sister-system decoupling.

---

## 1. Event Publisher Abstraction (`KafkaPublisher`)

The OME wraps interactions with the underlying streaming platform using the [[entities/kafka-publisher|KafkaPublisher]] component, built on top of the `confluent-kafka-go` library.

### Producer Tuning Parameters
To achieve ultra-low latency while maintaining durability guarantees, the producer is configured with specific tuning settings:
*   `acks: "all"`: Ensures that records are fully committed to all in-sync replicas before acknowledging a write, preventing data loss.
*   `linger.ms: 1`: Introduces a minimal 1-millisecond batching window to optimize throughput without introducing perceptible latency into the execution pipeline.

---

## 2. Kafka Topics & Data Contracts

The engine communicates across several dedicated Kafka topics within the `nte` namespace:

| Topic Name | Purpose | Key Format | Payload Model | Consuming Systems |
| :--- | :--- | :--- | :--- | :--- |
| `nte.orders.inbound` | Ingress stream for raw orders | Symbol / Order ID | `Order` | OME Ingress Layer |
| `nte.trades.matched` | Egress stream for executed trades | Partition Any | `TradeExecution` | `Trade Settlement System (TSS)`, Compliance Surveillance Monitor |
| `nte.orderbook.snapshots` | Egress stream for L2 book depths | Symbol (`[]byte(symbol)`) | JSON / Map | `Market Data Gateway (MDG)` |
| `nte.orders.rejected` | Egress stream for invalid orders | Order ID | Error Metadata | Compliance Surveillance Monitor |

---

## 3. Resilient Event Integration & Fallback Handling

To support high availability during local development, testing, or temporary broker outages, the OME implements a **resilient event integration** pattern (see [[decisions/resilient-event-integration|Resilient Event Integration]]):

*   When the server boots in `cmd/server/main.go`, it attempts to establish a connection to the Kafka cluster (`kafka-cluster.nte.gfmg.internal:9092`).
*   If the broker is unreachable, `NewKafkaPublisher` returns an error, which is caught by the server lifecycle manager.
*   The application logs a warning and operates in a **standalone fallback mode**, bypassing asynchronous event production while continuing local in-memory order matching and traffic simulation.

```go
publisher, err := events.NewKafkaPublisher(kafkaBroker, tradeTopic, snapshotTopic, log)
if err != nil {
    log.Warnf("Failed to connect to Kafka (running in fallback mode): %v", err)
}
```

---

## 4. Message Partitioning Strategy

*   **Trade Executions (`nte.trades.matched`)**: Produced using `kafka.PartitionAny`, allowing the Kafka cluster to distribute matched trades across partitions for maximum ingestion throughput by settlement engines.
*   **Order Book Snapshots (`nte.orderbook.snapshots`)**: Explicitly partitioned by symbol (`Key: []byte(symbol)`). This guarantees that state updates for a specific trading pair (e.g., `BTC/USD`) arrive at downstream consumers in strict chronological order per partition.