# mTLS — Folder 1: Certificate-Based Authorization Design

## Learning Path
msk-mtls-learning/
├── folder-1-mtls-authorization-design.md   ← YOU ARE HERE
├── folder-2-mtls-client-configuration.md
└── folder-3-mtls-acls-and-production.md

---

## 1. What mTLS Means in MSK Context

mTLS = Mutual TLS. Both the broker and the client present certificates.
The client certificate IS the identity — no passwords, no tokens.

```
Client (holds cert + private key)
        │
        │  presents client certificate during TLS handshake
        ▼
MSK Broker (holds server cert, trusts your CA)
        │
        │  extracts Distinguished Name (DN) from client cert
        ▼
Kafka ACL engine evaluates DN against topic/group ACLs
        │
        ▼
Allow / Deny
```

- Authentication = broker validates the cert was signed by a trusted CA
- Authorization = Kafka ACLs match the cert's Distinguished Name (DN)
- IAM is NOT involved in mTLS auth — these are two separate mechanisms

---

## 2. Certificate Authority Options in MSK

MSK does not issue certificates. You bring your own CA.

| CA Option                        | Use Case                                      |
|----------------------------------|-----------------------------------------------|
| AWS Private CA (ACM PCA)         | Production — managed, auditable, integrated   |
| Self-signed CA (openssl)         | Dev/test only — no revocation support         |
| Enterprise CA (e.g. HashiCorp Vault PKI) | Hybrid/on-prem environments           |

AWS Private CA is the recommended production choice because:
- Integrates with ACM for cert issuance
- Supports certificate revocation (CRL/OCSP)
- CloudTrail logs every cert issuance

---

## 3. Distinguished Name (DN) — The Identity in mTLS

The DN from the client certificate becomes the Kafka principal.

Example DN:
```
CN=orders-producer,OU=payments,O=mycompany,C=US
```

This maps to the Kafka principal:
```
User:CN=orders-producer,OU=payments,O=mycompany,C=US
```

Kafka ACLs are written against this exact string.

### DN Design Rules for Production

| Application Role  | Example CN Value         |
|-------------------|--------------------------|
| Producer service  | `CN=svc-orders-producer` |
| Consumer service  | `CN=svc-orders-consumer` |
| Admin/ops tool    | `CN=kafka-admin`         |

- Use CN as the primary differentiator
- Keep OU/O consistent across all certs for the same cluster
- Never reuse a CN across different applications

---

## 4. How MSK Trusts Your CA

When creating or updating an MSK cluster, you register your CA ARN:

```bash
aws kafka update-security \
  --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid \
  --client-authentication '{
    "Tls": {
      "CertificateAuthorityArnList": [
        "arn:aws:acm-pca:us-east-1:123456789012:certificate-authority/ca-uuid"
      ],
      "Enabled": true
    }
  }'
```

MSK will trust any client certificate signed by this CA.
Multiple CA ARNs can be registered for migration scenarios.

---

## 5. Authorization Model — mTLS Uses Kafka ACLs

Unlike IAM (which uses IAM policies), mTLS uses native Kafka ACLs.

| IAM Auth          | mTLS Auth              |
|-------------------|------------------------|
| IAM policy        | Kafka ACL              |
| kafka-cluster:*   | kafka-acls.sh          |
| ARN-based resource| Topic/group name-based |
| AWS Console/CLI   | Kafka CLI only         |

ACL is set using the Kafka ACL tool against the DN principal:

```bash
# Allow producer
kafka-acls.sh --bootstrap-server <broker>:9094 \
  --command-config client.properties \
  --add \
  --allow-principal "User:CN=svc-orders-producer" \
  --operation Write \
  --operation Describe \
  --topic orders-topic

# Allow consumer
kafka-acls.sh --bootstrap-server <broker>:9094 \
  --command-config client.properties \
  --add \
  --allow-principal "User:CN=svc-orders-consumer" \
  --operation Read \
  --operation Describe \
  --topic orders-topic \
  --group orders-consumer-group
```

---

## 6. Production Design Rules

- One certificate per application service, not per instance
- CN naming must be consistent and documented
- Certificates have expiry — build rotation into your deployment pipeline
- Use AWS Private CA with short-lived certs (90 days) and automate renewal
- Never embed private keys in container images or source code
- Store private keys in AWS Secrets Manager or use ACM with ECS/EC2

---

## Key Takeaway

In mTLS, the certificate DN is the identity and Kafka ACLs are the authorization layer.
Design your DN naming scheme before issuing any certificates — it cannot be changed without reissuing certs and updating all ACLs.
