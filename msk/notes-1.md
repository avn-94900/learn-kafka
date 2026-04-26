# MSK Authentication & Authorization — Master Learning Index

## Full Structure

```
msk/
├── notes-1.md                                         ← master index (this file)
│
├── [IAM Auth]
│   ├── folder-1-iam-authorization-design.md
│   ├── folder-2-client-configuration.md
│   └── folder-3-resource-arns-and-policies.md
│
├── mtls/
│   ├── folder-1-mtls-authorization-design.md
│   ├── folder-2-mtls-client-configuration.md
│   └── folder-3-mtls-acls-and-production.md
│
├── sasl-scram/
│   ├── folder-1-scram-authorization-design.md
│   ├── folder-2-scram-client-configuration.md
│   └── folder-3-scram-acls-and-production.md
│
└── unauthenticated/
    ├── folder-1-unauthenticated-design.md
    ├── folder-2-unauthenticated-client-configuration.md
    └── folder-3-unauthenticated-controls-and-limits.md
```

---

## Auth Method Quick Reference

| Method          | Port | Identity          | Authorization  | Best For                    |
|-----------------|------|-------------------|----------------|-----------------------------|
| IAM             | 9098 | AWS IAM Role      | IAM Policy     | AWS-native apps             |
| mTLS            | 9094 | Certificate DN    | Kafka ACL      | PKI orgs, hybrid clients    |
| SASL/SCRAM      | 9096 | Username/password | Kafka ACL      | Non-AWS or hybrid clients   |
| Unauthenticated | 9092 | None (ANONYMOUS)  | Kafka ACL / none | Dev/test only             |

---

## IAM Auth Learning Set

### Folder 1 — IAM Authorization Design
- How IAM replaces Kafka ACLs entirely
- Full kafka-cluster:* actions table
- Producer, Consumer, Admin JSON policies
- Trust policy for EC2 / ECS / Lambda
- Production design rules

### Folder 2 — Client Configuration
- 4-property SASL_SSL + AWS_MSK_IAM config block
- Producer and Consumer .properties
- CLI with client.properties
- STS credential flow
- Common misconfiguration table

### Folder 3 — Resource ARNs and Production Policy Design
- All 4 ARN formats (cluster, topic, group, transactional-id)
- Wildcard scoping rules
- Multi-topic prefix policy, cross-service ECS policy
- IAM Condition keys, Explicit Deny via SCP
- Exactly-once (EOS) transactional producer policy
- Production readiness checklist

---

## mTLS Learning Set

### Folder 1 — Certificate-Based Authorization Design
- How mTLS auth and Kafka ACL authorization work together
- AWS Private CA vs self-signed CA
- Distinguished Name (DN) as Kafka principal
- Registering CA with MSK cluster
- DN naming convention for production

### Folder 2 — Client Configuration
- Keystore and truststore creation (openssl + keytool)
- Java producer and consumer .properties
- CLI with client.properties
- Storing keystores in Secrets Manager
- Common mistakes table

### Folder 3 — Kafka ACLs and Production Design
- Full ACL command reference (producer, consumer, prefix wildcard)
- List and remove ACL commands
- Zero-downtime certificate rotation strategy
- mTLS vs IAM decision guide
- Production checklist

---

## SASL/SCRAM Learning Set

### Folder 1 — Credential-Based Authorization Design
- How SCRAM auth works with Secrets Manager
- Secret naming convention (`AmazonMSK_` prefix)
- Secret JSON format and KMS key requirement
- Username as Kafka principal
- Registering secrets with MSK cluster

### Folder 2 — Client Configuration
- Producer and Consumer .properties
- Runtime credential loading from Secrets Manager (Python example)
- CLI with client.properties
- Handling credential rotation in running applications
- Common mistakes table

### Folder 3 — ACLs, Rotation, and Production Design
- Full ACL command reference
- Secrets Manager automatic rotation setup
- Rotation flow and application impact window
- Manual secret update commands
- SCRAM vs mTLS vs IAM decision table
- KMS key policy for MSK SCRAM
- Production checklist

---

## Unauthenticated Learning Set

### Folder 1 — Design and Risk Assessment
- What unauthenticated means (ANONYMOUS principal)
- When it is and is not acceptable
- Network as the only security boundary
- Enabling/disabling via AWS CLI
- ANONYMOUS ACL usage in dev

### Folder 2 — Client Configuration
- Minimal producer and consumer .properties (no security config)
- CLI commands without auth
- Port reference for all 4 auth methods
- Getting bootstrap broker endpoints per auth method
- Client-side change when migrating to IAM

### Folder 3 — Controls, Limits, and Migration
- Capability comparison table (what unauthenticated cannot do)
- Compensating controls for dev environments
- Running multiple auth methods simultaneously
- Step-by-step migration checklist (unauthenticated → IAM)
- Full 4-method comparison table
- Detecting plaintext connections via CloudWatch

---

## Where to Start

| Goal                                              | Go To                                      |
|---------------------------------------------------|--------------------------------------------|
| Design auth for a new AWS-native application      | IAM → Folder 1                             |
| Configure a Java Kafka client for IAM             | IAM → Folder 2                             |
| Write or review IAM policies for MSK              | IAM → Folder 3                             |
| Set up mTLS with AWS Private CA                   | mTLS → Folder 1                            |
| Configure keystore/truststore for mTLS            | mTLS → Folder 2                            |
| Write Kafka ACLs for mTLS or SCRAM                | mTLS → Folder 3 or SCRAM → Folder 3        |
| Set up SCRAM credentials in Secrets Manager       | SCRAM → Folder 1                           |
| Handle SCRAM credential rotation in application   | SCRAM → Folder 2 and Folder 3              |
| Understand risks of unauthenticated access        | Unauthenticated → Folder 1                 |
| Migrate from plaintext to a real auth method      | Unauthenticated → Folder 3                 |
