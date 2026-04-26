# 16. Real-World Design & Troubleshooting

---

### Design a real-time data pipeline using Kafka

**Architecture:**
```
[Data Sources] → [Kafka Connect / Producers] → [Kafka Cluster] → [Stream Processing] → [Storage / Analytics]
```

**Component selection:**

| Layer | Component | Purpose |
|-------|-----------|---------|
| Ingestion | Kafka Connect (Debezium, JDBC) | CDC from databases |
| Ingestion | Custom Producers | Application events, IoT |
| Schema | Schema Registry (Avro) | Schema governance |
| Processing | Kafka Streams / KSQL | Lightweight transformations |
| Processing | Apache Flink | Complex event processing |
| Storage | Elasticsearch | Real-time search |
| Storage | S3 via Kafka Connect | Long-term archival |
| Monitoring | Prometheus + Grafana | Metrics and alerting |

**Topic configuration for high throughput:**
```properties
num.partitions=12
replication.factor=3
min.insync.replicas=2
retention.ms=604800000
compression.type=lz4
```

**Producer optimization:**
```properties
batch.size=32768
linger.ms=10
buffer.memory=67108864
enable.idempotence=true
```

**Consumer optimization:**
```properties
max.poll.records=1000
fetch.min.bytes=50000
fetch.max.wait.ms=500
```

**Key design decisions:**
- Partition by entity ID (e.g., `userId`, `orderId`) for ordering and locality
- Use Avro + Schema Registry for schema evolution
- Set `replication.factor=3`, `min.insync.replicas=2` for durability
- Monitor consumer lag continuously; alert if lag grows

---

### How would you debug consumer lag?

**Step 1 — Measure lag:**
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-consumer-group
```
Look at `LAG` column per partition.

**Step 2 — Identify the cause:**

| Symptom | Likely Cause |
|---------|-------------|
| Lag growing on all partitions | Consumer processing too slow |
| Lag on specific partitions | Uneven partition assignment or hot partition |
| Lag spikes then recovers | Temporary processing slowdown or GC pause |
| Lag constant but high | Consumer count < partition count |

**Step 3 — Remediation:**
- **Scale out:** Add more consumer instances (up to partition count)
- **Optimize processing:** Profile and optimize the consumer logic
- **Increase partitions:** If consumers are maxed out
- **Tune `max.poll.records`:** Reduce if processing per batch is too slow
- **Check `max.poll.interval.ms`:** Increase if processing legitimately takes long

---

### How do you troubleshoot high latency?

**Producer latency:**
- Check `request-latency-avg` and `request-latency-max` JMX metrics
- Reduce `linger.ms` if batching is adding unnecessary delay
- Check broker CPU and disk I/O
- Verify `acks` setting — `acks=all` is slower than `acks=1`

**Consumer latency (end-to-end):**
- Measure time from message produce timestamp to processing completion
- Check consumer lag — high lag = high end-to-end latency
- Profile consumer processing code for bottlenecks
- Check if consumer is blocked on external calls (DB, HTTP)

**Broker latency:**
- Check `RequestHandlerAvgIdlePercent` — low value = broker overloaded
- Check disk I/O utilization — SSDs significantly reduce latency
- Check network bandwidth saturation
- Check GC pauses in broker JVM logs

---

### How do you resolve broker failure?

**Immediate response:**
1. Kafka automatically elects new partition leaders from ISR — no manual action needed
2. Verify `UnderReplicatedPartitions` metric drops back to 0 after election
3. Check `OfflinePartitionsCount` — should be 0 (non-zero means data unavailable)

**If broker is permanently lost:**
```bash
# Reassign partitions to remaining brokers
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json --execute
```

**Prevention:**
- Use `replication.factor=3` so 1 broker failure is tolerated
- Set `unclean.leader.election.enable=false` to prevent data loss
- Monitor `UnderReplicatedPartitions` and alert immediately

---

### What are common Kafka production issues?

| Issue | Symptoms | Resolution |
|-------|----------|------------|
| **Consumer lag** | Growing lag metric | Scale consumers, optimize processing |
| **Under-replicated partitions** | `UnderReplicatedPartitions > 0` | Check broker health, network |
| **Disk full** | Write failures, broker crash | Reduce retention, add storage |
| **Rebalance storm** | Frequent rebalances, processing pauses | Tune `session.timeout.ms`, use sticky assignor |
| **Poison messages** | Consumer stuck, DLT filling up | Fix schema, add error handling |
| **Producer buffer full** | `BufferExhaustedException` | Increase `buffer.memory`, reduce `linger.ms` |
| **Offset commit failures** | Duplicate processing | Check network, increase `request.timeout.ms` |
| **Schema incompatibility** | Deserialization errors | Fix schema, use backward-compatible changes |

---

### How do you handle network partition scenarios?

A **network partition** splits the Kafka cluster into isolated groups that can't communicate.

**What Kafka does:**
- Brokers that can't reach ZooKeeper/KRaft controller are fenced off
- Controller detects missing heartbeats and triggers leader election
- `min.insync.replicas` prevents writes if not enough replicas are reachable

**Configuration to minimize data loss:**
```properties
unclean.leader.election.enable=false   # Never elect out-of-sync replica
min.insync.replicas=2                  # Require 2 in-sync replicas for writes
acks=all                               # Producer waits for all ISR
```

**Trade-off:** With `min.insync.replicas=2` and a partition that isolates 2 of 3 replicas, writes will fail (availability sacrificed for consistency — CP in CAP theorem).

**Producer behavior during partition:**
- Producer receives `NotEnoughReplicasException`
- Retries until `delivery.timeout.ms` is reached
- Application should handle this exception and alert

**Recovery:**
- Once network heals, isolated brokers rejoin and replicate missing messages
- Consumers resume from committed offsets — no data loss if `unclean.leader.election.enable=false`
