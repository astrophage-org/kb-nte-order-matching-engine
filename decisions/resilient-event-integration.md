# Resilient Event Integration: Kafka Fallback Mechanisms

## Status
Approved / Implemented

## Context
The Nexus Trading Exchange (NTE) [[summaries/architectural-overview]] Order Matching Engine (OME) relies heavily on Apache Kafka for asynchronous event broadcasting [[concepts/event-driven-architecture]], including publishing matched trades to `nte.trades.matched` for downstream consumption by the [[decisions/sister-system-integration|Trade Settlement System and Compliance Surveillance Monitor]], and publishing L2 book snapshots to `nte.orderbook.snapshots` for the [[summaries/architectural-overview|Market Data Gateway]].

However, during local development, offline testing, or standalone simulation runs, a fully operational Kafka cluster (`kafka-cluster.nte.gfmg.internal:9092`) may not be reachable. Requiring a hard dependency on an external broker cluster for every local execution hinders developer velocity and prevents isolated debugging of core [[concepts/order-matching]] and [[entities/order-book]] logic.

## Decision
We implement a **graceful degradation and fallback mechanism** within the server lifecycle initialization (`cmd/server/main.go`) and the event publishing layer (`internal/events/kafka_publisher.go`). 

1. **Non-Blocking Broker Connection**: When `events.NewKafkaPublisher` attempts to initialize the underlying `confluent-kafka-go` producer, connection failures or timeout errors are caught rather than triggering a fatal application panic.
2. **Fallback Mode Logging**: If the broker connection fails, the server logs a structured warning (`Failed to connect to Kafka (running in fallback mode)`) via `logrus` and permits the application to continue running.
3. **Nil-Safe Publisher Guarding**: In `cmd/server/main.go`, the `publisher` instance may be `nil` when operating in standalone/fallback mode. Traffic processing loops (`processTraffic`) explicitly guard all dispatch actions:
   ```go
   if len(trades) > 0 && publisher != nil {
       publisher.PublishTrades(trades)
       publisher.PublishSnapshot(order.Symbol, map[string]interface{}{"bids": 10, "asks": 5})
   }
   ```
4. **Core Isolation**: This ensures that core components like the [[entities/engine]] and [[entities/order-book]] continue to execute matching simulations, validate [[entities/domain-models|Order and TradeExecution models]], and exercise [[decisions/memory-driven-concurrency|memory-driven concurrency]] cleanly without crashing due to absent network infrastructure.