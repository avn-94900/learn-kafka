# 3. Kafka Consumers

---

### What is the difference between automatic and manual offset commit?

| Aspect | Automatic Commit | Manual Commit |
|--------|-----------------|---------------|
| **Config** | `enable.auto.commit=true` | `enable.auto.commit=false` |
| **When committed** | Periodically (every `auto.commit.interval.ms`) | Explicitly by application code |
| **Risk** | Message loss (committed before processing) | Duplicate processing (crash before commit) |
| **Control** | Low | High |
| **Use case** | Simple, loss-tolerant consumers | Critical processing requiring at-least-once or exactly-once |

**Manual commit options:**
```java
consumer.commitSync();                          // Blocks until broker confirms
consumer.commitAsync((offsets, exception) -> {  // Non-blocking
    if (exception != null) log.error("Commit failed", exception);
});
```

---

### What is `auto.offset.reset` configuration?

Controls where a consumer starts reading when **no committed offset exists** for its group (new group or offset expired).

| Value | Behavior |
|-------|----------|
| `latest` (default) | Start from the newest messages (skip existing) |
| `earliest` | Start from the oldest available message |
| `none` | Throw exception if no offset found |

```properties
auto.offset.reset=earliest
```

This does **not** affect consumers that already have a committed offset — they always resume from where they left off.

---

### What happens if a consumer fails while processing a message?

**With auto-commit enabled:**
- If the consumer dies after auto-commit but before finishing processing → **message loss**
- On restart, consumer resumes from the auto-committed offset (skips unprocessed messages)

**With manual commit:**
- Consumer restarts and resumes from the last committed offset
- Messages processed but not yet committed will be **reprocessed** (at-least-once)

**Recommended pattern:**
```java
try {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
    consumer.commitSync(); // Commit only after successful processing
} catch (Exception e) {
    consumer.seek(partition, lastProcessedOffset + 1); // Resume from safe point
}
```

---

### What happens if a consumer crashes before committing offset?

- The consumer group coordinator detects the crash via missed heartbeats (after `session.timeout.ms`)
- A **rebalance** is triggered; the partition is reassigned to another consumer
- The new consumer starts from the **last committed offset**
- Messages processed by the crashed consumer but not committed will be **reprocessed** → duplicates possible
- Mitigation: Use idempotent processing logic or exactly-once semantics

---

### How do you configure manual acknowledgment?

**Spring Kafka:**
```java
@KafkaListener(topics = "my-topic", groupId = "my-group")
public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try {
        processRecord(record);
        ack.acknowledge(); // Manually commit offset
    } catch (Exception e) {
        // Do not acknowledge — message will be redelivered
    }
}
```

**application.yml:**
```yaml
spring:
  kafka:
    listener:
      ack-mode: MANUAL_IMMEDIATE  # or MANUAL
```

**Plain Java Kafka client:**
```properties
enable.auto.commit=false
```
```java
consumer.commitSync(Collections.singletonMap(
    new TopicPartition(record.topic(), record.partition()),
    new OffsetAndMetadata(record.offset() + 1)
));
```

---

### How does Kafka handle consumer rebalancing?

A **rebalance** redistributes partition assignments among consumers in a group. It is triggered when:
- A consumer joins the group
- A consumer leaves or crashes (detected via `session.timeout.ms` / `heartbeat.interval.ms`)
- Topic partitions are added

**Rebalance process:**
1. Group coordinator (broker) notifies all consumers to stop consuming
2. All consumers rejoin and submit their partition assignments
3. Coordinator assigns partitions using a partition assignor strategy (`RangeAssignor`, `RoundRobinAssignor`, `StickyAssignor`)
4. Consumers resume from their committed offsets

**Minimizing rebalance impact:**
- Use `StickyAssignor` to minimize partition movement
- Tune `session.timeout.ms` and `heartbeat.interval.ms` appropriately
- Use `max.poll.interval.ms` to avoid rebalance due to slow processing
- Kafka 2.4+ supports **incremental cooperative rebalancing** (consumers only pause affected partitions)
