# Folder 1 — IAM Authorization Design for MSK

## Learning Path
msk-iam-learning/
├── folder-1-iam-authorization-design/   ← YOU ARE HERE
├── folder-2-client-configuration/
└── folder-3-resource-arns-and-policies/

---

## 1. Why IAM Authorization in MSK?

MSK supports multiple auth mechanisms: mTLS, SASL/SCRAM, and IAM.
IAM is the preferred choice in AWS-native production environments because:

- No certificate rotation overhead (unlike mTLS)
- No password management (unlike SCRAM)
- Integrates directly with AWS roles, SCPs, and CloudTrail
- Supports fine-grained resource-level permissions per topic and group

---

## 2. How IAM Auth Works in MSK (Flow)

```
Application (EC2 / ECS / Lambda)
        │
        │  assumes IAM Role
        ▼
   IAM Role (attached policy)
        │
        │  SASL_SSL + AWS_MSK_IAM mechanism
        ▼
   MSK Broker (IAM-enabled listener port 9098)
        │
        │  evaluates kafka-cluster:* actions
        ▼
   Allow / Deny per topic / group / cluster
```

- The MSK broker validates the IAM identity on every connection
- The IAM policy controls what Kafka operations are permitted
- No Kafka ACLs are used when IAM auth is active

---

## 3. IAM Policy Actions — Full Reference

| Action                              | What It Controls                        |
|-------------------------------------|-----------------------------------------|
| kafka-cluster:Connect               | Ability to connect to the cluster       |
| kafka-cluster:DescribeCluster       | Describe cluster metadata               |
| kafka-cluster:CreateTopic           | Create a new topic                      |
| kafka-cluster:DescribeTopic         | Read topic metadata                     |
| kafka-cluster:AlterTopic            | Modify topic config (retention, etc.)   |
| kafka-cluster:DeleteTopic           | Delete a topic                          |
| kafka-cluster:WriteData             | Produce messages to a topic             |
| kafka-cluster:ReadData              | Consume messages from a topic           |
| kafka-cluster:DescribeGroup         | Describe consumer group                 |
| kafka-cluster:AlterGroup            | Commit offsets, join group              |
| kafka-cluster:DeleteGroup           | Delete a consumer group                 |
| kafka-cluster:WriteDataIdempotently | Idempotent producer (exactly-once)      |

> kafka-cluster:Connect is ALWAYS required — without it, no other action works.

---

## 4. Least Privilege Role Design

### Producer Role Policy

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
      "Resource": "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/orders-topic"
    }
  ]
}
```

### Consumer Role Policy

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
        "kafka-cluster:ReadData"
      ],
      "Resource": "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/orders-topic"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:DescribeGroup",
        "kafka-cluster:AlterGroup"
      ],
      "Resource": "arn:aws:kafka:us-east-1:123456789012:group/my-cluster/*/orders-consumer-group"
    }
  ]
}
```

### Admin Role Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "kafka-cluster:*",
      "Resource": [
        "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/*",
        "arn:aws:kafka:us-east-1:123456789012:topic/my-cluster/*/*",
        "arn:aws:kafka:us-east-1:123456789012:group/my-cluster/*/*"
      ]
    }
  ]
}
```

> Admin role is for ops/automation only — never attach to application workloads.

---

## 5. Trust Policy — Who Can Assume the Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

For ECS tasks, replace `ec2.amazonaws.com` with `ecs-tasks.amazonaws.com`.
For Lambda, use `lambda.amazonaws.com`.

---

## 6. Production Design Rules

- One IAM role per application, not per team
- Never use wildcard `*` on topic resource for producers/consumers
- Always scope consumer group ARN to the specific group name
- Use IAM Conditions to restrict by VPC or source IP if needed
- Rotate nothing — IAM credentials are temporary (STS tokens, ~1 hour TTL)

---

## Key Takeaway

IAM authorization in MSK replaces Kafka ACLs entirely.
The IAM policy IS the Kafka access control layer.
Design roles around Kafka operations (produce/consume/admin), not around teams.
