# Messaging Patterns, Idempotency & Real-World Design — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Messaging Patterns & Architecture

### Q1. What constraints force systems to use asynchronous communication?

- **Temporal decoupling:** Producer and consumer don't need to be available simultaneously
- **Speed mismatch:** Producer generates faster than consumer can process
- **Resilience:** Downstream failures shouldn't crash the upstream caller
- **Fan-out:** One event needs to trigger multiple independent consumers
- **Audit trail:** Need a persistent, replayable record of events

---

### Q2. Advantages and disadvantages of async messaging

**Advantages:**
- Decouples services (producer doesn't wait for consumer)
- Absorbs burst traffic (Kafka as buffer)
- Enables event sourcing and audit logs
- Natural retry and replay support

**Disadvantages:**
- Harder to reason about consistency (eventual consistency)
- Debugging distributed flows is complex
- Response is not immediate — not suitable for request/response
- Consumer failure can cause silent processing gaps if lag not monitored

---

### Q3. When should you use Kafka instead of a synchronous REST/gRPC API?

| Use Kafka | Use REST/gRPC |
|---|---|
| Event-driven workflows (order placed → notify, invoice, ship) | Real-time response required (search, auth) |
| High-throughput data pipelines | Low-latency one-to-one calls |
| Audit log / event sourcing | Simple CRUD operations |
| Decoupling microservices | Strong consistency needed synchronously |
| Fan-out to multiple consumers | Single consumer with tight SLA |

---

### Q4. What is the Publisher-Subscriber pattern and how does Kafka implement it?

- **Pub-Sub:** Publishers emit events without knowing who will consume them
- In Kafka: a producer publishes to a topic; multiple consumer groups subscribe independently
- Each group maintains its own offset → **each gets a full copy of all messages**
- Kafka extends Pub-Sub with **persistence** (messages available for replay), unlike traditional brokers

---

## Idempotency & Reliability

### Q5. What are idempotency keys and how do they prevent duplicates?

- An **idempotency key** is a unique identifier for an operation (e.g., `orderId`, `paymentId`, `eventId`)
- On retry, the same key is submitted; the server recognizes it and returns the previous result without re-executing

```sql
-- Database pattern:
INSERT INTO payments (payment_id, amount, status)
VALUES (:paymentId, :amount, 'SUCCESS')
ON CONFLICT (payment_id) DO NOTHING;
```

- Can also use Redis: `SET payment:{id} 1 NX EX 86400` (set if not exists, expire in 24h)

---

### Q6. What are retry strategies? Which is best for distributed systems?

| Strategy | Description | Pros | Cons |
|---|---|---|---|
| Fixed retry | Retry every N ms | Simple | Thundering herd under load |
| Exponential backoff | Double delay each attempt | Reduces pressure | Can grow too large |
| Exponential + jitter | Random spread on delay | Prevents synchronized retries | Slightly more complex |
| Circuit breaker | Stop retrying after threshold | Prevents cascade failure | Needs state management |

**Best practice:** Exponential backoff with jitter + circuit breaker (e.g., Resilience4j) for microservices.

---

### Q7. What is the role of timeouts and deadlines?

- **Timeout:** Maximum wait per request attempt
- **Deadline:** Maximum total time for the entire operation (across all retries)
- Without deadlines, retries can run indefinitely and pile up
- In Kafka: `delivery.timeout.ms` is the deadline for a producer send
- In service calls: use deadlines (not just timeouts per attempt) to propagate urgency to downstream

---

## Real-World Design & Troubleshooting

### Q8. Design a real-time order processing pipeline using Kafka

```
[Order Service]
     ↓ produces to
[orders topic] (partitioned by orderId)
     ↓ consumed by
[Inventory Service] → updates stock
[Notification Service] → sends email/SMS
[Analytics Service] → updates dashboard

On failure:
[orders topic] → retry topics → [orders-dlt]
                                     ↓
                              [DLT Monitor] → alert + manual review
```

Key design decisions:
- Partition key = `orderId` → all events for one order go to same partition → ordered processing
- Replication factor = 3, `min.insync.replicas` = 2
- Consumers use `read_committed` if EOS is required
- DLT monitoring with alert + retry tooling

---

### Q9. How would you debug consumer lag?

```bash
# Check lag per group/partition
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group my-group

# Output: LAG column per partition
```

**Investigation steps:**
1. Is lag growing or stable? → Growing = consumer can't keep up
2. Is it one partition or all? → One = hot partition (uneven key distribution), All = consumer too slow
3. Check CPU/memory of consumer instances
4. Check downstream dependency (DB, API) latency
5. Check `max.poll.records` — reducing it helps if processing per record is slow

---

### Q10. How do you resolve a broker failure? What is the recovery sequence?

1. **Immediate:** Controller detects broker down, promotes ISR replica to leader for affected partitions
2. **Producer/consumer:** Automatically reconnect and refresh metadata (retries with backoff)
3. **When broker recovers:** Rejoins cluster, syncs from current leaders
4. **After sync:** Added back to ISR once caught up within `replica.lag.time.max.ms`
5. **Monitor:** Watch `under-replicated-partitions` metric → should return to 0 after recovery

**Prevent data loss:** `unclean.leader.election.enable=false` ensures only ISR replicas can become leaders.

---

### Q11. How do you handle a network partition scenario in Kafka?

- Kafka uses **quorum-based writes** via `min.insync.replicas`
- If brokers on one side can't reach majority → writes are rejected (safer than split-brain writes)
- With `acks=all` + `min.insync.replicas=2` + `replication.factor=3`:
  - Minority partition: writes fail (correct behavior)
  - Majority partition: cluster continues operating
- Producers get `NotEnoughReplicasException` → should have upstream alerting
- KRaft further improves this with Raft's majority-based consensus

---

### Q12. What are the best practices for Kafka topic design?

- **Partition count:** Start with expected parallelism (number of consumers). Don't over-partition — each partition has overhead (memory, file handles). Rule of thumb: `partitions = max expected consumers`
- **Replication factor:** Always 3 in production
- **Message key:** Choose a key that distributes load evenly and groups related events (avoid using timestamps or random UUIDs as keys for ordered processing)
- **Topic per event type** vs **topic per domain**: Prefer per-event-type for independent scaling; per-domain for coupled consumers
- **Avoid too many topics:** Each topic × partition × replication factor = file handles on broker
- **Naming convention:** `<domain>.<entity>.<action>` (e.g., `orders.order.placed`)
