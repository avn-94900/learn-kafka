# Kafka Streams, Connect, Monitoring & Performance — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Kafka Streams

### Q1. What is Kafka Streams? How is it different from Flink and Spark Streaming?

| | Kafka Streams | Apache Flink | Spark Streaming |
|---|---|---|---|
| Deployment | Library (runs in your app) | Separate cluster | Separate cluster |
| State management | RocksDB (local) | Managed state backend | Checkpoint to HDFS/S3 |
| Latency | Low (record-at-a-time) | Very low | Micro-batch (higher) |
| Exactly-once | Yes (with transactions) | Yes | Yes |
| Learning curve | Low (Kafka native) | High | Medium |
| Best for | Lightweight in-app stream processing | Complex stateful jobs at scale | Large-scale batch/stream analytics |

**Key point:** Kafka Streams is a library, not a cluster — it scales by adding application instances. No YARN, no Mesos needed.

---

### Q2. What is stateful vs stateless processing in Kafka Streams?

**Stateless:** Each record processed independently
- `filter()`, `map()`, `flatMap()`, `foreach()`, `branch()`

**Stateful:** Processing requires looking at accumulated state
- `count()`, `aggregate()`, `reduce()`, `join()`
- State stored in **state stores** (RocksDB locally, backed up to changelog topic in Kafka)
- State stores are partitioned — aligned with input topic partitions

---

### Q3. How do windowed operations work?

```java
stream
  .groupByKey()
  .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
  .count()
  .toStream()
  .to("order-counts-per-5min");
```

**Window types:**
- **Tumbling:** Fixed, non-overlapping (e.g., every 5 min)
- **Hopping:** Fixed size, overlapping (e.g., 5 min window, slide every 1 min)
- **Session:** Activity-based, closes after inactivity gap

**Late arrivals:** Use `.withGracePeriod(Duration.ofSeconds(30))` to accept late records. Without grace, late records are dropped.

---

## Kafka Connect

### Q4. How does Kafka Connect differ from Kafka Streams?

| | Kafka Connect | Kafka Streams |
|---|---|---|
| Purpose | Data integration (import/export) | Stream processing |
| Source | External systems (DB, S3, APIs) | Kafka topics |
| Output | Kafka topics or external systems | Kafka topics |
| Code required | Config-driven (no/low code) | Java/Scala code |

**Connect:** "Move data in and out of Kafka"
**Streams:** "Process data inside Kafka"

---

### Q5. What are source and sink connectors?

- **Source connector:** Pulls data from external system → writes to Kafka (e.g., Debezium for CDC from MySQL/Postgres)
- **Sink connector:** Reads from Kafka → writes to external system (e.g., Elasticsearch, S3, JDBC sink)
- Connectors run in **workers** (distributed or standalone mode)

---

### Q6. How does Avro + Schema Registry help schema evolution?

- Avro schemas are registered in the **Schema Registry** (Confluent or AWS Glue)
- Messages are serialized with a **schema ID** prefix (4 bytes) instead of full schema
- Consumer fetches schema by ID from registry → deserializes correctly
- Schema evolution rules:
  - **Backward compatible:** New schema can read old data (add fields with defaults)
  - **Forward compatible:** Old schema can read new data (remove optional fields)
  - **Full compatible:** Both directions
- Prevents consumer/producer coupling on schema versions

---

## Monitoring & Performance

### Q7. What are the most critical Kafka metrics to monitor?

| Metric | Location | Alert When |
|---|---|---|
| `records-lag-max` | Consumer | > threshold for sustained period |
| `records-consumed-rate` | Consumer | Drop unexpectedly |
| `request-latency-avg` | Producer | Spikes |
| `under-replicated-partitions` | Broker | > 0 (indicates replica lag or broker down) |
| `active-controller-count` | Broker | ≠ 1 (no controller = cluster issue) |
| `offline-partitions-count` | Broker | > 0 (data unavailable) |
| `network-io-rate` | Broker | Near saturation |

**Tools:** JMX → Prometheus JMX Exporter → Grafana, or Confluent Control Center, Datadog, Burrow for lag.

---

### Q8. How does Kafka achieve high throughput?

- **Sequential disk I/O:** Appends to log file — sequential writes are as fast as RAM
- **Zero-copy:** `sendfile()` syscall sends data from disk to network without user-space copy
- **Batching:** Producers batch multiple records per request; consumers fetch batches
- **Compression:** Per-batch compression (snappy/lz4/zstd) reduces network I/O
- **Partitioning:** Parallelism across brokers and consumers

---

### Q9. How do you tune Kafka for high throughput vs low latency?

**High throughput:**
```
linger.ms=20
batch.size=524288      # 512KB
compression.type=lz4
fetch.min.bytes=65536  # Consumer fetches larger batches
```

**Low latency:**
```
linger.ms=0
batch.size=16384       # default
acks=1                 # skip waiting for all ISR
fetch.min.bytes=1      # Consumer fetches immediately
```

---

### Q10. How do you handle large message sizes?

- Default max: 1MB (`message.max.bytes` on broker, `max.request.size` on producer)
- Options:
  1. **Increase limits** (broker: `message.max.bytes`, topic: `max.message.bytes`) — impacts broker memory
  2. **Store large payload externally** (S3, DB) and send a reference/pointer in Kafka message (recommended)
  3. **Compress the message** before sending
- Large messages increase GC pressure and slow down broker replication — avoid if possible
