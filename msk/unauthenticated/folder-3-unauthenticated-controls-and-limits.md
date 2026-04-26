# Unauthenticated (Plaintext) — Folder 3: Controls, Limits, and Migration

## Learning Path
msk-unauthenticated-learning/
├── folder-1-unauthenticated-design.md
├── folder-2-unauthenticated-client-configuration.md
└── folder-3-unauthenticated-controls-and-limits.md   ← YOU ARE HERE

---

## 1. What You Cannot Do with Unauthenticated Access

| Capability                              | Unauthenticated | IAM | mTLS | SCRAM |
|-----------------------------------------|-----------------|-----|------|-------|
| Per-application access control         | ✗               | ✓   | ✓    | ✓     |
| Audit who produced/consumed what        | ✗               | ✓   | ✗    | ✗     |
| Revoke access for one app without VPC change | ✗          | ✓   | ✓    | ✓     |
| Topic-level read/write isolation        | ✗ (ACL only)    | ✓   | ✓    | ✓     |
| Credential rotation                     | N/A             | Auto| Manual| Manual|
| Compliance (PCI, HIPAA, SOC2)           | ✗               | ✓   | ✓    | ✓     |

---

## 2. Compensating Controls When Using Unauthenticated (Dev Only)

If you must use unauthenticated in a dev environment, apply all of these:

### Security Group — Restrict to Application SG Only

```
Inbound rule on MSK security group:
  Port: 9092
  Source: sg-<application-security-group-id>   ← NOT 0.0.0.0/0
```

### VPC Endpoint — No Internet Gateway Route to MSK Subnets

```bash
# Verify MSK subnets have no route to IGW
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-<msk-subnet-id>" \
  --query "RouteTables[].Routes"
```

No route with `GatewayId` starting with `igw-` should exist for MSK subnets.

### Kafka ACLs — Enable and Restrict ANONYMOUS

```bash
# Enable ACLs on the cluster (requires broker config change via MSK config)
# Then deny ANONYMOUS on sensitive topics
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092 \
  --add \
  --deny-principal "User:ANONYMOUS" \
  --operation All \
  --topic sensitive-topic
```

---

## 3. Enabling Multiple Auth Methods Simultaneously

MSK allows running multiple auth methods at the same time.
This is the correct migration pattern — never cut over cold.

```bash
aws kafka update-security \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid \
  --client-authentication '{
    "Unauthenticated": { "Enabled": true },
    "Sasl": {
      "Iam": { "Enabled": true }
    }
  }'
```

During this window:
- Old clients use port 9092 (unauthenticated) — still working
- New clients use port 9098 (IAM) — already migrated
- Both work simultaneously until all clients are migrated

---

## 4. Migration Checklist — Unauthenticated to IAM

- [ ] Enable IAM auth on the cluster (keep unauthenticated enabled)
- [ ] Attach IAM roles to all application compute (EC2/ECS/Lambda)
- [ ] Write IAM policies for each application (producer/consumer roles)
- [ ] Update client config: change port 9092 → 9098, add 4 IAM properties
- [ ] Test each application with IAM auth in staging
- [ ] Deploy updated clients to production one service at a time
- [ ] Verify all clients are using port 9098 (check CloudWatch broker metrics)
- [ ] Disable unauthenticated access on the cluster
- [ ] Verify port 9092 connections are rejected (no active clients)
- [ ] Update security group to remove port 9092 inbound rule

---

## 5. Full MSK Auth Method Comparison

| Factor                     | Unauthenticated | mTLS              | SASL/SCRAM         | IAM                  |
|----------------------------|-----------------|-------------------|--------------------|----------------------|
| Port                       | 9092            | 9094              | 9096               | 9098                 |
| Identity type              | None (ANONYMOUS)| Certificate DN    | Username           | AWS IAM Role         |
| Authorization engine       | Kafka ACL       | Kafka ACL         | Kafka ACL          | IAM Policy           |
| Credential management      | None            | Cert + keystore   | Secrets Manager    | Instance metadata    |
| Rotation                   | N/A             | Manual (cert)     | Secrets Manager    | Automatic (STS)      |
| Works outside AWS          | ✓               | ✓                 | ✓                  | Limited              |
| CloudTrail audit           | ✗               | ✗                 | Partial            | ✓ Full               |
| Production ready           | ✗               | ✓                 | ✓                  | ✓                    |
| Compliance workloads       | ✗               | ✓                 | ✓                  | ✓ (preferred)        |
| Setup complexity           | None            | High              | Medium             | Low (AWS-native)     |

---

## 6. Detecting Unauthenticated Connections in Production

If unauthenticated is accidentally left enabled, detect active plaintext connections:

```bash
# Check if any clients are connecting on port 9092
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kafka \
  --metric-name ConnectionCount \
  --dimensions Name=Cluster,Value=my-cluster Name=BrokerID,Value=1 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Sum
```

Also check MSK broker logs in CloudWatch Logs for `PLAINTEXT` connection entries.

---

## Key Takeaway

Unauthenticated access is a starting point, not a destination.
Every production MSK cluster must use at least one real auth method.
The migration from unauthenticated is low-risk when done incrementally — enable new auth, migrate clients, then disable plaintext.
