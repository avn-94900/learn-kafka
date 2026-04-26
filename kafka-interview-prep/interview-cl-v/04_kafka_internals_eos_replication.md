# Kafka Internals, EOS & Replication — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Q1. What was ZooKeeper's role in Kafka and why is it being replaced by KRaft?

**ZooKeeper managed:**
- Broker registration and health
- Topic/partition metadata
- Leader election
- Consumer group coordination (legacy)

**Problems with ZooKeeper:**
- Separate system to operate and monitor
- Metadata bottleneck at scale (slow leader election for large clusters)
- Scaling limit: ~200K partitions in practice

**KRaft (Kafka Raft Metadata Mode):**
- Metadata managed by Kafka itself using Raft consensus
- Controller is a Kafka broker — no external dependency
- Faster leader election, millions of partitions supported
- GA since Kafka 3.3, ZooKeeper removal in Kafka 4.0

---

## Q2. How does Kafka leader election work?

- Each partition has one **leader** (handles all reads/writes) and N **followers**
- The **active controller** (one broker per cluster) manages leader assignments
- If a leader fails:
  1. Controller detects via ZooKeeper/KRaft
  2. Selects a new leader from the **ISR** (In-Sync Replicas)
  3. Updates partition metadata
  4. Brokers and clients refresh metadata
- `unclean.leader.election.enable=false` (recommended) prevents electing an out-of-sync replica (avoids data loss at the cost of unavailability)

---

## Q3. What is ISR and what happens when a replica falls out of ISR?

- **ISR (In-Sync Replicas):** Set of replicas that are caught up with the leader within `replica.lag.time.max.ms` (default: 30s)
- Replica falls out of ISR if:
  - It is too slow to fetch new messages
  - Network partition
  - Broker restart
- A fallen-out replica is still a **follower** but cannot be elected leader until it catches up and rejoins ISR
- `min.insync.replicas`: Minimum ISR size required to accept writes. If ISR drops below this → producer gets `NotEnoughReplicasException`

---

## Q4. What is Exactly-Once Semantics (EOS)? How does Kafka implement it?

EOS = a message is processed and its result is written **exactly once**, end-to-end.

**Components:**
1. **Idempotent producer** — prevents duplicate writes on producer retry (`enable.idempotence=true`)
2. **Transactions** — atomic write across multiple partitions/topics
   - `transactional.id` must be set on the producer
   - Use `beginTransaction()`, `send()`, `commitTransaction()` / `abortTransaction()`
3. **Consumer `isolation.level`:**
   - `read_uncommitted` (default) — reads all messages including aborted ones
   - `read_committed` — only reads committed transactional messages

**Full EOS pipeline:**
```
idempotent producer → transactional writes → consumer with read_committed
```

---

## Q5. What is the relationship between idempotency and transactions?

- Idempotency: prevents duplicate messages **within a single producer session and partition**
- Transactions: enables **atomic writes across multiple partitions** (all succeed or all fail)
- Transactions require idempotency to be enabled — `transactional.id` implicitly enables idempotence
- Idempotency alone does NOT give exactly-once across multiple topics or partitions

---

## Q6. How does Kafka replication work internally?

- Followers continuously fetch from leader using the same protocol as consumers
- Leader tracks **LEO (Log End Offset)** and **HW (High Watermark)** per follower
- **High Watermark:** The offset up to which all ISR replicas have replicated
- Consumers can only read up to the **High Watermark** — prevents reading uncommitted data
- `acks=all` means: leader waits until all ISR replicas have written before acknowledging

---

## Q7. How does MirrorMaker 2.0 work for cross-cluster replication?

- MirrorMaker 2 (MM2) is built on **Kafka Connect** framework
- Replicates topics, consumer group offsets, and ACLs between clusters
- Uses a **source connector** per source cluster
- Topic naming: `source.cluster.topic-name` (configurable)
- Supports **active-active** and **active-passive** topologies
- Tracks offsets via `__consumer_offsets` mapping between clusters
- Key improvement over MM1: exactly-once replication, offset sync, bidirectional replication

---

## Q8. How does Kafka achieve high availability? What configs matter most?

| Config | Recommendation | Why |
|---|---|---|
| `replication.factor` | 3 | Survive 2 broker failures |
| `min.insync.replicas` | 2 | Prevent writes to under-replicated partitions |
| `unclean.leader.election.enable` | `false` | No data loss from stale leader |
| `acks` | `all` | Full durability guarantee |
| `auto.leader.rebalance.enable` | `true` | Prevent hot-spots after broker restart |
