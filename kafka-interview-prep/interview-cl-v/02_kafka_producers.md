# Kafka Producers — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Q1. What are the three ACKS modes and their trade-offs?

| `acks` | Meaning | Risk | Throughput |
|---|---|---|---|
| `0` | Fire and forget | Data loss on broker failure | Highest |
| `1` (default) | Leader acknowledges | Loss if leader crashes before replication | Medium |
| `all` / `-1` | All ISR replicas acknowledge | Safest | Lowest |

**Key:** `acks=all` alone is not enough — pair with `min.insync.replicas >= 2` to prevent a single-replica ISR accepting writes.

---

## Q2. How do retries work in Kafka producers? What is `delivery.timeout.ms`?

- On transient failures (network blip, leader change), the producer retries automatically
- Key configs:
  - `retries` — number of retry attempts (default: `MAX_INT` in newer clients)
  - `retry.backoff.ms` — wait between retries
  - `delivery.timeout.ms` — total time budget for one send including retries (default: 120s)
- If `delivery.timeout.ms` expires before success → `TimeoutException`
- **Ordering concern:** Without idempotency, retries can cause out-of-order delivery if `max.in.flight.requests.per.connection > 1`

---

## Q3. What is an idempotent producer and why does it matter?

- Enable with `enable.idempotence=true`
- Each producer gets a **Producer ID (PID)** and attaches a **sequence number** to every message
- Broker deduplicates retried messages using `(PID, partition, sequence_number)`
- Prevents **duplicate writes** caused by producer retries
- Automatically sets: `acks=all`, `retries=MAX_INT`, `max.in.flight.requests.per.connection=5`
- Required for exactly-once semantics

---

## Q4. What happens when a Kafka producer fails mid-send?

- If the broker hasn't acknowledged: the producer retries (up to `delivery.timeout.ms`)
- If idempotency is on: broker deduplicates retried messages
- If idempotency is off + `max.in.flight > 1`: possible duplicates **and** reordering
- If producer crashes entirely: unacknowledged messages are **lost** (no persistent queue on producer side)
- Best practice: idempotent producer + at-least-once consumer = safe pipeline

---

## Q5. How does `linger.ms` and `batch.size` affect throughput?

- `linger.ms`: Producer waits up to this duration to accumulate messages into a batch (default: 0 = send immediately)
- `batch.size`: Max bytes per batch per partition (default: 16KB)
- Higher `linger.ms` (e.g., 5–20ms) + larger `batch.size` = better throughput, slightly higher latency
- Compression (`compression.type=snappy/lz4/zstd`) is applied per batch — larger batches = better compression ratio

---

## Q6. How do you implement a custom serializer?

```java
public class OrderSerializer implements Serializer<Order> {
    @Override
    public byte[] serialize(String topic, Order data) {
        return objectMapper.writeValueAsBytes(data); // or Avro/Protobuf
    }
}
```

- Set via `producer.config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, OrderSerializer.class)`
- In Spring Boot Kafka: configure in `ProducerFactory` bean
- Prefer schema-based serializers (Avro + Schema Registry) in production for schema evolution

---

## Q7. What is `max.block.ms`?

- Time the producer's `send()` call blocks when the send buffer is full or metadata is unavailable
- Default: 60 seconds
- After this duration → `TimeoutException`
- Tune lower in latency-sensitive systems to fail fast instead of blocking threads
