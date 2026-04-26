# Kafka Consumers — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Q1. What is the difference between automatic and manual offset commit?

| | Auto Commit | Manual Commit |
|---|---|---|
| Config | `enable.auto.commit=true` | `enable.auto.commit=false` |
| When committed | Periodically (`auto.commit.interval.ms`) | After explicit `consumer.commitSync()` / `commitAsync()` |
| Risk | May commit before processing → loss on crash | Requires careful placement |
| Use case | Low-stakes, simple consumers | Financial, idempotent processing |

**Common trap:** Auto-commit commits the offset of the last `poll()` result, not the last processed message — a crash between poll and processing loses that batch.

---

## Q2. What does `auto.offset.reset` control?

- Determines where a **new consumer group** (or one with expired offsets) starts reading
- `earliest` — read from the beginning of the partition
- `latest` — read only new messages (default)
- `none` — throw exception if no offset found

**Scenario:** You deploy a new consumer group to a topic with 7 days of data → `earliest` to process all history, `latest` to only process going forward.

---

## Q3. What happens if a consumer crashes before committing the offset?

- On restart (or rebalance), the consumer re-reads from the last **committed offset**
- Messages between last commit and crash are **reprocessed → duplicates**
- Mitigation:
  - Make processing idempotent (so duplicates have no effect)
  - Use smaller `max.poll.records` to reduce reprocessing window
  - Use transactional outbox pattern for critical writes

---

## Q4. How does consumer rebalancing work? What triggers it?

**Triggers:**
- Consumer joins or leaves the group
- Consumer fails to send heartbeat within `session.timeout.ms`
- `poll()` not called within `max.poll.interval.ms` (processing too slow)
- Topic partition count changes

**Process (default `eager` protocol):**
1. All consumers stop consuming (stop-the-world)
2. Group coordinator reassigns partitions
3. Consumers resume

**Cooperative/incremental rebalancing** (Kafka 2.4+): Only affected partitions are reassigned — others continue processing. Enabled via `partition.assignment.strategy=CooperativeStickyAssignor`.

---

## Q5. What is `max.poll.interval.ms` and why is it critical?

- Maximum time between two consecutive `poll()` calls before the broker considers the consumer dead
- Default: 5 minutes
- If processing a batch takes longer than this → **consumer is kicked out → rebalance**
- Solutions:
  - Process asynchronously, call `poll()` from a separate thread (careful with threading)
  - Reduce `max.poll.records`
  - Increase `max.poll.interval.ms` if processing is expected to be slow
  - Use `pause()` / `resume()` on partitions during long processing

---

## Q6. How do you configure manual acknowledgment in Spring Kafka?

```java
// In KafkaListenerContainerFactory bean:
factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

// In listener:
@KafkaListener(topics = "orders")
public void listen(ConsumerRecord<?, ?> record, Acknowledgment ack) {
    process(record);
    ack.acknowledge(); // commit offset only after successful processing
}
```

**AckMode options:**
- `MANUAL` — buffered, committed on next `poll()`
- `MANUAL_IMMEDIATE` — committed right away
- `RECORD` — after each record (auto)
- `BATCH` — after each poll batch (auto)

---

## Q7. What is `session.timeout.ms` vs `heartbeat.interval.ms`?

- `heartbeat.interval.ms`: How often the consumer sends a heartbeat to the coordinator (default: 3s)
- `session.timeout.ms`: If no heartbeat received within this window → consumer is considered dead (default: 45s)
- **Rule:** `heartbeat.interval.ms` should be ≤ `session.timeout.ms / 3`
- These catch **network/process failures**, not slow processing (that's `max.poll.interval.ms`)
