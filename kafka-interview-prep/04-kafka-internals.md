# 4. Kafka Internals & Coordination

---

### What is the role of ZooKeeper in Kafka?

ZooKeeper was Kafka's external coordination service (used in Kafka < 3.x by default):

- **Cluster Membership:** Tracks which brokers are alive
- **Leader Election:** Selects partition leaders when a broker fails
- **Configuration Management:** Stores topic configs, ACLs, broker metadata
- **Consumer Group Coordination:** (Legacy — now handled by brokers via group coordinator)
- **Controller Election:** Elects the Kafka controller broker that manages partition leadership

---

### Why is ZooKeeper being replaced by KRaft?

**ZooKeeper limitations:**
- Additional operational complexity — requires a separate ZooKeeper cluster
- Scaling bottleneck — all metadata changes go through ZooKeeper
- Network partition issues between Kafka and ZooKeeper
- Slower metadata propagation at large scale

**KRaft (Kafka Raft) benefits:**

| Aspect | ZooKeeper | KRaft |
|--------|-----------|-------|
| **Architecture** | External dependency | Self-contained (metadata stored in Kafka itself) |
| **Operational Complexity** | High (two systems to manage) | Lower (single system) |
| **Scaling** | Limited by ZooKeeper | Supports millions of partitions |
| **Performance** | Network overhead for metadata ops | Faster metadata operations |
| **Deployment** | Two separate clusters | Single Kafka cluster |

KRaft uses the **Raft consensus algorithm** internally. Kafka 3.3+ supports KRaft in production; ZooKeeper mode is deprecated in Kafka 3.x and removed in Kafka 4.0.

---

### How does Kafka perform leader election?

Each partition has one **leader** and N-1 **followers** (replicas).

**Election process:**
1. Kafka controller (a special broker) monitors broker health via ZooKeeper/KRaft
2. When a leader broker fails, the controller detects it
3. Controller selects a new leader from the **ISR (In-Sync Replicas)** list
4. Controller updates partition metadata and notifies all brokers
5. Producers and consumers redirect to the new leader

**Key config:**
```properties
unclean.leader.election.enable=false  # Prevents electing out-of-sync replicas (avoids data loss)
min.insync.replicas=2                 # Minimum replicas required for writes
```

---

### How does Kafka handle partition leadership changes?

**Scenarios that trigger leadership change:**
- Broker crash or restart
- Manual partition reassignment
- Preferred leader election (rebalancing leadership back to preferred replicas)

**What happens:**
1. Controller detects the leader is unavailable
2. New leader elected from ISR
3. Metadata propagated to all brokers via controller
4. Clients (producers/consumers) receive `NotLeaderForPartitionException` on next request
5. Clients automatically fetch updated metadata and reconnect to new leader

**Preferred leader election:**
- Kafka tracks the "preferred leader" (first replica in the replica list)
- Over time, leadership may drift; run preferred leader election to rebalance:
```bash
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED --all-topic-partitions
```
