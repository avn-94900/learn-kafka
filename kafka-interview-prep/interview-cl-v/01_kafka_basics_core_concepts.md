# Kafka Basics & Core Concepts — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Q1. How is a Kafka Topic different from a traditional Queue?

| | Queue | Kafka Topic |
|---|---|---|
| Consumption | One consumer per message | Multiple consumer groups, each reads independently |
| Retention | Message deleted after consumption | Retained by time/size policy |
| Replay | Not possible | Possible by resetting offsets |
| Ordering | FIFO (single queue) | Per-partition ordering only |

**Key point:** Kafka is a distributed log, not a queue. Messages are not removed on consumption.

---

## Q2. Kafka vs RabbitMQ — when would you choose which?

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Pull-based log | Push-based broker |
| Throughput | Very high (millions/sec) | Moderate |
| Message replay | Yes | No (once consumed, gone) |
| Routing | Limited (topic/partition) | Rich (exchanges, bindings) |
| Ordering | Per partition | Per queue |
| Best for | Event streaming, audit log, high-throughput pipelines | Task queues, RPC, complex routing |

---

## Q3. How do topics, partitions, and consumer groups interact?

- A topic is split into **N partitions**
- Each partition is assigned to **exactly one consumer** within a consumer group
- Multiple groups → each group gets all messages (independent offsets)
- Adding more consumers than partitions → idle consumers
- **Rule:** `parallelism = min(consumers, partitions)`

---

## Q4. How does Kafka manage offsets? What is `__consumer_offsets`?

- Offsets are stored in the internal topic `__consumer_offsets` (since Kafka 0.9; previously ZooKeeper)
- Consumers commit the **next offset to be read** (not last processed)
- Offset commit can be **automatic** (periodic) or **manual** (after processing)
- Each `(group, topic, partition)` tuple has its own committed offset

---

## Q5. How does Kafka guarantee message ordering?

- Ordering is guaranteed **within a partition only**
- Use a consistent **partition key** (e.g., user ID, order ID) to ensure related events land in the same partition
- No cross-partition ordering guarantee
- Increasing partition count after creation can break ordering for key-based partitioning

---

## Q6. What are the three message delivery semantics? Which is default?

| Semantic | How | Risk |
|---|---|---|
| At-most-once | Commit offset before processing | Message loss |
| At-least-once | Commit offset after processing | Duplicates on retry |
| Exactly-once | Idempotent producer + transactions | Highest overhead |

**Default:** At-least-once (manual commit after processing). Exactly-once requires explicit configuration.

---

## Q7. How does Kafka ensure fault tolerance?

- **Replication:** Each partition has 1 leader + N-1 follower replicas
- **ISR (In-Sync Replicas):** Only replicas caught up with the leader are in ISR
- On leader failure, a new leader is elected from ISR
- `min.insync.replicas` controls the minimum ISR count before a write is accepted

---

## Q8. How does Kafka handle consumer lag and backpressure?

- **Consumer lag** = latest offset − committed offset per partition
- Kafka has **no native backpressure** — the consumer pulls at its own rate
- Solutions:
  - Scale consumers (up to partition count)
  - Increase `max.poll.records` or reduce processing time
  - Use async processing or batch processing in consumer
  - Add more partitions (careful: irreversible if already at target)
- Monitor lag with `kafka-consumer-groups.sh --describe` or tools like Burrow/Grafana

---

## Q9. How is message retention configured? Can it be per-partition?

- Configured at **topic level**, not per-partition:
  - `retention.ms` — time-based (default 7 days)
  - `retention.bytes` — size-based per partition
- Both can be overridden per topic using `kafka-configs.sh --alter`
- **Per-partition retention is not supported** — only per-topic

---

## Q10. What is the difference between `log.retention.ms` and `log.segment.ms`?

- `log.retention.ms`: How long a log **segment** is kept after it is closed
- `log.segment.ms` / `log.segment.bytes`: When to **roll over** to a new segment
- A segment is only eligible for deletion once it is **closed and older than retention time**
- Small `segment.bytes` + high `retention.ms` → many small files
