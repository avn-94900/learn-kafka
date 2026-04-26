# mTLS — Folder 3: Kafka ACLs and Production Design

## Learning Path
msk-mtls-learning/
├── folder-1-mtls-authorization-design.md
├── folder-2-mtls-client-configuration.md
└── folder-3-mtls-acls-and-production.md   ← YOU ARE HERE

---

## 1. Kafka ACL Command Reference

All ACL management uses `kafka-acls.sh` against the mTLS broker endpoint.
The admin running these commands also needs a valid client certificate with ACL admin rights.

### Admin client.properties (for running ACL commands)

```properties
security.protocol=SSL
ssl.keystore.location=/app/certs/admin-keystore.jks
ssl.keystore.password=adminpassword
ssl.key.password=adminpassword
ssl.truststore.location=/app/certs/client-truststore.jks
ssl.truststore.password=truststorepassword
```

---

## 2. ACL Patterns

### Producer ACL

```bash
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:CN=svc-orders-producer" \
  --operation Write \
  --operation Describe \
  --topic orders-topic
```

### Consumer ACL (topic + group)

```bash
# Topic read permission
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:CN=svc-orders-consumer" \
  --operation Read \
  --operation Describe \
  --topic orders-topic

# Consumer group permission
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:CN=svc-orders-consumer" \
  --operation Read \
  --group orders-consumer-group
```

### Topic prefix wildcard ACL

```bash
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --command-config admin-client.properties \
  --add \
  --allow-principal "User:CN=svc-orders-producer" \
  --operation Write \
  --operation Describe \
  --topic orders- \
  --resource-pattern-type prefixed
```

---

## 3. List and Remove ACLs

```bash
# List all ACLs for a topic
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --command-config admin-client.properties \
  --list \
  --topic orders-topic

# Remove a specific ACL
kafka-acls.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --command-config admin-client.properties \
  --remove \
  --allow-principal "User:CN=svc-orders-producer" \
  --operation Write \
  --topic orders-topic
```

---

## 4. Certificate Rotation Strategy

Certificates expire. Rotation must be zero-downtime.

```
Step 1 — Issue new certificate with same CN from CA
Step 2 — Deploy new keystore alongside old one (dual-cert window)
Step 3 — Update application to load new keystore
Step 4 — Verify connections succeed with new cert
Step 5 — Revoke old certificate in AWS Private CA
Step 6 — Remove old keystore from deployment
```

- ACLs do NOT need to change if CN stays the same
- Only the keystore file changes during rotation
- AWS Private CA CRL is checked by MSK broker on each new connection

---

## 5. ACL vs IAM — When to Choose mTLS

| Criteria                          | Choose mTLS + ACL         | Choose IAM               |
|-----------------------------------|---------------------------|--------------------------|
| AWS-native workloads only         |                           | ✓                        |
| On-prem or hybrid clients         | ✓                         |                          |
| Non-Java clients (Go, Python, C)  | ✓                         | ✓ (with SDK support)     |
| Fine-grained topic-level control  | ✓                         | ✓                        |
| No cert management overhead       |                           | ✓                        |
| Existing PKI infrastructure       | ✓                         |                          |

---

## 6. Production Checklist

- [ ] AWS Private CA created and ARN registered in MSK cluster
- [ ] Unique CN per application service — documented in a cert registry
- [ ] Keystores stored in Secrets Manager, not on disk or in images
- [ ] ACLs set per topic and per consumer group — no wildcard `*` on topic
- [ ] Certificate expiry alerts set in CloudWatch (ACM PCA expiry metric)
- [ ] Rotation runbook documented and tested before first expiry
- [ ] Admin cert separate from application certs
- [ ] CRL endpoint accessible from MSK broker VPC (Private CA CRL in S3)

---

## Key Takeaway

mTLS authorization is entirely Kafka ACL-driven.
The CN in the certificate is the only identity the ACL engine sees.
Get the CN naming convention right before issuing certs — changing it means reissuing certs AND rewriting all ACLs.
