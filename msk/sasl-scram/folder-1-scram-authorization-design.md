# SASL/SCRAM — Folder 1: Credential-Based Authorization Design

## Learning Path
msk-sasl-scram-learning/
├── folder-1-scram-authorization-design.md   ← YOU ARE HERE
├── folder-2-scram-client-configuration.md
└── folder-3-scram-acls-and-production.md

---

## 1. What SASL/SCRAM Means in MSK

SASL = Simple Authentication and Security Layer
SCRAM = Salted Challenge Response Authentication Mechanism

The client authenticates with a username and password.
MSK stores credentials in AWS Secrets Manager — not inside Kafka.

```
Application (holds username + password)
        │
        │  SASL_SSL + SCRAM-SHA-512 handshake
        ▼
MSK Broker (fetches credential hash from Secrets Manager)
        │
        │  validates SCRAM challenge/response
        ▼
Kafka ACL engine evaluates username against topic/group ACLs
        │
        ▼
Allow / Deny
```

- Authentication = broker validates the SCRAM credential from Secrets Manager
- Authorization = Kafka ACLs match the username
- MSK supports SCRAM-SHA-512 only (not SHA-256)

---

## 2. How MSK Stores SCRAM Credentials

Credentials live in AWS Secrets Manager with a specific naming convention.

Secret name format:
```
AmazonMSK_<any-suffix>
```

The prefix `AmazonMSK_` is mandatory — MSK will not recognize secrets without it.

Secret value format (JSON):

```json
{
  "username": "orders-producer",
  "password": "str0ngP@ssword123!"
}
```

The secret must be encrypted with a customer-managed KMS key (not the default AWS-managed key).

---

## 3. Registering the Secret with the MSK Cluster

After creating the secret, associate it with the cluster:

```bash
aws kafka batch-associate-scram-secret \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid \
  --secret-arn-list arn:aws:secretsmanager:us-east-1:123456789012:secret:AmazonMSK_orders-producer-abc123
```

To list currently associated secrets:

```bash
aws kafka list-scram-secrets \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid
```

---

## 4. Username as Kafka Principal

The `username` field in the secret becomes the Kafka principal:

```
User:orders-producer
User:orders-consumer
User:kafka-admin
```

Kafka ACLs are written against these principal strings exactly.

### Username Design Rules

| Application Role  | Username                  |
|-------------------|---------------------------|
| Producer service  | `svc-orders-producer`     |
| Consumer service  | `svc-orders-consumer`     |
| Admin/ops         | `kafka-admin`             |

- One username per application service
- Never share credentials across services
- Username cannot be changed without deleting and recreating the secret + ACLs

---

## 5. Authorization Model — SCRAM Uses Kafka ACLs

Same as mTLS — SCRAM authorization is handled by Kafka ACLs, not IAM.

```bash
# Producer ACL
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:svc-orders-producer" \
  --operation Write \
  --operation Describe \
  --topic orders-topic

# Consumer ACL
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9096 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:svc-orders-consumer" \
  --operation Read \
  --operation Describe \
  --topic orders-topic \
  --group orders-consumer-group
```

> Port 9096 is the SASL/SCRAM listener in MSK.

---

## 6. KMS Key Requirement

MSK requires SCRAM secrets to use a customer-managed KMS key.

```bash
# Create KMS key
aws kms create-key --description "MSK SCRAM credentials key"

# Create secret with that key
aws secretsmanager create-secret \
  --name AmazonMSK_orders-producer \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/key-uuid \
  --secret-string '{"username":"svc-orders-producer","password":"str0ngP@ssword123!"}'
```

---

## 7. Production Design Rules

- One secret per application service — never reuse credentials
- Use strong passwords (20+ chars, mixed case, symbols)
- Enable automatic rotation in Secrets Manager (30–90 day cycle)
- The application must handle credential refresh on rotation (re-read from Secrets Manager)
- KMS key policy must allow MSK service principal to decrypt
- Never hardcode credentials in application config files or environment variables

---

## Key Takeaway

In SASL/SCRAM, the username in Secrets Manager is the Kafka identity.
Kafka ACLs control what that username can do.
The secret naming prefix `AmazonMSK_` is not optional — MSK silently ignores secrets without it.
