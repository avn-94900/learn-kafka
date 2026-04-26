# 8. Monitoring, Tuning & Performance

---

### How do you monitor Kafka?

**Monitoring layers:**
- **JMX Metrics:** Kafka exposes metrics via JMX; scrape with Prometheus JMX Exporter
- **Kafka CLI tools:** `kafka-consumer-groups.sh`, `kafka-topics.sh` for quick checks
- **Monitoring platforms:** Prometheus + Grafana, Confluent Control Center, Datadog, New Relic

**Common monitoring stack:**
```
Kafka Brokers (JMX) → Prometheus JMX Exporter → Prometheus → Grafana Dashboards
```

---

### What metrics are most important?

**Consumer Lag:**
```
kafka.consumer:type=consumer-fetch-manager-metrics,name=records-lag-max
```
- High lag = consumers can't keep up with producers
- Alert threshold depends on SLA; typically alert if lag grows continuously

**Throughput:**
```
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
```

**Latency:**
```
kafka.producer:type=producer-metrics,name=request-latency-avg
kafka.producer:type=producer-metrics,name=request-latency-max
```

**Disk Usage:**
```
kafka.log:type=LogSize,name=Size
```
- Monitor per broker and per topic to prevent disk exhaustion

**Other critical metrics:**
- `UnderReplicatedPartitions` — partitions not fully replicated (data risk)
- `ActiveControllerCount` — should always be 1
- `OfflinePartitionsCount` — should always be 0
- `LeaderCount` per broker — should be balanced

---

### How does Kafka achieve high throughput?

- **Sequential I/O:** Log-structured append-only writes; OS page cache for reads
- **Zero-copy:** `sendfile()` syscall transfers data from disk to network without copying to user space
- **Batching:** Producer batches multiple records (`batch.size`, `linger.ms`); reduces network round trips
- **Compression:** Compress batches with `snappy`, `lz4`, or `zstd`
- **Partitioning:** Multiple partitions allow parallel writes and reads

---

### How does Kafka achieve horizontal scalability?

| Component | Scaling Method | Benefit |
|-----------|----------------|---------|
| **Partitions** | Increase partition count | More parallel producers/consumers |
| **Brokers** | Add more brokers | Distribute partition load |
| **Consumers** | Add instances to consumer group | Parallel consumption (up to partition count) |
| **Producers** | Multiple producer instances | Increased write throughput |

**Note:** You can increase partitions but cannot decrease them without recreating the topic.

---

### How do you tune Kafka for optimal performance?

**Producer tuning:**
```properties
batch.size=65536          # Larger batches = fewer requests
linger.ms=10              # Wait up to 10ms to fill batch
compression.type=lz4      # Fast compression
buffer.memory=67108864    # 64MB producer buffer
acks=1                    # Balance durability vs throughput
```

**Consumer tuning:**
```properties
max.poll.records=500      # Records per poll
fetch.min.bytes=50000     # Wait for at least 50KB before returning
fetch.max.wait.ms=500     # Max wait time for fetch.min.bytes
```

**Broker tuning:**
```properties
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
log.flush.interval.messages=10000   # Flush to disk every 10K messages
```

**OS-level:**
- Use SSDs for Kafka log directories
- Increase file descriptor limits (`ulimit -n`)
- Tune TCP settings (`net.core.rmem_max`, `net.core.wmem_max`)
- Use XFS or ext4 filesystem

---

### How do you handle large message sizes?

Kafka is optimized for small-to-medium messages (< 1MB). For large messages:

**Option 1 — Increase size limits (not recommended for very large messages):**
```properties
# Broker
message.max.bytes=10485760          # 10MB

# Topic
max.message.bytes=10485760

# Consumer
max.partition.fetch.bytes=10485760

# Producer
max.request.size=10485760
```

**Option 2 — Store payload externally (recommended):**
- Store large payload in S3, HDFS, or a database
- Send only a reference (URL/ID) in the Kafka message
- Consumer fetches the actual payload using the reference

**Option 3 — Split large messages:**
- Break large messages into chunks at the producer
- Reassemble at the consumer using a correlation ID and sequence number
