# 9. Security & Reliability

---

### What are Kafka security mechanisms?

Kafka supports three security pillars:
1. **Authentication** — Who are you? (SASL, mTLS)
2. **Authorization** — What can you do? (ACLs)
3. **Encryption** — Is data protected in transit? (TLS/SSL)

---

### How do you implement Authentication?

**SASL mechanisms:**

| Mechanism | Description | Use Case |
|-----------|-------------|----------|
| `SASL/PLAIN` | Username/password (plaintext) | Simple setups (use with TLS) |
| `SASL/SCRAM-SHA-256/512` | Salted challenge-response | More secure than PLAIN |
| `SASL/GSSAPI` | Kerberos | Enterprise/on-prem environments |
| `SASL/OAUTHBEARER` | OAuth 2.0 tokens | Cloud-native, SSO integration |

**Broker config (SASL/SCRAM):**
```properties
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
sasl.enabled.mechanisms=SCRAM-SHA-256
```

**mTLS (mutual TLS):** Both broker and client present certificates — strongest authentication.

---

### How do you implement Authorization?

Kafka uses **ACLs (Access Control Lists)** managed via `kafka-acls.sh` or programmatically.

**Grant a consumer group read access:**
```bash
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:my-consumer \
  --operation Read --topic my-topic \
  --group my-consumer-group
```

**Grant a producer write access:**
```bash
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:my-producer \
  --operation Write --topic my-topic
```

**Broker config to enable ACLs:**
```properties
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
```

---

### How do you implement Encryption?

**TLS/SSL for in-transit encryption:**

**Broker config:**
```properties
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/var/ssl/kafka.server.keystore.jks
ssl.keystore.password=<keystore-password>
ssl.key.password=<key-password>
ssl.truststore.location=/var/ssl/kafka.server.truststore.jks
ssl.truststore.password=<truststore-password>
ssl.client.auth=required   # For mTLS
```

**Client config:**
```properties
security.protocol=SSL
ssl.truststore.location=/var/ssl/client.truststore.jks
ssl.truststore.password=<truststore-password>
```

**At-rest encryption:** Kafka does not natively encrypt data at rest — use encrypted disk volumes (AWS EBS encryption, LUKS, etc.).

---

### What are best practices for topic design?

- **Naming convention:** Use a consistent pattern, e.g., `<domain>.<entity>.<event>` → `payments.orders.created`
- **Partition count:** Start with `(target throughput) / (throughput per partition)`; consider future growth
- **Replication factor:** Use 3 for production; never less than 2
- **Retention:** Set based on downstream consumer SLA and storage budget
- **Compaction:** Use for event-sourcing / changelog topics where only latest value per key matters
- **Avoid too many topics:** Each topic/partition has overhead; consolidate where possible
- **Schema:** Always use a schema (Avro + Schema Registry) for structured data

---

### How do you prevent message loss?

**Producer side:**
```properties
acks=all                              # Wait for all ISR replicas
enable.idempotence=true               # Prevent duplicates from retries
retries=2147483647                    # Retry indefinitely
delivery.timeout.ms=120000            # Overall delivery timeout
```

**Broker side:**
```properties
replication.factor=3                  # At least 3 replicas
min.insync.replicas=2                 # Require at least 2 in-sync replicas
unclean.leader.election.enable=false  # Never elect out-of-sync replica as leader
```

**Consumer side:**
- Use manual offset commit
- Commit only after successful processing
- Handle exceptions and avoid silently swallowing errors
