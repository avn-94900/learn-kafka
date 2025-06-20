# Top Apache Kafka Interview Questions for Java Backend Developers

## Fundamental Concepts

### 1. **What is Apache Kafka and what are its key features?**
   - Apache Kafka is a distributed streaming platform that allows you to publish, subscribe to, store, and process streams of records in real-time.
   - Key features: high throughput, fault tolerance, horizontal scalability, durability, and exactly-once semantics.

### 2. **Explain the core components of Kafka architecture.**
   - Producer API: Allows applications to send data to Kafka topics
   - Consumer API: Allows applications to read data from Kafka topics
   - Streams API: Allows transforming data from input to output topics
   - Connect API: Allows implementing connectors to integrate with external systems
   - Broker: Kafka server that stores data and serves client requests
   - ZooKeeper/KRaft: For cluster coordination and metadata management

### 3. **What are Kafka topics and partitions?**
   - Topics: Categories or feeds to which records are published
   - Partitions: Topics are divided into partitions, which are ordered, immutable sequences of records
   - Each partition has a configurable retention period and can be distributed across multiple brokers  
  
---  
  ### Can each Kafka partition have a different retention period?
  Apache Kafka allows you to set different retention periods **per topic**, but **not per partition within a topic**.
  
  ### Key Points:
  
  **1. Per-topic retention configuration:**
  You can configure a custom `retention.ms` value for each topic.
  Example command:
  
  ```bash
  kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type topics --entity-name my-topic \
    --alter --add-config retention.ms=86400000  # 1 day
  ```
  
  **2. Not per-partition:**
  Kafka does not support setting different retention values for individual partitions within the same topic. All partitions of a topic share the same configuration, including `retention.ms`, `segment.ms`, etc.
  
  ### Workaround:
  
  If you need different retention for different types of data:
  
  * Create separate topics (e.g., `topic-A`, `topic-B`), each with its own `retention.ms` setting.
  * Send data to the appropriate topic based on the desired retention policy.
  
  ### Summary:
  
  * Different topics can have different retention periods — this is supported.
  * Different partitions within the same topic cannot have different retention periods — not supported.
  
  ---
    
 
   
### 4. **How does Kafka ensure fault tolerance and high availability?**
   - Replication of partitions across multiple brokers
   - Election of leader partitions with follower replicas
   - Configurable acknowledgment levels for producers
   - Ability to recover from broker failures without data loss

### 5. **What are consumer groups and how do they work?**
   - Consumer groups allow multiple consumers to distribute the processing load
   - Each partition is consumed by exactly one consumer within a group
   - Adding consumers (up to the number of partitions) increases throughput
   - If a consumer fails, its partitions are reassigned to other members

### 6. **What are producer acknowledgment modes in Kafka? What's the difference between them?**
   - `acks=0`: Fire and forget (no acknowledgment)
   - `acks=1`: Leader acknowledgment only
   - `acks=all`: Full acknowledgment (leader and all in-sync replicas)
   - Trade-offs involve throughput vs. reliability


### 7. **Explain auto.offset.reset configuration in Kafka consumer.**

   - `earliest`: Start consuming from the beginning of the partition
   - `latest`: Start consuming only new messages (default)
   - `none`: Throw exception if no previous offset is found

### 8. **What's the difference between automatic and manual offset commitment?**
   - Automatic: Consumer automatically commits offsets periodically (enable.auto.commit=true)
   - Manual: Developer explicitly commits offsets using commitSync() or commitAsync()
      - Manual offers better control but requires more careful implementation

### 9. **Explain the role of ZooKeeper in Kafka. What is KRaft mode?**
   - ZooKeeper manages broker coordination, leader election, topic configuration
   - KRaft (Kafka Raft) is Kafka's native metadata quorum that allows Kafka to run without ZooKeeper
   - KRaft simplifies the architecture and improves scalability

## Advanced Concepts

### 1. **What is exactly-once semantics in Kafka? How can you implement it?**

   - Guarantees each message is processed exactly once, not lost or duplicated
   - Implemented using:
     - Idempotent producers (enable.idempotence=true)
     - Transactional API for atomic multi-partition writes
     - Isolation levels for consumers (read_committed)

### 2. **Explain Kafka Streams and its key concepts.**
   - High-level API for building stream processing applications
   - Key abstractions: KStream (record stream), KTable (changelog stream)
   - Stateless operations: map, filter, flatMap
   - Stateful operations: aggregations, joins, windowing
   - Exactly-once processing guarantees

### 3. **How would you implement a custom serializer and deserializer in Kafka?**

```java
public class CustomSerializer implements Serializer<MyObject> {
    @Override
    public byte[] serialize(String topic, MyObject data) {
        // Serialize MyObject to byte array
        // Often uses JSON/Avro/Protobuf libraries
        return serializedBytes;
    }
}

public class CustomDeserializer implements Deserializer<MyObject> {
    @Override
    public MyObject deserialize(String topic, byte[] data) {
        // Deserialize byte array back to MyObject
        return deserializedObject;
    }
}
```

### 4. **What is Kafka Connect and how would you use it?**
   - Framework for connecting Kafka with external systems
   - Source connectors: Import data from external systems into Kafka
   - Sink connectors: Export data from Kafka to external systems
   - Distributed and standalone modes
   - REST API for managing connectors

## Performance and Monitoring

### 1. **How do you tune Kafka for optimal performance?**
   - Producer tuning: batch.size, linger.ms, compression
   - Consumer tuning: fetch.min.bytes, fetch.max.wait.ms
   - Broker tuning: num.network.threads, num.io.threads
   - Topic tuning: partitions, replication factor, retention policies
   - JVM tuning: heap size, garbage collection

### 2. **What metrics would you monitor in a Kafka cluster?**
   - Broker metrics: under-replicated partitions, request rate, disk usage
   - Producer metrics: batch size, record send rate, request latency
   - Consumer metrics: consumer lag, poll rate, processing time
   - JVM metrics: GC pause times, heap usage
   - OS metrics: CPU, memory, disk I/O, network utilization

### 3. **How do you handle large message sizes in Kafka?**
   - Use compression (snappy, gzip, lz4)
   - Increase message.max.bytes on broker
   - Increase max.request.size in producer
   - Consider storing large payloads externally (e.g., S3) and only send references

## Troubleshooting and Best Practices

### 1. **What are some common issues you might encounter with Kafka and how would you resolve them?**
   - Consumer lag: Increase consumer instances, optimize processing, rebalance partitions
   - Producer timeouts: Check network, increase request.timeout.ms
   - Broker failures: Ensure proper replication, monitor under-replicated partitions
   - Out of memory errors: Tune JVM heap, check for memory leaks

### 2. **Explain Kafka security mechanisms and how you would implement them.**
   - Authentication: SSL/TLS, SASL (PLAIN, SCRAM, Kerberos)
   - Authorization: ACLs to control access to topics and operations
   - Encryption: SSL/TLS for in-transit encryption
   - Implementation involves configuring security properties in producers/consumers

### 3. **What are some best practices for Kafka topic design?**

   - Choose appropriate key for partitioning
   - Calculate optimal partition count based on throughput requirements
   - Use compacted topics for state stores
   - Consider data retention and cleanup policies

### 4. **How would you handle schema evolution in Kafka?**

   - Use schema registry with Avro, Protobuf, or JSON Schema
   - Follow compatibility rules: backward, forward, or full compatibility
   - Implement proper versioning strategies
   - Consider using generic records for maximum flexibility

