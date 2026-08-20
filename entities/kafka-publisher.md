<!-- anchor: internal/events/kafka_publisher.go:L1-L100 sha:HEAD -->

# KafkaPublisher and Producer Tuning

The `KafkaPublisher` component (`internal/events/kafka_publisher.go`) manages asynchronous event broadcasting from the Order Matching Engine (OME) to the core event broker cluster. It wraps the `confluent-kafka-go` producer library, translating internal execution and state changes into structured JSON payloads dispatched across specialized Kafka topics.

For a broader view of asynchronous messaging across the platform, see [[concepts/event-driven-architecture]] and [[decisions/sister-system-integration]].

---

## Responsibilities

*   **Producer Initialization & Configuration**: Connects to the Kafka broker cluster using low-latency tuning parameters.
*   **Trade Execution Broadcasting**: Serializes and publishes matched trade execution payloads (`models.TradeExecution`) to the `nte.trades.matched` topic for downstream clearing, settlement, and compliance monitoring.
*   **Order Book Snapshot Publishing**: Marshals and broadcasts Level 2 (L2) book snapshots to the `nte.orderbook.snapshots` topic, keyed by trading symbols for partition-safe consumption by market data gateways.
*   **Resilient Fallback Handling**: Integrates with the application startup lifecycle (`cmd/server/main.go`) to allow graceful degradation into standalone/development mode if broker connections cannot be established (see [[decisions/resilient-event-integration]]).

---

## Dependencies

*   `github.com/confluentinc/confluent-kafka-go/v2/kafka`: The underlying C-backed Confluent Kafka Go client used for high-throughput message production.
*   `github.com/sirupsen/logrus`: Provides structured JSON logging for producer errors, marshaling failures, and connection status.
*   `github.com/Astrophage/order-matching-engine/models`: Supplies domain types such as `models.TradeExecution` for serialization.
*   Sister systems consuming published events:
    *   [[entities/engine]] / [[entities/order-book]]: Generates the trades and state mutations that trigger publishing.
    *   `trade-settlement-system` and `compliance-surveillance-monitor`: Consume `nte.trades.matched`.
    *   `market-data-gateway`: Consumes `nte.orderbook.snapshots`.

---

## Producer Tuning Parameters

To satisfy ultra-low-latency financial matching requirements while maintaining necessary durability guarantees, the `KafkaPublisher` configures the producer with the following settings:

```go
p, err := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers": brokers,
    "acks":              "all",
    "linger.ms":         1, // Optimize for low latency, slight batching
})
```

### 1. `acks: "all"` (Durability & Consistency)
*   **Behavior**: Requires acknowledgment from all in-sync replicas (ISRs) before a produce request is considered successful.
*   **Rationale**: Financial exchanges cannot tolerate silent message loss on trade executions or market state. Setting `acks` to `all` guarantees that once a trade execution or book snapshot is acknowledged by the producer, it is durably replicated across the Kafka cluster.

### 2. `linger.ms: 1` (Latency vs. Throughput Balance)
*   **Behavior**: Forces the producer to wait up to 1 millisecond to batch multiple records together before transmitting them over the network.
*   **Rationale**: Without a linger period, every individual trade fill or book update immediately triggers a distinct network packet, inducing CPU and network context-switching overhead. A 1ms window permits slight batching of high-frequency micro-bursts without introducing perceptible latency into the egress pipeline.

---

## Event Topics & Partitioning Strategy

| Topic Name | Payload Type | Partition Key | Primary Consumers |
| :--- | :--- | :--- | :--- |
| `nte.trades.matched` | `models.TradeExecution` | `PartitionAny` (Round-robin / dynamic) | `trade-settlement-system`, `compliance-surveillance-monitor` |
| `nte.orderbook.snapshots` | `map[string]interface{}` (L2 Depth) | `symbol` (e.g., `BTC/USD`) | `market-data-gateway` |

*   **Symbol Partitioning**: Snapshot events explicitly assign the trading symbol as the Kafka message key (`Key: []byte(symbol)`). This guarantees that updates for a given instrument (such as `BTC/USD`) land on the same partition sequentially, preserving order consistency for downstream market data clients.