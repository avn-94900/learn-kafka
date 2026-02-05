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

// Python example using kafka-python library
/*
from kafka import KafkaProducer
from kafka.partitioner import RoundRobinPartitioner
import json

# Create producer with default partitioner
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None
)

# Method 1: Direct partition assignment
producer.send('my-topic', 
              value={'message': 'direct partition'}, 
              key='key1', 
              partition=0)  # Explicitly send to partition 0

# Method 2: Key-based (default behavior)
producer.send('my-topic', 
              value={'message': 'key-based'}, 
              key='user123')  # Partition determined by key hash

# Method 3: Custom partitioner
def custom_partition_function(key_bytes, all_partitions, available_partitions):
    if key_bytes and key_bytes.startswith(b'vip-'):
        return 0
    return hash(key_bytes) % len(available_partitions) if key_bytes else 0

producer_custom = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    partitioner=custom_partition_function,
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None
)

producer.flush()
producer.close()
*/
```