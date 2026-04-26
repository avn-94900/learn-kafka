# SASL/SCRAM — Folder 2: Client Configuration

## Learning Path
msk-sasl-scram-learning/
├── folder-1-scram-authorization-design.md
├── folder-2-scram-client-configuration.md   ← YOU ARE HERE
└── folder-3-scram-acls-and-production.md

---

## 1. Core Client Properties

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096

# SCRAM over TLS — port 9096
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512

# Credentials inline (dev/test only — never production)
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="svc-orders-producer" \
  password="str0ngP@ssword123!";
```

> Port 9096 is the SASL/SCRAM listener. Port 9098 is IAM. Port 9094 is mTLS.
> Inline credentials in jaas.config are acceptable only for local testing.

---

## 2. Producer Configuration

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="svc-orders-producer" \
  password="str0ngP@ssword123!";

key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
retries=3
```

---

## 3. Consumer Configuration

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="svc-orders-consumer" \
  password="str0ngP@ssword123!";

group.id=orders-consumer-group
auto.offset.reset=earliest
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

---

## 4. Loading Credentials from Secrets Manager at Runtime (Production Pattern)

Never put passwords in config files. Fetch from Secrets Manager at startup:

```python
import boto3, json

def get_scram_credentials(secret_name):
    client = boto3.client("secretsmanager", region_name="us-east-1")
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

creds = get_scram_credentials("AmazonMSK_orders-producer")

jaas_config = (
    f'org.apache.kafka.common.security.scram.ScramLoginModule required '
    f'username="{creds["username"]}" password="{creds["password"]}";'
)
```

Then pass `jaas_config` into your Kafka client properties at runtime.
This pattern works for Java, Python (kafka-python / confluent-kafka), and Go.

---

## 5. CLI Tools with SCRAM

Create `client.properties`:

```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="svc-orders-producer" \
  password="str0ngP@ssword123!";
```

Produce:

```bash
kafka-console-producer.sh \
  --broker-list b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --topic orders-topic \
  --producer.config client.properties
```

Consume:

```bash
kafka-console-consumer.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --topic orders-topic \
  --group orders-consumer-group \
  --from-beginning \
  --consumer.config client.properties
```

---

## 6. Handling Credential Rotation in the Application

When Secrets Manager rotates the password, the running Kafka client will eventually get auth failures. Handle this:

```
Application gets SASL auth failure
        │
        ▼
Catch authentication exception
        │
        ▼
Re-fetch credentials from Secrets Manager
        │
        ▼
Rebuild Kafka client with new jaas.config
        │
        ▼
Resume producing / consuming
```

- Build a credential refresh wrapper around your Kafka client factory
- Do not restart the entire application on rotation — just rebuild the client

---

## 7. Common Mistakes

| Mistake                                        | Result                                      |
|------------------------------------------------|---------------------------------------------|
| Using port 9092 or 9094 with SCRAM config      | Connection refused or wrong auth mechanism  |
| Secret name missing `AmazonMSK_` prefix        | MSK ignores the secret, auth always fails   |
| Secret encrypted with default AWS key          | MSK cannot decrypt, association fails       |
| Hardcoded password in application config       | Credential leak risk, rotation breaks app   |
| Username in secret doesn't match ACL principal | Auth succeeds but all Kafka ops denied      |

---

## Key Takeaway

SCRAM client config is 3 properties: protocol, mechanism, and jaas.config.
In production, always fetch credentials from Secrets Manager at runtime — never hardcode them.
Build rotation handling into the client from day one, not after the first rotation incident.
