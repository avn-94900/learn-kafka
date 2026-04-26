# Unauthenticated (Plaintext) — Folder 1: Design and Risk Assessment

## Learning Path
msk-unauthenticated-learning/
├── folder-1-unauthenticated-design.md   ← YOU ARE HERE
├── folder-2-unauthenticated-client-configuration.md
└── folder-3-unauthenticated-controls-and-limits.md

---

## 1. What Unauthenticated Access Means in MSK

No credentials, no certificates, no tokens.
Any client that can reach the broker on port 9092 can connect and operate.

```
Application (no credentials)
        │
        │  plaintext TCP connection
        ▼
MSK Broker (port 9092 — no auth check)
        │
        │  no identity established
        ▼
Kafka ACL engine (if enabled) — principal = ANONYMOUS
        │
        ▼
Allow / Deny based on ANONYMOUS ACLs
```

- There is no authentication step — the broker accepts all connections
- If Kafka ACLs are disabled (default), all operations are permitted to everyone
- If Kafka ACLs are enabled, the principal is always `User:ANONYMOUS`

---

## 2. When Unauthenticated Access Is Acceptable

This is a short list — unauthenticated should be the exception, not the default.

| Scenario                                      | Acceptable? |
|-----------------------------------------------|-------------|
| Local development / laptop testing            | ✓           |
| Isolated internal dev cluster (no prod data)  | ✓           |
| Performance benchmarking (controlled env)     | ✓           |
| Production cluster with real data             | ✗ Never     |
| Any cluster reachable from the internet       | ✗ Never     |
| Staging environment mirroring production      | ✗ No        |

---

## 3. Network Is the Only Security Boundary

When using unauthenticated access, the VPC and security groups are the entire security model.

```
Internet
    │
    │  blocked by Security Group (no inbound 9092 from 0.0.0.0/0)
    ▼
VPC boundary
    │
    │  only trusted subnets / security groups allowed on port 9092
    ▼
MSK Broker (port 9092)
    │
    │  no auth — anyone inside the VPC can connect
    ▼
Kafka (all operations permitted if ACLs disabled)
```

Security group rule for unauthenticated MSK (dev cluster example):

```
Inbound rule:
  Type: Custom TCP
  Port: 9092
  Source: sg-0abc12345 (application security group only)
```

Never allow `0.0.0.0/0` on port 9092.

---

## 4. Enabling Unauthenticated Access on MSK

Unauthenticated is enabled by default when creating an MSK cluster unless you explicitly disable it.

To check current auth configuration:

```bash
aws kafka describe-cluster \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid \
  --query "ClusterInfo.ClientAuthentication"
```

Output when unauthenticated is the only method:

```json
{
  "Unauthenticated": {
    "Enabled": true
  }
}
```

---

## 5. Unauthenticated + Kafka ACLs (ANONYMOUS Principal)

If you want some access control without full auth, you can enable Kafka ACLs and grant permissions to `ANONYMOUS`.

```bash
# Allow ANONYMOUS to write to a specific topic
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092 \
  --add \
  --allow-principal "User:ANONYMOUS" \
  --operation Write \
  --operation Describe \
  --topic dev-test-topic
```

This is still not real authorization — any client in the VPC gets the same access.
Use this only to practice ACL syntax in a dev environment.

---

## 6. Disabling Unauthenticated Access (Migration Path)

When moving from unauthenticated to a real auth method:

```bash
aws kafka update-security \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid \
  --client-authentication '{
    "Unauthenticated": { "Enabled": false },
    "Sasl": {
      "Iam": { "Enabled": true }
    }
  }'
```

- MSK supports enabling multiple auth methods simultaneously during migration
- Enable the new method first, migrate clients, then disable unauthenticated
- Disabling unauthenticated while clients still use port 9092 will break those clients immediately

---

## Key Takeaway

Unauthenticated access has no identity layer — the network IS the security.
It is only acceptable in isolated dev environments with strict security group controls.
The migration path is: enable new auth method → migrate clients → disable unauthenticated. Never the reverse.
