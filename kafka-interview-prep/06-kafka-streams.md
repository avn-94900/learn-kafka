# 6. Kafka Streams & Data Processing

---

### What is Kafka Streams?

Kafka Streams is a **client library** (not a separate cluster) for building real-time stream processing applications that read from and write to Kafka topics. It runs inside your application process — no separate infrastructure needed.

**Key characteristics:**
- Exactly-once processing semantics
- Stateful and stateless operations
- Event-time and processing-time support
- Fault-tolerant via Kafka changelog topics
- Scales by adding application instances

---

### What are the key concepts in Kafka Streams?

- **KStream:** Unbounded stream of records (each record is an independent event)
- **KTable:** Changelog stream representing the latest value per key (like a materialized view)
- **GlobalKTable:** Fully replicated KTable available on all instances (for lookups/joins)
- **Topology:** The processing graph of sources, processors, and sinks
- **State Store:** Local storage (RocksDB) for stateful operations; backed up to Kafka changelog topics
- **Stream Task:** Unit of parallelism; one task per partition

**Example:**
```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> stream = builder.stream("input-topic");

stream
    .filter((key, value) -> value != null)
    .mapValues(String::toUpperCase)
    .to("output-topic");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

---

### How is Kafka Streams different from Apache Flink and Spark Streaming?

| Feature | Kafka Streams | Apache Flink | Spark Streaming |
|---------|---------------|--------------|-----------------|
| **Deployment** | Library (embedded in app) | Separate cluster framework | Separate cluster framework |
| **Processing Model** | Event-by-event, exactly-once | Event-time, exactly-once | Micro-batching |
| **State Management** | Local RocksDB + Kafka changelog | Managed state with checkpoints | External state stores |
| **Latency** | Low (milliseconds) | Very low (milliseconds) | Medium (seconds) |
| **Fault Tolerance** | App restart + state restore | Checkpointing | Driver restart |
| **Scaling** | Add app instances | Task parallelism | Executor scaling |
| **Learning Curve** | Low | High | Medium |
| **Data Sources** | Kafka only | Multiple (Kafka, files, DB, etc.) | Multiple |

**When to use:**
- **Kafka Streams:** Kafka-centric, simple-to-medium complexity, no cluster overhead
- **Flink:** Complex event processing, very low latency, multiple data sources
- **Spark:** Large-scale batch + streaming, ML integration

---

### What is stateful vs stateless processing?

**Stateless processing:**
- Each record is processed independently with no memory of previous records
- Operations: `filter`, `map`, `mapValues`, `flatMap`, `branch`
- No state store needed; easy to scale

**Stateful processing:**
- Processing depends on accumulated state from previous records
- Operations: `aggregate`, `count`, `reduce`, `join`, `windowed aggregations`
- Requires a **state store** (local RocksDB, backed by Kafka changelog topic)
- State is fault-tolerant and restored on restart

```java
// Stateless
stream.filter((k, v) -> v.contains("ERROR"));

// Stateful — count events per key
stream.groupByKey()
      .count(Materialized.as("event-counts"));
```

---

### How do you handle windowed operations?

Windows group records by time for aggregations.

**Window types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Tumbling** | Fixed, non-overlapping windows | Hourly totals |
| **Hopping** | Fixed size, overlapping (advance < size) | Rolling averages |
| **Sliding** | Windows defined by time difference between records | Event correlation |
| **Session** | Dynamic, gap-based windows | User session tracking |

**Example — tumbling window (count per 1 minute):**
```java
stream.groupByKey()
      .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
      .count()
      .toStream()
      .to("windowed-counts");
```

**Grace period** allows late-arriving records to be included:
```java
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(10))
```
