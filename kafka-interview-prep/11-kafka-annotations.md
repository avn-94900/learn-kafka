# 11. Kafka Annotations (Spring Boot Kafka)

---

### What are common Kafka annotations in Spring Boot?

| Annotation | Purpose |
|------------|---------|
| `@EnableKafka` | Enables Kafka listener infrastructure |
| `@KafkaListener` | Marks a method as a Kafka message consumer |
| `@KafkaHandler` | Handles specific message types within a `@KafkaListener` class |
| `@RetryableTopic` | Configures non-blocking retry with retry topics |
| `@DltHandler` | Marks the Dead Letter Topic handler method |
| `@SendTo` | Routes output of a listener to another topic |

---

### What is `@EnableKafka`?

`@EnableKafka` activates the detection of `@KafkaListener` annotations on Spring beans. It must be present on a `@Configuration` class.

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    // Bean definitions
}
```

In Spring Boot, this is auto-configured when `spring-kafka` is on the classpath — you typically don't need to add it manually.

---

### What is `@KafkaListener`?

Marks a method (or class) to listen to messages from one or more Kafka topics.

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void handleOrder(String message) {
    // process message
}
```

**With full record access:**
```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void handleOrder(ConsumerRecord<String, String> record) {
    log.info("Offset: {}, Value: {}", record.offset(), record.value());
}
```

---

### How does `@KafkaListener` handle multiple partitions?

```java
// Listen to specific partitions
@KafkaListener(
    topicPartitions = @TopicPartition(
        topic = "orders",
        partitions = {"0", "1", "2"}
    ),
    groupId = "order-service"
)
public void handleOrder(String message) { }

// Listen to all partitions (default — just specify topic)
@KafkaListener(topics = "orders", groupId = "order-service", concurrency = "3")
public void handleOrder(String message) { }
```

`concurrency` creates multiple consumer threads, each assigned to different partitions.

---

### What is `@KafkaHandler`?

Used inside a class-level `@KafkaListener` to route messages to different methods based on the message payload type.

```java
@KafkaListener(topics = "events", groupId = "event-service")
public class EventListener {

    @KafkaHandler
    public void handleOrder(OrderEvent event) { }

    @KafkaHandler
    public void handlePayment(PaymentEvent event) { }

    @KafkaHandler(isDefault = true)
    public void handleUnknown(Object unknown) { }
}
```

---

### How do you configure consumer groups using annotations?

```java
@KafkaListener(topics = "orders", groupId = "order-processor")
public void process(String message) { }
```

Or via `application.yml` as default:
```yaml
spring:
  kafka:
    consumer:
      group-id: order-processor
```

---

### How do you configure manual acknowledgment?

```yaml
spring:
  kafka:
    listener:
      ack-mode: MANUAL_IMMEDIATE
```

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void handleOrder(String message, Acknowledgment ack) {
    try {
        processOrder(message);
        ack.acknowledge();
    } catch (Exception e) {
        // Don't acknowledge — message will be redelivered
    }
}
```

---

### What is `@RetryableTopic`?

Configures **non-blocking retry** using retry topics. Instead of blocking the consumer thread, failed messages are routed to retry topics with increasing delays, then to a DLT if all retries fail.

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0),
    dltTopicSuffix = "-dlt"
)
@KafkaListener(topics = "orders", groupId = "order-service")
public void handleOrder(String message) {
    // If this throws, message goes to orders-retry-0, orders-retry-1, etc.
}
```

This creates topics: `orders-retry-0`, `orders-retry-1`, `orders-retry-2`, `orders-dlt`

---

### How do you configure retry backoff policies?

```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(
        delay = 500,          // Initial delay in ms
        multiplier = 2.0,     // Exponential multiplier
        maxDelay = 30000      // Cap at 30 seconds
    )
)
@KafkaListener(topics = "payments")
public void handlePayment(String message) { }
```

**Fixed delay:**
```java
@Backoff(delay = 2000)  // Always 2 seconds between retries
```

---

### How does Dead Letter Topic (DLT) work with annotations?

When all retry attempts are exhausted, the message is sent to the DLT.

```java
@RetryableTopic(attempts = "3", dltTopicSuffix = "-dlt")
@KafkaListener(topics = "orders")
public void handleOrder(String message) {
    throw new RuntimeException("Processing failed");
}

@DltHandler
public void handleDlt(String message, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.error("Message sent to DLT from topic: {}, message: {}", topic, message);
    // Alert, store for manual review, etc.
}
```

---

### How do you support batch processing using annotations?

```yaml
spring:
  kafka:
    listener:
      type: BATCH
```

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void handleOrders(List<String> messages) {
    messages.forEach(this::processOrder);
}

// With full record access
@KafkaListener(topics = "orders")
public void handleOrders(List<ConsumerRecord<String, String>> records) {
    records.forEach(r -> processOrder(r.value()));
}
```
