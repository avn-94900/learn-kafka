# Consolidated Kafka Notes

## Table of Contents

1. [Introduction and Basics](#1-introduction-and-basics)
2. [Kafka Architecture](#2-kafka-architecture)
3. [Producers](#3-producers)
4. [Consumers](#4-consumers)
5. [Configuration](#5-configuration)
6. [Exactly Once Semantics](#6-exactly-once-semantics)
7. [Advanced Topics](#7-advanced-topics)
8. [Interview Questions](#8-interview-questions)
9. [Code Examples and Projects](#9-code-examples-and-projects)
10. [Spring Boot Integration](#10-spring-boot-integration)

---

## 1. Introduction and Basics

### What is Apache Kafka

Apache Kafka is a distributed streaming platform designed to handle high-throughput, fault-tolerant data pipelines. It provides:

- Asynchronous messaging model (non-blocking network calls between services)
- Publish/subscribe model
- Mediates communication between services

### Core Components

#### Kafka Cluster

A Kafka cluster is a distributed system consisting of multiple servers working together.

**Characteristics:**
- Collection of multiple Kafka brokers operating as a single system
- Provides scalability, fault tolerance, and high availability
- Minimum 3 brokers recommended for production environments

#### Kafka Broker

A broker is an individual Kafka server instance that acts as an intermediary between data producers and consumers.

**Responsibilities:**
- Kafka server that stores data and serves client requests
- Mediates communication between producers and consumers
- Receives, stores, and allows retrieval of messages

#### Kafka Producer

Producers are applications that publish data to Kafka topics.

**Behavior:**
- Application that sends messages to Kafka
- Sends messages to Kafka brokers, not directly to consumers
- Determines which topic and partition to send messages to

#### Kafka Consumer

Consumers are applications that subscribe to topics and process published messages.

**Capabilities:**
- Application that reads messages from Kafka
- Requests and reads data from Kafka brokers
- Can consume messages from any producer (with appropriate permissions)

## Topic and Partition Concepts

### Kafka Topic

Topics are logical channels or categories to which messages are published.

**Properties:**
- Named feed or category of messages
- Similar to a table in a database or folder in a file system
- Each topic consists of one or more partitions
- You can create any number of topics based on needs

### Kafka Partitions

Partitions are the units of parallelism in Kafka that allow data distribution across multiple brokers.

**Characteristics:**
- Ordered, immutable sequence of records within a topic
- Enables horizontal scaling and parallel processing
- Distributed across multiple brokers for load balancing
- Inside a single partition, messages are always ordered by offset
- Once a message is written, it is never changed (immutable)
- Consumers read messages in order, one by one

## Partitioner Logic

When a producer sends a message to Kafka, a Partitioner determines which partition within a topic the message should go to.

**Partitioning Rules:**

| Scenario | Partitioning Logic |
|----------|-------------------|
| **Key provided** | Hash function applied to key: `hash(key) % number of partitions` |
| **No key provided** | Round-robin approach to distribute messages evenly |
| **Custom partitioner** | User-defined routing logic based on application needs |

## Offset Management

### What are Offsets

Offsets are sequential identifiers assigned to messages within a partition.

**Properties:**
- Unique, sequential identifier for each message in a partition
- First message gets offset 0, next gets 1, and so on
- Once assigned, an offset never changes
- Consumers use offsets to track their position in a partition

### Consumer Groups

Consumer groups allow multiple consumers to collaborate on processing messages from topics.

**Benefits:**
- Collection of consumers working together to process messages
- Partitions are distributed among consumers in a group
- Adding consumers to a group (up to number of partitions) increases throughput
- If a consumer fails, its partitions are reassigned to other group members, providing Fault Tolerance.

## Message Structure

Each Kafka message (record) includes:

| Component | Description | Required |
|-----------|-------------|----------|
| **Topic** | Logical category or name | Yes |
| **Key** | Used to determine partition; groups related messages | Optional |
| **Value** | Actual business data or payload | Yes |
| **Timestamp** | Time when message was produced | Auto-generated |
| **Partition** | Specific partition within topic | Auto-assigned if not specified |

## Serialization

### What is a Serializer

Kafka stores all data as byte arrays. A Serializer converts keys and values into byte arrays before sending to Kafka.

**Built-in Serializers:**
- String
- Integer
- Long
- Byte array

**Custom Serializers:**
Can be created for complex data types like JSON objects or custom classes.

## Key Concepts Summary

| Concept | Purpose | Key Benefit |
|---------|---------|-------------|
| **Cluster** | Group of brokers | High availability and scalability |
| **Broker** | Individual server | Message storage and serving |
| **Topic** | Message category | Logical organization |
| **Partition** | Topic subdivision | Parallel processing |
| **Producer** | Message sender | Data ingestion |
| **Consumer** | Message receiver | Data processing |
| **Consumer Group** | Consumer coordination | Load balancing |
| **Offset** | Message position | Progress tracking |

---

## 2. Kafka Architecture

### High-Level Architecture Overview

Kafka follows a distributed architecture pattern with the following key characteristics:
- **Distributed System**: Multiple brokers working together as a cluster
- **Horizontal Scalability**: Can scale by adding more brokers
- **Fault Tolerance**: Data replication across multiple brokers
- **Decoupled Communication**: Producers and consumers operate independently

### Cluster Components
A Kafka cluster consists of multiple interconnected components working together:

### Cluster Characteristics
- **Minimum Brokers**: At least 3 brokers recommended for production
- **Odd Number**: Typically deployed with odd number of brokers for quorum
- **Network Communication**: Brokers communicate via TCP protocols
- **Cluster ID**: Each cluster has a unique identifier

### Broker Architecture

Each broker is a JVM process with several key components:

### Broker Responsibilities
- **Message Storage**: Stores messages in log segments on disk
- **Client Handling**: Processes producer and consumer requests
- **Replication**: Maintains replicas of partitions from other brokers
- **Leader Election**: Participates in partition leadership elections
- **Metadata Management**: Maintains cluster and topic metadata

### Topic and Partition Architecture

Topics are distributed across the cluster through partitions:

### Partition Leadership Model
- **Leader Replica**: Handles all reads and writes for a partition
- **Follower Replicas**: Replicate data from the leader
- **In-Sync Replicas (ISR)**: Replicas that are caught up with the leader
- **Preferred Leader**: The first replica in the replica list

### Replication Architecture

Kafka ensures fault tolerance through replication:

### Replication Process
1. **Producer Sends Message**: Message sent to partition leader
2. **Leader Writes**: Leader writes message to its local log
3. **Follower Fetch**: Followers fetch new messages from leader
4. **Acknowledgment**: Leader waits for replicas based on `acks` setting
5. **Commit**: Message is committed when written to min ISR replicas

### Replication Configuration
- **Replication Factor**: Number of replicas for each partition
- **Min ISR**: Minimum number of in-sync replicas required
- **Unclean Leader Election**: Whether to allow out-of-sync replicas to become leaders

### ZooKeeper Integration

ZooKeeper manages critical cluster metadata and coordination:

### ZooKeeper Responsibilities
- **Broker Registration**: Tracks live brokers in the cluster
- **Topic Metadata**: Stores topic and partition information
- **Controller Election**: Elects one broker as the cluster controller
- **Configuration Management**: Stores dynamic configurations
- **Consumer Coordination**: Manages consumer group memberships (older clients)

**Note**: Kafka is moving away from ZooKeeper dependency with KRaft (Kafka Raft) mode in newer versions.

### Controller Architecture

One broker in the cluster acts as the controller:

### Controller Election Process
1. **Startup**: All brokers attempt to create `/controller` node in ZooKeeper
2. **Election**: First broker to create the node becomes controller
3. **Notification**: Controller registers watch on broker changes
4. **Failover**: If controller fails, new election occurs automatically

### Producer Architecture Integration

### Producer Flow
1. **Metadata Request**: Producer requests cluster metadata
2. **Partition Assignment**: Determines target partition using partitioner
3. **Batching**: Groups messages into batches for efficiency
4. **Compression**: Applies compression if configured
5. **Send**: Sends batch to partition leader
6. **Acknowledgment**: Receives confirmation based on `acks` setting

### Consumer Architecture Integration

### Consumer Group Protocol
1. **Join Group**: Consumers join the group and elect a leader
2. **Sync Group**: Leader assigns partitions and shares with group
3. **Heartbeat**: Consumers send heartbeats to stay in group
4. **Rebalance**: Triggered when consumers join/leave or partitions change

### End-to-End Message Flow

### Message Lifecycle in Architecture
1. **Production**: Producer serializes and sends message to leader
2. **Storage**: Leader writes to log and replicates to followers
3. **Indexing**: Broker updates offset and time indexes
4. **Consumption**: Consumer polls leader for messages
5. **Processing**: Consumer deserializes and processes messages
6. **Commitment**: Consumer commits offsets for processed messages

### Fault Tolerance Architecture

#### Broker Failure

#### Network Partition
- **Split Brain Prevention**: Minimum ISR prevents data loss
- **Leader Election**: New leader elected from ISR members
- **Producer Behavior**: Continues writing to available brokers
- **Consumer Behavior**: Continues reading from available replicas

### High Availability Features
- **Replication**: Multiple copies of data across brokers
- **Leader Election**: Automatic failover for partition leadership
- **ISR Management**: Maintains list of caught-up replicas
- **Rack Awareness**: Distributes replicas across different racks/zones

### Performance Architecture Considerations

#### Disk I/O Optimization
- **Sequential Writes**: All writes are sequential (append-only)
- **Page Cache**: Leverages OS page cache for reads
- **Zero-Copy**: Uses sendfile() system call for efficient data transfer
- **Compression**: Reduces disk usage and network I/O

#### Network Optimization
- **Batching**: Groups multiple messages in single network request
- **Pipelining**: Multiple in-flight requests to maximize throughput
- **Connection Pooling**: Reuses connections between clients and brokers

#### Memory Management
- **Off-Heap Storage**: Uses file system cache instead of JVM heap
- **Message Batching**: Reduces object creation overhead
- **Compression**: Reduces memory footprint of batched messages

### Scaling Architecture Patterns

#### Horizontal Scaling
- **Add Brokers**: Increase cluster capacity by adding more brokers
- **Partition Increase**: Add partitions to increase parallelism
- **Consumer Scaling**: Add consumers up to partition count
- **Topic Separation**: Separate different data types into different topics

#### Vertical Scaling
- **Disk Capacity**: Add more/faster disks to brokers
- **Memory**: Increase broker memory for better caching
- **CPU**: More cores for handling concurrent requests
- **Network**: Higher bandwidth for increased throughput

This architecture enables Kafka to handle massive scale deployments with high throughput, low latency, and strong durability guarantees.

---

## 3. Producers

### Producer Fundamentals

#### Producer Architecture

A Kafka producer is responsible for publishing messages to Kafka topics. The producer handles:

- Message serialization
- Partition assignment
- Batching for efficiency
- Retry logic for failed sends

#### Producer Configuration

##### Key Producer Settings

| Configuration | Description | Default | Impact |
|---------------|-------------|---------|--------|
| `bootstrap.servers` | Kafka broker addresses | None | Connection |
| `key.serializer` | Key serialization class | None | Data format |
| `value.serializer` | Value serialization class | None | Data format |
| `acks` | Acknowledgment level | 1 | Durability |
| `retries` | Retry attempts | Integer.MAX_VALUE | Reliability |
| `batch.size` | Batch size in bytes | 16384 | Throughput |
| `linger.ms` | Batching delay | 0 | Latency vs throughput |

##### Producer Acknowledgment Modes

| ACK Setting | Behavior | Durability | Performance |
|-------------|----------|------------|-------------|
| `acks=0` | No acknowledgment (fire-and-forget) | Lowest | Highest |
| `acks=1` | Leader acknowledgment only | Medium | Medium |
| `acks=all` | All in-sync replicas acknowledge | Highest | Lowest |

#### Producer Code Example

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");
props.put("retries", 3);

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>(
    "my-topic",     // topic
    "user-123",     // key
    "Hello World"   // value
);

producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        System.out.println("Message sent to partition: " + metadata.partition() + 
                          " with offset: " + metadata.offset());
    } else {
        exception.printStackTrace();
    }
});

producer.close();
```

#### Producer Retry Mechanism

##### Retriable vs Non-Retriable Errors

| Error Type | Examples | Producer Behavior |
|------------|----------|-------------------|
| **Retriable** | Network timeouts, leader not available | Automatic retry |
| **Non-Retriable** | Invalid topic, message too large | Immediate failure |

##### Retry Configuration

```properties
retries=Integer.MAX_VALUE
retry.backoff.ms=100
request.timeout.ms=30000
delivery.timeout.ms=120000
```

---

## 4. Consumers

### Consumer Fundamentals

#### Consumer Architecture

A Kafka consumer reads messages from Kafka topics. The consumer handles:

- Message deserialization
- Offset management
- Partition assignment coordination
- Rebalancing within consumer groups

#### Consumer Configuration

##### Key Consumer Settings

| Configuration | Description | Default | Impact |
|---------------|-------------|---------|--------|
| `bootstrap.servers` | Kafka broker addresses | None | Connection |
| `group.id` | Consumer group identifier | None | Group membership |
| `key.deserializer` | Key deserialization class | None | Data format |
| `value.deserializer` | Value deserialization class | None | Data format |
| `auto.offset.reset` | Offset reset strategy | latest | Starting position |
| `enable.auto.commit` | Automatic offset commit | true | Offset management |
| `max.poll.records` | Records per poll | 500 | Batch size |

##### Auto Offset Reset Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| `earliest` | Start from beginning of partition | Process all historical data |
| `latest` | Start from newest messages | Process only new data |
| `none` | Throw exception if no offset found | Strict offset management |

#### Consumer Code Example

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-consumer-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("auto.offset.reset", "earliest");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("my-topic"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("Consumed message: key=%s, value=%s, partition=%d, offset=%d%n",
                         record.key(), record.value(), record.partition(), record.offset());
    }
}
```

## Offset Management

### Automatic Offset Commit

When `enable.auto.commit=true`:

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000  # Commit every 5 seconds
```

**Pros:**
- Simple to implement
- No additional code required

**Cons:**
- Risk of message loss if consumer crashes after commit but before processing
- Risk of duplicate processing if consumer crashes after processing but before commit

### Manual Offset Commit

When `enable.auto.commit=false`:

```java
// Synchronous commit
consumer.commitSync();

// Asynchronous commit
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        System.err.println("Commit failed: " + exception.getMessage());
    }
});

// Commit specific offsets
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("my-topic", 0), new OffsetAndMetadata(100));
consumer.commitSync(offsets);
```

### Offset Commit Strategies

| Strategy | When to Use | Trade-offs |
|----------|-------------|------------|
| **Auto-commit** | Simple applications, acceptable message loss | Easy but less control |
| **Manual sync** | Critical applications, no message loss | More control but blocking |
| **Manual async** | High-throughput applications | Non-blocking but complex error handling |

## Consumer Groups and Rebalancing

### Consumer Group Coordination

Consumer groups enable:
- Load balancing across multiple consumers
- Fault tolerance through automatic failover
- Scalability by adding/removing consumers

### Rebalancing Process

1. **Trigger Events:**
   - Consumer joins or leaves group
   - Partition count changes
   - Consumer heartbeat timeout

2. **Rebalancing Steps:**
   - Stop consuming messages
   - Revoke current partition assignments
   - Reassign partitions among active consumers
   - Resume consuming from new assignments

### Rebalancing Configuration

```properties
session.timeout.ms=30000        # Time before consumer considered dead
heartbeat.interval.ms=3000      # Heartbeat frequency
max.poll.interval.ms=300000     # Max time between polls
```

## Producer-Consumer Interaction Patterns

### One-to-One Pattern

```
Producer → Topic (Single Partition) → Consumer
```

**Use Case:** Ordered processing, simple workflows

### One-to-Many Pattern

```
Producer → Topic (Multiple Partitions) → Consumer Group (Multiple Consumers)
```

**Use Case:** Parallel processing, high throughput

### Many-to-Many Pattern

```
Multiple Producers → Topic → Multiple Consumer Groups
```

**Use Case:** Event streaming, microservices communication

## Best Practices

### Producer Best Practices

1. **Use appropriate serializers** for your data format
2. **Set proper acks level** based on durability requirements
3. **Configure retries** for network resilience
4. **Use batching** for better throughput
5. **Handle exceptions** properly in callbacks

### Consumer Best Practices

1. **Choose appropriate offset management** strategy
2. **Handle rebalancing** gracefully
3. **Process messages idempotently** when possible
4. **Monitor consumer lag** regularly
5. **Use proper deserialization** error handling

### Performance Considerations

| Component | Optimization | Benefit |
|-----------|--------------|---------|
| **Producer** | Increase batch.size and linger.ms | Higher throughput |
| **Producer** | Use compression | Reduced network usage |
| **Consumer** | Increase max.poll.records | Fewer network calls |
| **Consumer** | Optimize processing logic | Reduced lag |
| **Both** | Use appropriate serialization | Better performance |

---

## 5. Configuration

### Kafka's message retention policy and configuration

#### Valid Combined Configuration (no error):

```properties
log.retention.ms=604800000       # Retain messages for 7 days
log.retention.bytes=1073741824   # Retain up to 1 GB of log data
log.cleanup.policy=delete        # Use time/size-based deletion (default)
```

#### How Kafka Applies These Together:

Kafka uses **both time and size** thresholds:

* If **retention.ms** is reached → Kafka starts deleting old log segments.
* If **retention.bytes** is reached → Kafka deletes oldest segments to keep size under the limit.
* If **both are configured**, **whichever condition is met first** will trigger cleanup.

> This is totally safe and **recommended for production** to control disk usage and data lifecycle.

#### Notes:

* These apply to **all topics** unless overridden with per-topic configs.
* You must **restart the Kafka broker** after changing `server.properties` for the changes to take effect.

### Kafka server Backpressure Handling:

#### What This Means:

```properties
buffer.memory=67108864       # 64 MB buffer in the producer
max.block.ms=30000           # Wait up to 30 seconds if buffer is full
```

#### Step-by-step Behavior:

1. **Producer sends messages to Kafka**.

2. If the broker is slow or overloaded:

   * Messages are not immediately sent.
   * They are **temporarily stored in the producer's memory buffer** (up to 64 MB, as per `buffer.memory`). 

3. If the buffer fills up:

   * The producer **pauses sending new messages**.
   * It **waits up to 30 seconds** (as per `max.block.ms`) for space to become available in the buffer (i.e., for old messages to be sent out).

4. If after 30 seconds:

   * Messages are still not sent (due to broker unavailability or continued slowness),
   * The producer **throws an exception**, typically `TimeoutException`.

#### Example Exception:

```java
org.apache.kafka.common.errors.TimeoutException: 
Failed to allocate memory within the configured max blocking time 30000 ms
```

#### Kafka Handles It Like This:

```text
[Producer] --> [Buffer (buffer.memory)] --> [Kafka Broker]

               ^         ↑            ↓
               |         |            |
         (if full)   wait (max.block.ms)   then send to broker
```

#### Summary:

| Setting                       | Purpose                                       |
| ----------------------------- | --------------------------------------------- |
| `buffer.memory`               | How much data the producer can buffer locally |
| `max.block.ms`                | How long to wait when buffer is full          |
| If buffer full + time exceeds | ➡️ Throw error (backpressure response)        |

### Kafka Consumer Lag Management

* Producer wrote messages with offsets: `100 to 200`
* Consumer has only read up to offset `150`

  **Consumer Lag = 200 - 150 = 50 messages**

#### How to Manage Consumer Lag in Kafka

#### 1. Monitor Lag

Use tools like:

* Prometheus + Grafana
* Kafka Manager
* Confluent Control Center
* Kafka Exporter

To track:

* Lag per consumer group
* Partition-level lag

Metric:

```
kafka.consumer.ConsumerFetcherManager.metrics.records-lag
```

#### 2. Tune Consumer Settings

Key configs:

| Property               | Description                                           |
| ---------------------- | ----------------------------------------------------- |
| `max.poll.records`     | Controls how many messages to fetch per poll.         |
| `max.poll.interval.ms` | If processing takes too long, this must be increased. |
| `fetch.max.bytes`      | Max bytes fetched per request.                        |
| `session.timeout.ms`   | Time after which Kafka marks consumer as dead.        |

#### 3. Scale Consumers

* If lag is consistently growing, add more consumer **instances** to the **consumer group**.
* Each consumer in the group handles **one or more partitions**, so adding more reduces the load per consumer.

#### 4. Optimize Processing Logic

* If a consumer is slow due to business logic:

  * Offload to **async processing**
  * Use **worker threads or a thread pool**
  * Avoid long blocking calls

#### 5. Increase Topic Retention

If you expect consumers to lag:

```properties
log.retention.ms=604800000   # Retain data for 7 days
```

This ensures that **messages are not deleted** before slow consumers read them.

#### Why You Should Care?

If consumer lag is high:

* Real-time applications may become **outdated**
* Risk of **data loss** (if retention time is exceeded)
* You'll have **high disk usage** on Kafka brokers

### Message delivery semantics in Kafka — `at-most-once`, `at-least-once`, and `exactly-once`

#### 1. At-Most-Once (Fastest, May Lose Messages)

> Messages are delivered **zero or one time**. If failure happens after send, **message is lost**.

##### Producer Config (application.yml)

```yaml
spring:
  kafka:
    producer:
      acks: 0                          # No acknowledgment from broker (fire-and-forget)
      retries: 0                       # No retries if send fails
      enable-idempotence: false        # No deduplication logic
```

##### Consumer Config

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: true         # Offsets auto-committed (even before processing)
      auto-commit-interval: 1000       # Commit offset every 1 sec (default)
```

#### 2. At-Least-Once (Reliable, May Duplicate)

> Messages are delivered **one or more times**. Duplicates can occur due to retries.

##### Producer Config

```yaml
spring:
  kafka:
    producer:
      acks: all                        # Wait for all in-sync replicas to acknowledge
      retries: 5                       # Retry sending if broker fails
      enable-idempotence: false        # No deduplication; may cause duplicates
```

##### Consumer Config

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false        # Prevent auto-offset commit
      auto-offset-reset: earliest      # Start from earliest if no offset found
```

> You must **manually commit offsets** after processing, using:

```java
consumer.commitSync();
```

#### 3. Exactly-Once (Reliable + No Duplicates or Loss)

> Messages are **delivered once and only once**, even with retries and failures.

##### Producer Config

```yaml
spring:
  kafka:
    producer:
      acks: all                                 # Strong durability: wait for all replicas
      retries: 5                                # Retry send on failure
      enable-idempotence: true                  # Enable deduplication for safe retries
      transactional-id-prefix: txn-id           # Enables transactional producer with prefix
```

##### Consumer Config

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false                 # Manual offset commit required
      isolation-level: read_committed           # Read only committed transactional messages
```

> In your code, wrap send + offset commit inside a **transaction**:

```java
kafkaTemplate.executeInTransaction(ops -> {
    // send message
    // manually commit consumer offset (with KafkaTransactionManager)
    return true;
});
```

#### Summary Table

| Semantics     | Duplicates | Loss  | Config Key Points                                                             |
| ------------- | ---------- | ----- | ----------------------------------------------------------------------------- |
| At-most-once  | ❌ No       | ✅ Yes | `acks: 0`, `enable-idempotence: false`, `auto-commit: true`                   |
| At-least-once | ✅ Yes      | ❌ No  | `acks: all`, `retries: 5`, `enable-idempotence: false`, `manual commit`       |
| Exactly-once  | ❌ No       | ❌ No  | `acks: all`, `enable-idempotence: true`, `transactional-id`, `read_committed` |

---

## 6. Exactly Once Semantics

### How Exactly-Once Semantics (EOS) Works in Kafka

Exactly-Once Semantics is one of the most important features in Kafka, introduced in version 0.11.0. It guarantees that each message is processed exactly once, even in the presence of failures.

#### Key Components of Kafka's EOS Implementation:

1. **Idempotent Producer**:
   - Each producer is assigned a unique Producer ID (PID)
   - Producers attach sequence numbers to each message
   - Brokers track these sequence numbers to detect and reject duplicates
   - Ensures "exactly-once" delivery between producer and broker

2. **Transactional API**:
   - Allows grouping multiple produce operations into an atomic unit
   - Uses a transaction coordinator (a specialized broker)
   - Creates transaction logs with unique transaction IDs
   - Enables atomicity across multiple topic-partitions

3. **Consumer Transaction Awareness**:
   - Consumers can be configured to read only committed transactions
   - Uses `isolation.level` configuration:
     - `read_committed`: only reads committed transactions
     - `read_uncommitted`: reads all messages (default)

4. **End-to-End EOS Process Flow**:
   - Producer initiates transaction with `beginTransaction()`
   - Messages are sent within the transaction context
   - Transaction is either committed (`commitTransaction()`) or aborted (`abortTransaction()`)
   - Transaction coordinator writes the final transaction status to the log
   - Consumers reading with `read_committed` see only committed messages

#### The Problem EOS Solves

Without EOS, Kafka could provide:
- **At-least-once delivery**: Messages are never lost but might be duplicated
- **At-most-once delivery**: Messages might be lost but never duplicated

EOS provides the best of both worlds: messages are neither lost nor duplicated.

#### Key Components of Kafka's EOS Implementation:

1. **Idempotent Producer**:
   - Assigns a unique Producer ID (PID) to each producer
   - Attaches sequence numbers to messages
   - Brokers track these sequence numbers to detect and reject duplicates
   - Ensures "exactly-once" delivery between producer and broker

2. **Transactional API**:
   - Enables atomic writes across multiple partitions
   - Uses a transaction coordinator (a specialized broker)
   - Maintains transaction logs with unique transaction IDs
   - Ensures all messages in a transaction are committed or none are

3. **Consumer Transaction Awareness**:
   - Uses `isolation.level` configuration:
     - `read_committed`: only reads committed transactions
     - `read_uncommitted`: reads all messages (default)

4. **End-to-End EOS Process**:
   - Producer initiates transaction with `beginTransaction()`
   - Messages sent within the transaction context
   - Either `commitTransaction()` or `abortTransaction()`
   - Transaction coordinator records final status
   - Consumers with `read_committed` see only committed messages

#### Spring Boot Implementation Details

The Spring Boot project demonstrates EOS with:

1. **Configuration**:
   - Producer configured with `transaction-id-prefix` and `enable.idempotence=true`
   - Consumer configured with `isolation.level=read_committed`
   - `KafkaTransactionManager` to manage transactions

2. **Transactional Producer**:
   - Uses Spring's `@Transactional` annotation for simple transaction handling
   - Demonstrates single-message and batch-message transactions
   - Shows how exceptions automatically trigger transaction rollback

3. **Transaction-Aware Consumer**:
   - Consumes only committed transactions
   - Processes messages in its own transaction scope
   - Demonstrates offset commit as part of the transaction

4. **Key Concepts Demonstrated**:
   - Atomic multi-partition writes
   - Transaction rollback on failure
   - Read isolation for consumers
   - Exactly-once processing guarantees

This implementation is suited for scenarios like payment processing where transaction integrity is critical.

---

## 7. Advanced Topics

### Custom Partitioning

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.utils.Utils;
import java.util.Map;
import java.util.Properties;

public class KafkaPartitionExamples {
    
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        String topic = "my-topic";
        
        // Method 1: Direct Partition Assignment
        directPartitionExample(producer, topic);
        
        // Method 2: Key-based Hashing (default behavior)
        keyBasedPartitionExample(producer, topic);
        
        // Method 3: Custom Partitioner
        customPartitionerExample();
        
        producer.close();
    }
    
    // Method 1: Explicitly specify the partition
    public static void directPartitionExample(KafkaProducer<String, String> producer, String topic) {
        System.out.println("=== Direct Partition Assignment ===");
        
        // Send to partition 0 explicitly
        ProducerRecord<String, String> record1 = new ProducerRecord<>(
            topic,           // topic
            0,               // partition (explicitly set to 0)
            "key1",          // key
            "message1"       // value
        );
        
        // Send to partition 2 explicitly
        ProducerRecord<String, String> record2 = new ProducerRecord<>(
            topic,           // topic
            2,               // partition (explicitly set to 2)
            "key2",          // key
            "message2"       // value
        );
        
        producer.send(record1, (metadata, exception) -> {
            if (exception == null) {
                System.out.println("Message sent to partition: " + metadata.partition());
            }
        });
        
        producer.send(record2, (metadata, exception) -> {
            if (exception == null) {
                System.out.println("Message sent to partition: " + metadata.partition());
            }
        });
    }
    
    // Method 2: Let Kafka decide based on key hash (default behavior)
    public static void keyBasedPartitionExample(KafkaProducer<String, String> producer, String topic) {
        System.out.println("=== Key-based Partition Assignment ===");
        
        // Partition determined by hash of key
        ProducerRecord<String, String> record1 = new ProducerRecord<>(
            topic,
            "user123",       // key - will be hashed to determine partition
            "user data 1"
        );
        
        ProducerRecord<String, String> record2 = new ProducerRecord<>(
            topic,
            "user456",       // different key - may go to different partition
            "user data 2"
        );
        
        producer.send(record1, (metadata, exception) -> {
            if (exception == null) {
                System.out.println("Key 'user123' sent to partition: " + metadata.partition());
            }
        });
        
        producer.send(record2, (metadata, exception) -> {
            if (exception == null) {
                System.out.println("Key 'user456' sent to partition: " + metadata.partition());
            }
        });
    }
    
    // Method 3: Custom Partitioner Configuration
    public static void customPartitionerExample() {
        System.out.println("=== Custom Partitioner Configuration ===");
        
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // Set custom partitioner
        props.put("partitioner.class", "com.example.CustomPartitioner");
        
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        
        // Messages will be partitioned according to custom logic
        ProducerRecord<String, String> record = new ProducerRecord<>(
            "my-topic",
            "special-key",
            "custom partitioned message"
        );
        
        producer.send(record, (metadata, exception) -> {
            if (exception == null) {
                System.out.println("Custom partitioner sent to partition: " + metadata.partition());
            }
        });
        
        producer.close();
    }
}

// Custom Partitioner Implementation
class CustomPartitioner implements Partitioner {
    
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, 
                        Object value, byte[] valueBytes, Cluster cluster) {
        
        // Get available partitions for the topic
        int numPartitions = cluster.partitionCountForTopic(topic);
        
        if (key == null) {
            // Round-robin for null keys
            return (int) (System.currentTimeMillis() % numPartitions);
        }
        
        String keyStr = (String) key;
        
        // Custom logic: VIP users go to partition 0
        if (keyStr.startsWith("vip-")) {
            return 0;
        }
        
        // Priority users go to partition 1 (if available)
        if (keyStr.startsWith("priority-") && numPartitions > 1) {
            return 1;
        }
        
        // Regular users use hash-based partitioning for remaining partitions
        int regularPartitionStart = Math.min(2, numPartitions - 1);
        int availablePartitions = numPartitions - regularPartitionStart;
        
        if (availablePartitions <= 0) {
            return 0; // Fallback
        }
        
        return regularPartitionStart + (Utils.toPositive(Utils.murmur2(keyBytes)) % availablePartitions);
    }
    
    @Override
    public void close() {
        // Cleanup if needed
    }
    
    @Override
    public void configure(Map<String, ?> configs) {
        // Configuration if needed
    }
}
```

### Partition for Load Balancing Analogy

#### Scenario: E-commerce platform

**Services (Consumers / Consumer Groups)**:

1. **Order Service** → processes new orders
2. **Inventory Service** → updates stock
3. **Notification Service** → sends emails or SMS to users

**Producer**:

* **Frontend / Checkout Service** → sends events like `OrderCreated` to Kafka topic `OrdersTopic`.

**Kafka Topic**:

* `OrdersTopic` with 3 partitions (P1, P2, P3)

#### Step 1: Consumer groups

Each service is a **consumer group** because we want multiple instances of the service to scale horizontally but still process each message **once per service type**.

* **Group1 → OrderService instances** (C1, C2)
* **Group2 → InventoryService instances** (D1, D2)
* **Group3 → NotificationService instances** (E1, E2)

#### Step 2: Partition assignment (load balancing)

Imagine 3 partitions (P1, P2, P3) in `OrdersTopic`. Kafka distributes partitions among consumers **within each group**:

**Order Service (Group1)**

```
P1 → C1
P2 → C2
P3 → C1
```

* C1 handles P1 & P3
* C2 handles P2

**Inventory Service (Group2)**

```
P1 → D1
P2 → D2
P3 → D1
```

* D1 handles P1 & P3
* D2 handles P2

**Notification Service (Group3)**

```
P1 → E1
P2 → E2
P3 → E1
```

* E1 handles P1 & P3
* E2 handles P2

**Note:** Each group has its own independent assignment.

#### Step 3: How producer and consumer groups interact

1. **Producer** sends `OrderCreated` events → Kafka topic `OrdersTopic`
2. **Kafka partitions messages** across P1, P2, P3
3. **Kafka assigns partitions** to consumers in each consumer group

   * Each service instance reads **only its assigned partitions**
4. **Scaling**: If you add a new instance to a service (consumer group), Kafka redistributes partitions among instances automatically

#### Step 4: Benefits in real life

* **Horizontal scalability** → multiple instances of Order/Inventory/Notification Service
* **Load balancing** → partitions evenly divided among service instances
* **Fault tolerance** → if C1 crashes, its partitions (P1 & P3) are reassigned to C2
* **Independent processing** → Inventory service sees all orders, Notification service sees all orders, but they don't block each other

#### Analogy

* **Topic = Book of orders**

* **Partition = Chapter in the book**

* **Consumer group = Class of students reading the book**

* **Consumer instance = One student in the class**

* Each student in the same class reads **different chapters** → load balanced

* Another class can also read the same book independently → multiple consumer groups

---
