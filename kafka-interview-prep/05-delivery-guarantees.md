# 5. Delivery Guarantees & Exactly-Once Processing

---

### What is Exactly-Once Semantics (EOS)?

Exactly-Once Semantics (EOS) guarantees that each message is **processed exactly once** — no data loss and no duplicates — even in the presence of failures, retries, or network issues.

In Kafka, EOS covers:
- **Producer → Broker:** Idempotent producer prevents duplicate writes from retries
- **Broker → Broker (Streams):** Transactional API ensures atomic multi-partition writes
- **Consumer side:** `isolation.level=read_committed` ensures only committed messages are read

---

### How does Kafka implement Exactly-Once Processing?

EOS is built on two pillars:

**1. Idempotent Producer**
- Each producer gets a unique Producer ID (PID)
- Messages carry sequence numbers; broker deduplicates retries
- Enabled via: `enable.idempotence=true`

**2. Transactional API**
```java
props.put("transactional.id", "my-transactional-id");
producer.initTransactions();

producer.beginTransaction();
try {
    producer.send(record1);
    producer.send(record2);
    producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata); // For consume-transform-produce
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**Consumer side:**
```properties
isolation.level=read_committed  # Only reads messages from committed transactions
```

---

### How are delivery guarantees achieved in Kafka?

| Guarantee | Producer Config | Consumer Config | Risk |
|-----------|----------------|-----------------|------|
| **At-most-once** | `acks=0`, no retries | Auto-commit before processing | Data loss |
| **At-least-once** | `acks=all`, retries enabled | Manual commit after processing | Duplicates |
| **Exactly-once** | `enable.idempotence=true` + transactional API | `isolation.level=read_committed`, manual commit inside transaction | Performance overhead |

---

### What is the relationship between transactions and idempotency?

- **Idempotency** is a prerequisite for transactions — enabling `transactional.id` automatically enables idempotency
- **Idempotency alone** provides exactly-once per producer session for a single partition
- **Transactions** extend this to atomic writes across **multiple partitions and topics**
- Together they enable the **consume-transform-produce** pattern with exactly-once guarantees in Kafka Streams

**Scope comparison:**

| Feature | Idempotency | Transactions |
|---------|-------------|--------------|
| Scope | Single producer, single partition | Multiple partitions/topics |
| Atomicity | No | Yes (all-or-nothing) |
| Consumer visibility | Immediate | Only after commit (with `read_committed`) |
| Overhead | Low | Higher |
