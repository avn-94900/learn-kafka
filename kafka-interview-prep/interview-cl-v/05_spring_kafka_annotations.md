# Spring Boot Kafka Annotations — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Q1. What does `@EnableKafka` do and where does it go?

- Enables detection of `@KafkaListener` annotations on Spring beans
- Place on a `@Configuration` class
- When using Spring Boot with `spring-kafka` on classpath → **auto-configured** (not required to add manually in most cases)
- Required if you extend `KafkaListenerConfigurer` manually or use a non-Boot setup

---

## Q2. How does `@KafkaListener` handle multiple partitions and concurrent consumers?

```java
// Listen to specific partitions
@KafkaListener(topicPartitions = @TopicPartition(
    topic = "orders",
    partitions = {"0", "1", "2"}
))

// Concurrency: spawns N listener threads (each gets one or more partitions)
@KafkaListener(topics = "orders", concurrency = "3")
```

- `concurrency` creates N `KafkaMessageListenerContainer` threads, each acting as a separate consumer in the group
- Total consumers = `concurrency` value; partitions are distributed among them
- Setting `concurrency > partitionCount` → idle threads

---

## Q3. What is `@KafkaHandler` and how does it differ from `@KafkaListener`?

```java
@KafkaListener(topics = "events")
@Component
public class EventListener {

    @KafkaHandler
    public void handleOrder(OrderEvent event) { ... }

    @KafkaHandler
    public void handlePayment(PaymentEvent event) { ... }

    @KafkaHandler(isDefault = true)
    public void handleUnknown(Object unknown) { ... }
}
```

- `@KafkaListener` on the class — one consumer for the topic
- `@KafkaHandler` on methods — routes to the correct method based on **payload type**
- Requires a deserializer that returns the correct type (e.g., `JsonDeserializer` with type mapping)

---

## Q4. How do you configure manual acknowledgment via annotations?

```java
// Factory config:
factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

// Listener:
@KafkaListener(topics = "payments")
public void listen(ConsumerRecord<String, Payment> record, Acknowledgment ack) {
    try {
        processPayment(record.value());
        ack.acknowledge();
    } catch (Exception e) {
        // don't ack → message will be reprocessed
    }
}
```

---

## Q5. What is `@RetryableTopic`? How does it work under the hood?

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0),
    dltTopicSuffix = "-dlt"
)
@KafkaListener(topics = "orders")
public void listen(Order order) { ... }
```

- On failure, routes message to **retry topics** (e.g., `orders-retry-0`, `orders-retry-1000`, `orders-retry-2000`)
- Each retry topic has a **header** with retry delay — consumer sleeps until the delay expires
- After exhausting all retries → routes to **DLT** (`orders-dlt`)
- Non-blocking retries: main topic consumer is not paused during retry wait
- Uses separate `@KafkaListener` internally for each retry topic

---

## Q6. How does Dead Letter Topic (DLT) work? When should you send to DLT vs retry?

**DLT:** A topic where unprocessable messages land after all retries are exhausted.

```java
@RetryableTopic(attempts = "3", dltStrategy = DltStrategy.FAIL_ON_ERROR)
@KafkaListener(topics = "orders")
public void listen(Order order) { ... }

// Handle DLT messages:
@DltHandler
public void handleDlt(Order order, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.error("DLT message from {}: {}", topic, order);
    // alert, persist to DB, manual review queue
}
```

**Retry vs DLT decision:**

| Retry | Send to DLT immediately |
|---|---|
| Transient failures (DB timeout, downstream API down) | Deserialization failure |
| Recoverable errors | Business rule violation (invalid state) |
| Known transient exceptions | Poison pill messages |

---

## Q7. How do you configure batch processing with `@KafkaListener`?

```java
// Factory config:
factory.setBatchListener(true);

// Listener:
@KafkaListener(topics = "orders")
public void listenBatch(List<ConsumerRecord<String, Order>> records, Acknowledgment ack) {
    records.forEach(r -> process(r.value()));
    ack.acknowledge();
}
```

- Receives all records from a single `poll()` call as a `List`
- Error handling is more complex — one failure affects the entire batch
- Pair with `BatchToRecordErrorHandler` or `DefaultBatchErrorHandler` for fine-grained control

---

## Q8. How do you configure retry backoff policies? What are the options?

```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(
        delay = 500,          // initial delay in ms
        multiplier = 2.0,     // exponential multiplier
        maxDelay = 30000,     // cap at 30s
        random = true         // jitter to avoid thundering herd
    )
)
```

**Backoff strategies:**
- **Fixed:** `@Backoff(delay = 1000)` — same wait each time
- **Exponential:** `multiplier > 1` — delay doubles each attempt
- **Exponential with jitter:** `random = true` — adds randomness to prevent synchronized retries across consumers
