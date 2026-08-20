<!-- anchor: internal/events/kafka_publisher.go:L1-L100 sha:HEAD -->

# Kafka Event Publisher (`internal/events/kafka_publisher.go`)

The `KafkaPublisher` acts as the primary asynchronous egress boundary for the [[summaries/order-matching-engine|Order Matching Engine (OME)]]. It interfaces directly with Apache Kafka using the `confluent-kafka-go` client library to broadcast matched trades and order book snapshots to downstream consumer applications across the Global Financial Markets Group (GFMG) ecosystem.

For broader architectural context on data egress patterns, refer to [[concepts/data-pipeline]]. For details on durability guarantees, see [[decisions/kafka-durability]].

---

## Responsibilities

*   **Producer Lifecycle Management**: Initializes and manages the underlying `confluent-kafka-go` producer instance with high-throughput and low-latency configuration tunings.
*   **Trade Execution Egress**: Serializes [[entities/models-trade|TradeExecution]] domain objects into JSON payloads and asynchronously dispatches them to the configured matched trades topic (`nte.trades.matched`).
*   **Market Data Snapshot Egress**: Publishes L2/L3 order book snapshots and depth metrics to the market data topic (`nte.orderbook.snapshots`), partitioned explicitly by trading symbol to preserve ordering guarantees for individual markets.
*   **Asynchronous Non-Blocking Delivery**: Decouples network I/O from the ultra-low latency matching core ([[entities/internal-matching-engine]]) by utilizing asynchronous message production (`kafka.PartitionAny`).

---

## Dependencies

*   **`github.com/confluentinc/confluent-kafka-go/v2/kafka`**: Official Confluent Kafka Go client used for interacting with the Kafka cluster.
*   **`github.com/sirupsen/logrus`**: Structured logging framework used for error tracking and publishing telemetry.
*   **[[entities/models-trade]] (`models/trade.go`)**: Provides the `models.TradeExecution` domain model structure serialized during trade event publishing.

---

## Implementation Details

### Initialization

The publisher is instantiated via `NewKafkaPublisher`, which configures the underlying producer with durability and performance optimizations:

```go
func NewKafkaPublisher(brokers, tradeTopic, snapshotTopic string, log *logrus.Logger) (*KafkaPublisher, error) {
	p, err := kafka.NewProducer(&kafka.ConfigMap{
		"bootstrap.servers": brokers,
		"acks":              "all",
		"linger.ms":         1, // Optimize for low latency, slight batching
	})
	if err != nil {
		return nil, err
	}
	return &KafkaPublisher{
		producer:      p,
		tradeTopic:    tradeTopic,
		snapshotTopic: snapshotTopic,
		log:           log,
	}, nil
}
```

*   **`acks=all`**: Ensures maximum durability by requiring acknowledgments from all in-sync replicas before a produce request is considered successful (see [[decisions/kafka-durability]]).
*   **`linger.ms=1`**: Introduces a micro-batching window of 1 millisecond to optimize throughput without compromising microsecond-level execution SLA requirements.