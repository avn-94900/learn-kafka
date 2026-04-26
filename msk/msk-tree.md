

```
IAM Authentication & Authorization in MSK
│
├── 1. Foundations (Must Know First)
│   ├── Kafka Basics
│   │   ├── Producers
│   │   ├── Consumers
│   │   ├── Topics
│   │   ├── Partitions
│   │   └── Brokers
│   │
│   ├── AWS IAM Fundamentals
│   │   ├── IAM Users
│   │   ├── IAM Roles
│   │   ├── IAM Policies
│   │   ├── Identity vs Resource policies
│   │   └── Least Privilege principle
│   │
│   ├── MSK Basics
│   │   ├── What MSK cluster is
│   │   ├── Broker endpoints
│   │   ├── VPC-based deployment
│   │   └── Client connectivity basics
│   │
│   └── Network Basics
│       ├── VPC
│       ├── Security Groups
│       ├── Subnets
│       └── Ports used by Kafka
│
├── 2. IAM Authentication in MSK
│   ├── How IAM Authentication Works
│   │   ├── IAM Identity sends request
│   │   ├── MSK validates IAM credentials
│   │   ├── Temporary tokens generation
│   │   └── TLS-secured connection
│   │
│   ├── IAM Authentication Mechanism
│   │   ├── SASL/IAM protocol
│   │   ├── Token-based authentication
│   │   └── AWS SDK role-based signing
│   │
│   ├── Supported Clients
│   │   ├── Java Kafka clients
│   │   ├── Python clients
│   │   ├── Kafka CLI tools
│   │   └── Lambda/MSK integration
│   │
│   └── Required Components
│       ├── IAM Role
│       ├── IAM Policy
│       ├── MSK Cluster with IAM enabled
│       └── TLS enabled brokers
│
├── 3. IAM Authorization in MSK
│   ├── Kafka-Level Permissions
│   │   ├── Connect to cluster
│   │   ├── Create topics
│   │   ├── Read from topics
│   │   ├── Write to topics
│   │   └── Describe topics
│   │
│   ├── IAM Policy Actions (Very Important)
│   │   ├── kafka-cluster:Connect
│   │   ├── kafka-cluster:DescribeCluster
│   │   ├── kafka-cluster:CreateTopic
│   │   ├── kafka-cluster:ReadData
│   │   ├── kafka-cluster:WriteData
│   │   └── kafka-cluster:AlterTopic
│   │
│   ├── Resource-Level Permissions
│   │   ├── Cluster ARN
│   │   ├── Topic ARN
│   │   └── Group ARN
│   │
│   └── Least Privilege Design
│       ├── Producer-only role
│       ├── Consumer-only role
│       └── Admin role
│
├── 4. Client Configuration with IAM
│   ├── Java Kafka Client Setup
│   │   ├── IAM auth plugin
│   │   ├── JAAS config
│   │   ├── TLS configuration
│   │   └── Bootstrap broker config
│   │
│   ├── Producer Configuration
│   │   ├── security.protocol=SASL_SSL
│   │   ├── sasl.mechanism=AWS_MSK_IAM
│   │   └── IAM role usage
│   │
│   ├── Consumer Configuration
│   │   ├── Consumer group setup
│   │   ├── IAM permission mapping
│   │   └── Topic access validation
│   │
│   └── CLI Tools with IAM
│       ├── kafka-console-producer
│       └── kafka-console-consumer
│
├── 5. MSK Resource ARNs Understanding
│   ├── Cluster ARN
│   ├── Topic ARN
│   ├── Group ARN
│   └── Transactional ID ARN
│
├── 6. Monitoring & Troubleshooting IAM Access
│   ├── CloudWatch Logs
│   ├── Authentication failures
│   ├── Authorization failures
│   ├── Consumer lag issues
│   └── Policy debugging
│
├── 7. Integration Patterns
│   ├── Lambda → MSK
│   ├── EC2 → MSK
│   ├── ECS/EKS → MSK
│   ├── Cross-account MSK access
│   └── PrivateLink connectivity
│
├── 8. Security Best Practices
│   ├── Use IAM Roles instead of users
│   ├── Enforce TLS always
│   ├── Restrict topic-level permissions
│   ├── Rotate credentials
│   └── Enable audit logging
│
└── 9. Advanced Topics
    ├── Multi-account MSK access
    ├── Custom IAM policy design
    ├── Cross-region Kafka access
    ├── Hybrid (On-prem → MSK)
    └── IAM vs Kafka ACL comparison
```