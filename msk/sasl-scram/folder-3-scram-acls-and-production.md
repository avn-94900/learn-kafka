# SASL/SCRAM — Folder 3: ACLs, Rotation, and Production Design

## Learning Path
msk-sasl-scram-learning/
├── folder-1-scram-authorization-design.md
├── folder-2-scram-client-configuration.md
└── folder-3-scram-acls-and-production.md   ← YOU ARE HERE

---

## 1. Full ACL Reference for SCRAM

Admin client.properties for running ACL commands:

```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="kafka-admin" \
  password="adminP@ssword!";
```

### Producer ACL

```bash
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:svc-orders-producer" \
  --operation Write \
  --operation Describe \
  --topic orders-topic
```

### Consumer ACL (topic + group)

```bash
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:svc-orders-consumer" \
  --operation Read \
  --operation Describe \
  --topic orders-topic

kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:svc-orders-consumer" \
  --operation Read \
  --group orders-consumer-group
```

### Topic prefix wildcard ACL

```bash
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:svc-orders-producer" \
  --operation Write \
  --operation Describe \
  --topic orders- \
  --resource-pattern-type prefixed
```

---

## 2. Secrets Manager Rotation Strategy

### Automatic Rotation Setup

```bash
aws secretsmanager rotate-secret \
  --secret-id AmazonMSK_orders-producer \
  --rotation-rules AutomaticallyAfterDays=30
```

MSK does NOT automatically pick up the new password after rotation.
The application must re-read the secret and rebuild the Kafka client.

### Rotation Flow

```
Day 0  — Secret created, associated with MSK cluster
Day 30 — Secrets Manager rotates password automatically
         │
         ▼
         New password written to secret (new version)
         │
         ▼
         MSK broker fetches updated credential hash on next connection
         │
         ▼
         Running application gets SASL auth failure on next reconnect
         │
         ▼
         Application catches exception → re-fetches secret → rebuilds client
```

- MSK broker syncs the new credential within ~5 minutes of secret update
- Design your application to tolerate a brief auth failure window during rotation

---

## 3. Updating a Secret Password Manually

```bash
aws secretsmanager update-secret \
  --secret-id AmazonMSK_orders-producer \
  --secret-string '{"username":"svc-orders-producer","password":"newStr0ngP@ss!"}'
```

After updating, re-associate if needed (usually not required for updates):

```bash
aws kafka batch-associate-scram-secret \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid \
  --secret-arn-list arn:aws:secretsmanager:us-east-1:123456789012:secret:AmazonMSK_orders-producer-abc123
```

---

## 4. SCRAM vs mTLS vs IAM — Decision Guide

| Factor                            | SCRAM          | mTLS           | IAM            |
|-----------------------------------|----------------|----------------|----------------|
| Identity type                     | Username/pass  | Certificate DN | AWS Role       |
| Credential storage                | Secrets Manager| Keystore file  | Instance metadata |
| Rotation complexity               | Medium         | High           | None (auto)    |
| Works outside AWS                 | ✓              | ✓              | Limited        |
| Kafka ACLs required               | ✓              | ✓              | ✗ (IAM policy) |
| AWS-native audit (CloudTrail)     | Partial        | ✗              | ✓ Full         |
| Best for                          | Hybrid/on-prem | PKI-heavy orgs | AWS-only apps  |

---

## 5. KMS Key Policy for MSK SCRAM

The KMS key used to encrypt SCRAM secrets must allow MSK to decrypt:

```json
{
  "Sid": "Allow MSK to use the key",
  "Effect": "Allow",
  "Principal": {
    "Service": "kafka.amazonaws.com"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

Add this statement to your KMS key policy alongside the key admin and user statements.

---

## 6. Production Checklist

- [ ] Secret name starts with `AmazonMSK_` — verified
- [ ] Secret encrypted with customer-managed KMS key
- [ ] KMS key policy includes `kafka.amazonaws.com` principal
- [ ] Secret associated with MSK cluster via `batch-associate-scram-secret`
- [ ] One secret per application service — no shared credentials
- [ ] Automatic rotation enabled (30–90 days)
- [ ] Application handles SASL auth failure with credential refresh logic
- [ ] Kafka ACLs set per topic and per consumer group
- [ ] Admin credentials stored separately from application credentials
- [ ] No passwords in environment variables, config files, or container images

---

## Key Takeaway

SCRAM is the right choice when clients exist outside AWS or when PKI infrastructure is not available.
The `AmazonMSK_` prefix and the customer-managed KMS key are non-negotiable requirements — both must be correct before the first connection attempt.
