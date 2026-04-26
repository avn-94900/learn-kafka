# Unauthenticated (Plaintext) — Folder 2: Client Configuration

## Learning Path
msk-unauthenticated-learning/
├── folder-1-unauthenticated-design.md
├── folder-2-unauthenticated-client-configuration.md   ← YOU ARE HERE
└── folder-3-unauthenticated-controls-and-limits.md

---

## 1. Client Properties — Producer

The simplest possible Kafka client configuration.

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092

# No security config needed
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
```

No `security.protocol`, no `sasl.*`, no `ssl.*` — plaintext only.

---

## 2. Client Properties — Consumer

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092

group.id=dev-consumer-group
auto.offset.reset=earliest
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

---

## 3. CLI Tools — No Auth

No `--producer.config` or `--consumer.config` needed.

Produce:

```bash
kafka-console-producer.sh \
  --broker-list b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092 \
  --topic dev-test-topic
```

Consume:

```bash
kafka-console-consumer.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092 \
  --topic dev-test-topic \
  --group dev-consumer-group \
  --from-beginning
```

---

## 4. Port Reference — All MSK Auth Methods

| Port | Protocol    | Auth Method         |
|------|-------------|---------------------|
| 9092 | PLAINTEXT   | Unauthenticated     |
| 9094 | SSL         | mTLS                |
| 9096 | SASL_SSL    | SASL/SCRAM          |
| 9098 | SASL_SSL    | IAM                 |

Using the wrong port for your auth method will always result in a connection failure.

---

## 5. Getting the Bootstrap Broker Endpoints

```bash
aws kafka get-bootstrap-brokers \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid
```

Sample output:

```json
{
  "BootstrapBrokerString": "b-1.my-cluster...amazonaws.com:9092,b-2...:9092",
  "BootstrapBrokerStringTls": "b-1.my-cluster...amazonaws.com:9094,b-2...:9094",
  "BootstrapBrokerStringSaslScram": "b-1.my-cluster...amazonaws.com:9096,b-2...:9096",
  "BootstrapBrokerStringSaslIam": "b-1.my-cluster...amazonaws.com:9098,b-2...:9098"
}
```

Each auth method has its own bootstrap broker string.
Use the correct one for your chosen auth method.

---

## 6. Switching from Unauthenticated to IAM (Client-Side Change)

When the cluster is migrated to IAM auth, the client change is:

Before (unauthenticated):
```properties
bootstrap.servers=...amazonaws.com:9092
```

After (IAM):
```properties
bootstrap.servers=...amazonaws.com:9098
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

The port change (9092 → 9098) and the 4 added properties are the complete migration.

---

## Key Takeaway

Unauthenticated client config is the baseline — zero security properties.
Every other auth method adds exactly the security properties on top of this baseline.
The port number is the first thing to verify when a client cannot connect after an auth method change.
