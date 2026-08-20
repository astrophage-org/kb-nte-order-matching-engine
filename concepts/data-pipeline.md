# End-to-End Data Flow and Ingress/Egress Routing

This document details the end-to-end data pipeline within the Nexus Trading Exchange (NTE) Order Matching Engine (OME), managed under the Global Financial Markets Group (GFMG). It outlines how orders move from upstream gateways through internal memory structures and out to downstream enterprise consumers.

For architectural context, refer to the [[index]] and [[summaries/order-matching-engine]]. For implementation specifics of core pipeline components, see [[entities/cmd-server-main]], [[entities/internal-matching-engine]], [[entities/internal-matching-order-book]], and [[entities/internal-events-kafka-publisher]].

---

## Pipeline Overview

The OME pipeline is structured as a low-latency, event-driven stream processing loop divided into three primary phases:
1. **Ingress Layer**: Ingestion and validation of raw orders.
2. **Matching Core**: In-memory evaluation and execution using [[concepts/symbol-sharding]] and [[decisions/in-memory-queues]].
3. **Egress Layer**: Asynchronous distribution of execution reports and market data snapshots.

```
 [ Upstream Gateways ]
         │
         │ (nte.orders.inbound)
 ┌────────────────────────────────────────────────────────┐
 │ [[entities/cmd-server-main|Server Core]]               │
 └────────┬───────────────────────────────────────────────┘
         │ Routes models.Order
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ [[entities/internal-matching-engine|Engine]]           │ ──> [[concepts/symbol-sharding]]
 └────────┬───────────────────────────────────────────────┘
         │ Acquires Symbol Lock
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ [[entities/internal-matching-order-book|OrderBook]]    │ ──> [[concepts/iceberg-orders]]
 └────────┬───────────────────────────────────────────────┘
         │ Generates []models.TradeExecution
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ [[entities/internal-events-kafka-publisher|Publisher]] │ ──> [[decisions/kafka-durability]]
 └────────┬──────────────────────────────┬────────────────┘
         │                              │
         ▼ (nte.trades.matched)         ▼ (nte.orderbook.snapshots)
 [Trade Settlement & Compliance]   [Market Data Gateway]
```

---

## 1. Ingress Routing

Orders originate from external brokers, FIX gateways, or internal market data simulators. 

* **Inbound Topic**: `nte.orders.inbound` (defined in `config/exchange.yaml`).
* **Processing**: The [[entities/cmd-server-main]] orchestrator ingests raw order structures, validating parameters such as `Symbol`, `TraderID`, `Side`, `Type`, `Price`, and `Quantity`.
* **Domain Mapping**: Validated records are instantiated into [[entities/models-order]] domain models, which include support for complex types like [[concepts/iceberg-orders]] via `IsIceberg`, `DisplayQuantity`, and `HiddenQuantity` fields.

---

## 2. Internal State Routing and Matching

Once instantiated, orders are dispatched to the [[entities/internal-matching-engine]] utilizing [[concepts/symbol-sharding]] and [[decisions/in-memory-queues]].