# 10. High Availability & Replication

---

### How does Kafka ensure high availability?

- **Replication:** Each partition is replicated across multiple brokers (`replication.factor`)
- **Automatic leader election:** When a leader broker fails, a new leader is elected from ISR
- **No single point of failure:** Multiple brokers, multiple ZooKeeper/KRaft nodes
- **Consumer group rebalancing:** Partitions are reassigned if a consumer fails
- **Rack-aware replication:** Replicas spread across different racks/availability zones

**Minimum HA configuration:**
```properties
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
```

---

### How does replication work in Kafka?

- Each partition has one **leader** and `replication.factor - 1` **followers**
- Producers always write to the **leader**
- Followers continuously **fetch** from the leader to stay in sync
- Consumers by default read from the **leader** (Kafka 2.4+ supports follower reads)
- If a follower falls too far behind (`replica.lag.time.max.ms`), it is removed from ISR

**Replication flow:**
```
Producer → Leader (writes to log) → Followers fetch → Followers acknowledge → Leader responds to producer
```

---

### What is ISR (In-Sync Replica)?

**ISR (In-Sync Replicas)** is the set of replicas that are fully caught up with the leader's log.

- A replica is in ISR if it has fetched all messages within `replica.lag.time.max.ms`
- Only ISR members are eligible to become the new leader (when `unclean.leader.election.enable=false`)
- `min.insync.replicas` defines the minimum ISR size required for a write to succeed

**Example:**
- `replication.factor=3`, `min.insync.replicas=2`
- If 2 replicas are in ISR, writes succeed
- If only 1 replica is in ISR, writes fail with `NotEnoughReplicasException` — prevents data loss

---

### How does MirrorMaker 2.0 work?

MirrorMaker 2.0 (MM2) is a **Kafka Connect-based** tool for replicating data across Kafka clusters.

**Architecture:**
- Uses Kafka Connect source connectors to read from source cluster
- Writes to destination cluster
- Supports bi-directional replication
- Synchronizes consumer group offsets between clusters

**Configuration:**
```properties
clusters=primary,backup
primary.bootstrap.servers=primary-kafka:9092
backup.bootstrap.servers=backup-kafka:9092

primary->backup.enabled=true
primary->backup.topics=orders.*, payments.*
```

**Improvements over MirrorMaker 1:**
- No data loss during rebalancing
- Preserves message timestamps and headers
- Consumer group offset replication (enables failover without replay)
- Configurable replication policies
- Automatic topic creation on destination

---

### How do you replicate data across clusters?

**Options:**

1. **MirrorMaker 2.0** — Built-in, Connect-based, recommended for most use cases
2. **Confluent Replicator** — Enterprise, more features (schema registry sync, etc.)
3. **Custom Kafka Connect pipeline** — Source connector on cluster A, sink connector to cluster B

**Use cases:**
- **Disaster Recovery:** Active-passive setup; failover to backup cluster
- **Geo-distribution:** Serve consumers from a nearby cluster
- **Data Center Migration:** Gradually move workloads
- **Dev/Staging:** Mirror production data to lower environments

**Failover considerations:**
- MM2 translates offsets between clusters (offset mapping topic)
- Consumers can use `RemoteClusterUtils` to translate offsets for seamless failover
- Topic names on destination are prefixed by default: `primary.orders` (configurable)
