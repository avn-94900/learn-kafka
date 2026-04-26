# Folder 3 — MSK Resource ARNs and Production Policy Design

## Learning Path
msk-iam-learning/
├── folder-1-iam-authorization-design/
├── folder-2-client-configuration/
└── folder-3-resource-arns-and-policies/  ← YOU ARE HERE

---

## 1. MSK Resource ARN Formats

Every IAM policy statement targeting MSK must use one of these ARN formats.

### Cluster ARN
```
arn:aws:kafka:<region>:<account-id>:cluster/<cluster-name>/<cluster-uuid>
```
Example:
```
arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/abc123-def4-5678-ghij-klmnopqrstuv-1
```

### Topic ARN
```
arn:aws:kafka:<region>:<account-id>:topic/<cluster-name>/<cluster-uuid>/<topic-name>
```
Example:
```
arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/abc123-def4-5678-ghij-klmnopqrstuv-1/orders-topic
```

### Consumer Group ARN
```
arn:aws:kafka:<region>:<account-id>:group/<cluster-name>/<cluster-uuid>/<group-name>
```
Example:
```
arn:aws:kafka:us-east-1:123456789012:group/my-cluster/abc123-def4-5678-ghij-klmnopqrstuv-1/orders-consumer-group
```

### Transactional ID ARN
```
arn:aws:kafka:<region>:<account-id>:transactional-id/<cluster-name>/<cluster-uuid>/<transactional-id>
```
Used only when exactly-once semantics (EOS) producers are required.

---

## 2. Wildcard Usage in ARNs

| Pattern                          | Meaning                                      |
|----------------------------------|----------------------------------------------|
| `cluster/my-cluster/*`           | Any UUID for this named cluster              |
| `topic/my-cluster/*/orders-*`    | All topics starting with "orders-"           |
| `topic/my-cluster/*/*`           | All topics in the cluster (use with caution) |
| `group/my-cluster/*/svc-orders-*`| All groups with prefix "svc-orders-"         |

> Wildcard on cluster UUID (`*`) is safe and standard — the UUID is not predictable.
> Wildcard on topic name should be intentional and scoped by prefix, not `*` alone.

---

## 3. How to Find Your Cluster UUID

```bash
aws kafka list-clusters --region us-east-1 \
  --query "ClusterInfoList[?ClusterName=='my-cluster'].ClusterArn" \
  --output text
```

The UUID is the last segment of the returned ARN after the cluster name.

---

## 4. Multi-Topic Producer Policy (Prefix Pattern)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "kafka-cluster:Connect",
      "Resource": "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:WriteData"
      ],
      "Resource": "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/orders-*"
    }
  ]
}
```

This allows the producer to write to any topic prefixed with `orders-` without updating the policy per new topic.

---

## 5. Cross-Service Consumer Policy (ECS Task Role)

Scenario: ECS task consumes from MSK and writes results to S3.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "kafka-cluster:Connect",
      "Resource": "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/*"
    },
    {
      "Effect": "Allow",
      "Action": ["kafka-cluster:DescribeTopic", "kafka-cluster:ReadData"],
      "Resource": "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/orders-topic"
    },
    {
      "Effect": "Allow",
      "Action": ["kafka-cluster:DescribeGroup", "kafka-cluster:AlterGroup"],
      "Resource": "arn:aws:kafka:us-east-1:123456789012:group/my-cluster/*/ecs-orders-processor"
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-output-bucket/orders/*"
    }
  ]
}
```

---

## 6. IAM Condition Keys for MSK

Add conditions to restrict access further without changing the resource ARN.

### Restrict to VPC only

```json
{
  "Effect": "Allow",
  "Action": "kafka-cluster:Connect",
  "Resource": "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/*",
  "Condition": {
    "StringEquals": {
      "aws:SourceVpc": "vpc-0abc12345def67890"
    }
  }
}
```

### Restrict to specific AWS account (cross-account guard)

```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalAccount": "123456789012"
  }
}
```

---

## 7. Deny Pattern — Explicit Topic Protection

Prevent any role from deleting a critical topic, even admins:

```json
{
  "Effect": "Deny",
  "Action": "kafka-cluster:DeleteTopic",
  "Resource": "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/payments-topic"
}
```

Attach this as a Service Control Policy (SCP) at the OU level for org-wide enforcement.

---

## 8. Transactional Producer Policy (Exactly-Once)

```json
{
  "Effect": "Allow",
  "Action": [
    "kafka-cluster:Connect",
    "kafka-cluster:WriteData",
    "kafka-cluster:WriteDataIdempotently",
    "kafka-cluster:DescribeTopic"
  ],
  "Resource": [
    "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/*",
    "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/payments-topic",
    "arn:aws:kafka:us-east-1:123456789012:transactional-id/my-cluster/*/payments-txn-*"
  ]
}
```

---

## 9. Production Checklist

- [ ] Each application has its own IAM role — no shared roles across services
- [ ] kafka-cluster:Connect scoped to cluster ARN, not `*`
- [ ] Topic ARNs use name or prefix, never bare wildcard
- [ ] Consumer group ARN matches the exact group.id used in client config
- [ ] Transactional ID ARN added only for EOS producers
- [ ] DeleteTopic and DeleteGroup denied via SCP for production clusters
- [ ] CloudTrail enabled — all kafka-cluster:* actions are logged
- [ ] No IAM user credentials used — only roles with instance/task metadata

---

## Key Takeaway

The ARN structure is the enforcement boundary.
Scoping topic and group ARNs precisely is what makes IAM authorization production-safe.
Wildcards on topic prefix are acceptable; wildcards on topic name alone are not.
