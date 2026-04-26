# 14. Messaging Systems & Architecture Concepts

---

### What constraints force systems to use asynchronous communication?

- **Temporal decoupling:** Producer and consumer don't need to be online at the same time
- **Scalability:** Synchronous calls create tight coupling; async allows independent scaling
- **Resilience:** If downstream service is down, messages queue up instead of failing immediately
- **Load leveling:** Absorb traffic spikes without overwhelming consumers
- **Long-running operations:** Offload time-consuming tasks to background workers
- **Fan-out:** One event needs to trigger multiple independent actions

---

### What are advantages of asynchronous messaging?

| Advantage | Description |
|-----------|-------------|
| **Decoupling** | Producers and consumers are independent; changes don't cascade |
| **Scalability** | Scale producers and consumers independently |
| **Resilience** | System continues working even if some components are down |
| **Buffering** | Absorbs traffic spikes; consumers process at their own pace |
| **Flexibility** | Easy to add new consumers without changing producers |
| **Auditability** | Message log provides a complete history of events |
| **Replay** | Reprocess historical events for debugging or new features |

---

### What are disadvantages of asynchronous communication?

- **Complexity:** More moving parts; harder to debug and trace
- **Eventual consistency:** Data may be temporarily out of sync across services
- **Message ordering:** Difficult to guarantee global ordering
- **Duplicate messages:** At-least-once delivery can cause duplicates; requires idempotent consumers
- **Latency:** Asynchronous processing introduces delay
- **Monitoring:** Need to track consumer lag, message throughput, error rates
- **Operational overhead:** Requires managing message brokers (Kafka, RabbitMQ, etc.)

---

### What are common messaging patterns?

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Point-to-Point (Queue)** | One producer, one consumer per message | Task distribution, job queues |
| **Publish-Subscribe** | One producer, multiple consumers (fan-out) | Event broadcasting, notifications |
| **Request-Reply** | Async request with correlation ID for response | Async RPC, command-query |
| **Event Sourcing** | Store all state changes as events | Audit logs, time travel, CQRS |
| **CQRS** | Separate read and write models | High-read systems, complex queries |
| **Saga** | Distributed transaction via compensating actions | Multi-service transactions |
| **Dead Letter Queue** | Failed messages routed to special queue | Error handling, poison messages |

---

### What is Publisher–Subscriber pattern?

In **Pub-Sub**, publishers send messages to a topic without knowing who (if anyone) will consume them. Multiple subscribers can independently consume the same message.

**Characteristics:**
- **Decoupling:** Publishers and subscribers don't know about each other
- **Fan-out:** One message delivered to all subscribers
- **Dynamic subscriptions:** Subscribers can join/leave without affecting publishers

**Kafka implementation:**
- Topic = channel
- Multiple consumer groups = independent subscribers
- Each group gets a copy of every message

**Example use case:** Order placed → notify inventory service, shipping service, analytics service

---

### What is Event-Driven Architecture?

**Event-Driven Architecture (EDA)** is a design pattern where services communicate by producing and consuming events (state changes or significant occurrences).

**Core concepts:**
- **Event:** Immutable fact about something that happened (e.g., "OrderPlaced")
- **Event Producer:** Service that publishes events
- **Event Consumer:** Service that reacts to events
- **Event Broker:** Middleware that routes events (Kafka, EventBridge, etc.)

**Benefits:**
- Loose coupling between services
- Real-time responsiveness
- Scalability and resilience
- Audit trail of all state changes

**Challenges:**
- Eventual consistency
- Complex debugging (distributed tracing needed)
- Event schema evolution

**Example flow:**
```
User places order → OrderService publishes "OrderPlaced" event
  → InventoryService reserves stock
  → PaymentService charges card
  → NotificationService sends email
```

---

### When should you use message queues instead of synchronous APIs?

**Use message queues when:**
- Operations are **long-running** (> 1 second)
- You need **resilience** to downstream failures
- You want to **decouple** services (independent deployment/scaling)
- You need to **buffer** traffic spikes
- You want **fan-out** (one event, multiple consumers)
- You need **guaranteed delivery** (at-least-once)
- You want to **replay** events

**Use synchronous APIs when:**
- You need **immediate response** (< 100ms)
- Operation is **simple and fast**
- You need **strong consistency** (read-your-writes)
- Failure should **immediately propagate** to the caller
- You want **simpler debugging** (single request trace)

**Hybrid approach:** Use sync for queries, async for commands (CQRS pattern).
