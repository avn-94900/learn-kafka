# Folder 2 — Client Configuration with IAM Auth

## Learning Path
msk-iam-learning/
├── folder-1-iam-authorization-design/
├── folder-2-client-configuration/       ← YOU ARE HERE
└── folder-3-resource-arns-and-policies/

---

## 1. Required Dependency

Add the AWS MSK IAM Auth library to your project.

```xml
<!-- Maven -->
<dependency>
  <groupId>software.amazon.msk</groupId>
  <artifactId>aws-msk-iam-auth</artifactId>
  <version>2.1.1</version>
</dependency>
```

This library handles STS token generation and SASL handshake automatically.
No manual credential passing is needed.

---

## 2. Core Client Properties (Shared by Producer and Consumer)

```properties
# Broker endpoint — use IAM port 9098
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9098

# Security
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM

# IAM Auth plugin
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

> Port 9098 is the IAM-enabled listener. Port 9092 is plaintext, 9094 is mTLS.
> The IAM role is picked up automatically from the EC2/ECS/Lambda instance metadata.

---

## 3. Producer Configuration

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9098
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler

# Producer tuning
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
retries=3
enable.idempotence=true
```

> enable.idempotence=true requires kafka-cluster:WriteDataIdempotently in the IAM policy.

---

## 4. Consumer Configuration

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9098
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler

# Consumer identity
group.id=orders-consumer-group
auto.offset.reset=earliest

key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

> The group.id value must match the consumer group ARN scoped in the IAM policy.
> If the group name doesn't match the policy resource, AlterGroup will be denied.

---

## 5. CLI Tools with IAM Auth

Create a client.properties file:

```properties
# client.properties
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

Produce via CLI:

```bash
kafka-console-producer.sh \
  --broker-list b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9098 \
  --topic orders-topic \
  --producer.config client.properties
```

Consume via CLI:

```bash
kafka-console-consumer.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9098 \
  --topic orders-topic \
  --group orders-consumer-group \
  --from-beginning \
  --consumer.config client.properties
```

---

## 6. IAM Role Assumption — How the Client Gets Credentials

```
Application starts
      │
      ▼
IAMClientCallbackHandler calls EC2 Instance Metadata Service (IMDS)
      │
      ▼
IMDS returns temporary STS credentials for the attached IAM role
      │
      ▼
Credentials used to sign the SASL handshake with MSK broker
      │
      ▼
MSK broker validates signature against IAM → Allow or Deny
```

- Credentials auto-refresh before expiry (~1 hour STS token TTL)
- No code change needed for credential rotation
- Works identically on EC2, ECS (task role), Lambda, and EKS (IRSA)

---

## 7. Common Configuration Mistakes

| Mistake                                  | Result                              |
|------------------------------------------|-------------------------------------|
| Using port 9092 with IAM config          | Connection refused or auth failure  |
| group.id not matching IAM policy ARN     | AlterGroup denied, consumer fails   |
| Missing kafka-cluster:Connect            | All operations fail immediately     |
| enable.idempotence without policy action | Producer initialization fails       |
| Wrong region in bootstrap server         | STS token region mismatch           |

---

## Key Takeaway

The client config is minimal — 4 properties handle all IAM auth.
The IAM role attached to the compute resource IS the identity.
Match group.id exactly to what is scoped in the consumer IAM policy.
