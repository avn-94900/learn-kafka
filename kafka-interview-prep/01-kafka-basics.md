# 1. Kafka Basics & Core Concepts

---

### What is Apache Kafka and what problems does it solve?

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant, real-time data pipelines and event-driven architectures.

**Problems it solves:**
- **Decoupling:** Producers and consumers are independent; no direct dependency
- **Buffering:** Handles traffic spikes by absorbing bursts between systems
- **Scalability:** Horizontal scaling via partitions and brokers
- **Durability:** Persistent log storage with configurable retention
- **Replay:** Consumers can re-read historical messages
- **Fan-out:** One topic can be consumed by multiple independent consumers

---

### What are the key features of Kafka?

- **Distributed & Fault-Tolerant:** Data replicated across multiple brokers
- **High Throughput:** Sequential I/O, batching, zero-copy transfers
- **Scalable:** Add partitions and brokers without downtime
- **Durable:** Messages persisted to disk with configurable retention
- **Pull-based Consumers:** Consumers control their own read pace
- **Exactly-Once Semantics:** Via idempotent producers and transactions
- **Kafka Streams:** Built-in stream processing library
- **Kafka Connect:** Plug-and-play connectors for external systems

---

### What is a Kafka Topic and how is it different from a Queue?

**Kafka Topic:** A category or feed name to which records are published. Topics are partitioned for parallel processing and scalability.

| Aspect | Kafka Topic | Traditional Queue |
|--------|-------------|-------------------|
| **Message Persistence** | Retained based on policy (time/size) | Deleted after consumption |
| **Consumer Model** | Multiple consumers can read same message | One consumer per message |
| **Ordering** | Ordered within partition | FIFO ordering |
| **Scalability** | Horizontal via partitions | Limited |
| **Replay** | Consumers can replay messages | No replay |
| **Durability** | Configurable persistence | Usually volatile |

---

### What is the difference between Kafka and RabbitMQ?

| Feature | Apache Kafka | RabbitMQ |
|---------|--------------|----------|
| **Architecture** | Distributed commit log | Message broker with queues |
| **Message Persistence** | Always persistent | Optional |
| **Message Ordering** | Partition-level ordering | Queue-level FIFO |
| **Consumers** | Pull-based, can replay | Push/Pull, consumed once |
| **Scalability** | Horizontal via partitions | Vertical, clustering |
| **Protocol** | Custom binary protocol | AMQP, MQTT, STOMP |
| **Use Cases** | Event streaming, big data | Task queues, RPC patterns |
| **Latency** | Higher throughput, moderate latency | Lower latency, moderate throughput |
| **Message Routing** | Topic-based | Complex routing (exchanges, bindings) |

---

### What are topics and partitions in Kafka?

- **Topic:** A logical channel/category for messages. Producers write to topics; consumers read from them.
- **Partition:** A topic is split into one or more partitions. Each partition is an ordered, immutable sequence of records stored on disk.

**Why partitions matter:**
- Enable parallelism — multiple consumers in a group each read from separate partitions
- Determine max consumer parallelism (can't have more active consumers than partitions)
- Messages with the same key always go to the same partition (ordering guarantee)

---

### What are offsets and how are they managed?

An **offset** is a unique sequential integer assigned to each message within a partition. It represents the position of a consumer in the partition log.

**Offset Management:**
- **Auto-commit:** `enable.auto.commit=true` with `auto.commit.interval.ms`
- **Manual commit:** `commitSync()` (blocking) or `commitAsync()` (non-blocking)
- **Storage:** Stored in the internal `__consumer_offsets` topic
- **Per consumer group:** Each group tracks its own offsets independently

```java
consumer.commitSync();                        // Synchronous commit
consumer.commitAsync();                       // Asynchronous commit
consumer.seek(partition, offset);             // Reset to specific offset
```

---

### What are consumer groups and how do they work?

A **consumer group** is a set of consumers that jointly consume a topic. Kafka assigns each partition to exactly one consumer within the group.

**How it works:**
- Each partition is consumed by only one consumer in the group at a time
- Multiple groups can independently consume the same topic (fan-out)
- If a consumer leaves/joins, Kafka triggers a **rebalance** to redistribute partitions
- Group coordinator (a broker) manages group membership and offset commits

**Scaling rule:** Max parallelism = number of partitions. Extra consumers beyond partition count sit idle.

---

### How does Kafka ensure message durability?

- **Replication:** Each partition has `replication.factor` copies across different brokers
- **Acknowledgments:** Producer waits for leader (`acks=1`) or all ISR replicas (`acks=all`) to confirm write
- **Log Persistence:** Messages written to disk; configurable flush policies
- **Leader-Follower Model:** Each partition has one leader and N-1 followers

---

### How does Kafka ensure fault tolerance?

- **Broker Failure:** Automatic leader election from In-Sync Replicas (ISR)
- **Network Partitions:** `min.insync.replicas` prevents writes when too few replicas are in sync
- **Disk Failure:** Replication ensures data is available on other brokers
- **Producer Retries:** Configurable retry mechanism for failed sends
- **Consumer Failure:** Consumer group rebalance reassigns partitions to healthy consumers

---

### What is message retention policy and how is it configured?

Retention controls how long Kafka keeps messages before deleting them.

**Time-based:**
```properties
log.retention.hours=168        # Default: 7 days
log.retention.ms=604800000
```

**Size-based:**
```properties
log.retention.bytes=1073741824  # 1 GB per partition
```

**Compaction (keep latest value per key):**
```properties
log.cleanup.policy=compact
log.cleaner.enable=true
```

Can be set at broker level (global) or overridden per topic.

---

### Can each Kafka partition have a different retention period?

No — retention is configured at the **topic level**, not per partition. All partitions of a topic share the same retention settings. You can override broker defaults per topic:

```bash
kafka-configs.sh --alter --entity-type topics --entity-name my-topic \
  --add-config retention.ms=86400000
```

---

### How does Kafka ensure ordering of messages?

**Within a partition:** Messages are strictly ordered by offset. Consumers read them in the exact order they were written.

**Key-based routing:** Messages with the same key always go to the same partition (via hashing), preserving order for that key.

**Cross-partition:** No ordering guarantee. For global ordering, use a single partition (limits throughput).

**Producer config for strict ordering with retries:**
```java
props.put("max.in.flight.requests.per.connection", "1");
props.put("enable.idempotence", "true");
```

---

### How does Kafka handle backpressure and consumer lag?

**Backpressure:**
- **Producer-side:** `buffer.memory` and `max.block.ms` — producer blocks or throws if buffer is full
- **Consumer-side:** `max.poll.records` limits records per poll; consumer controls its own pace
- **Flow Control:** TCP flow control at the network level

**Consumer Lag Management:**
- Monitor `records-lag-max` metric per consumer group
- Scale out: add more consumer instances (up to partition count)
- Tune `max.poll.records` and processing batch size
- Set up lag-based alerts for proactive response

---

### What are the message delivery guarantees in Kafka?

| Semantic | Configuration | Use Case | Trade-offs |
|----------|---------------|----------|------------|
| **At-most-once** | `acks=0`, no retries | Metrics/logs where loss is acceptable | Fast, potential data loss |
| **At-least-once** | `acks=all`, retries enabled | Most common pattern | Possible duplicates |
| **Exactly-once** | `enable.idempotence=true` + transactional API | Financial transactions | Higher latency, complexity |

**Exactly-once producer config:**
```properties
enable.idempotence=true
acks=all
retries=2147483647
max.in.flight.requests.per.connection=5
```
