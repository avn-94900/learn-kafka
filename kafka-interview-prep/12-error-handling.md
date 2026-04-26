# 12. Kafka Error Handling

---

### What are common Kafka error scenarios?

| Scenario | Cause | Impact |
|----------|-------|--------|
| **Consumer lag** | Slow processing, under-provisioned consumers | Delayed processing |
| **Deserialization failure** | Schema mismatch, corrupt message | Consumer crash or skip |
| **Broker unavailable** | Network issue, broker crash | Producer/consumer errors |
| **Rebalance storm** | Frequent consumer joins/leaves | Processing pauses |
| **Poison message** | Malformed data that always fails processing | Consumer stuck in retry loop |
| **Offset commit failure** | Network issue during commit | Duplicate processing on restart |
| **Disk full on broker** | Insufficient retention/storage config | Write failures |

---

### What happens when message processing fails?

**Without error handling:** Exception propagates, consumer thread may crash, partition unprocessed.

**With Spring Kafka default behavior:**
- Retries the same message up to `maxAttempts` times
- After exhausting retries, sends to DLT (if configured) or logs and skips

**Recommended pattern:**
```java
@KafkaListener(topics = "orders")
public void handle(String message) {
    try {
        processOrder(message);
    } catch (RecoverableException e) {
        throw e;  // Let retry mechanism handle it
    } catch (NonRecoverableException e) {
        log.error("Non-recoverable error, skipping message: {}", message, e);
        // Optionally send to DLT manually
    }
}
```

---

### What is a Dead Letter Topic (DLT)?

A **Dead Letter Topic** is a special Kafka topic where messages are sent after all retry attempts are exhausted. It acts as a quarantine for problematic messages.

**Purpose:**
- Prevents poison messages from blocking the main consumer
- Preserves failed messages for investigation and reprocessing
- Decouples error handling from main processing flow

**Naming convention:** `<original-topic>-dlt` (e.g., `orders-dlt`)

**DLT message headers include:**
- Original topic, partition, offset
- Exception class and message
- Retry count

---

### When should messages be retried vs sent to DLT?

**Retry (transient errors):**
- Network timeouts
- Temporary database unavailability
- External service temporarily down
- Optimistic locking failures

**Send to DLT (permanent errors):**
- Deserialization failures (corrupt/incompatible message)
- Business validation failures (invalid data that will always fail)
- Non-existent resource references
- Repeated failures after max retries

```java
@RetryableTopic(
    attempts = "4",
    exclude = {DeserializationException.class, ValidationException.class}  // Skip retry, go straight to DLT
)
@KafkaListener(topics = "orders")
public void handle(String message) { }
```

---

### What are poison messages?

A **poison message** is a message that consistently causes processing failures, causing the consumer to get stuck in an infinite retry loop.

**Causes:**
- Malformed or corrupt data
- Schema incompatibility
- Business logic that always throws for specific data

**Handling:**
- Use `@RetryableTopic` with a max attempts limit
- Route to DLT after max retries
- Use `DefaultErrorHandler` with a `DeadLetterPublishingRecoverer`

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
}
```

---

### What happens if deserialization fails?

By default, a `DeserializationException` is thrown and the consumer may get stuck retrying the same undeserializable message.

**Solution — ErrorHandlingDeserializer:**
```yaml
spring:
  kafka:
    consumer:
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: com.example.OrderDeserializer
```

This wraps the actual deserializer; on failure, it returns a null value and sets an exception header. The listener can then route the message to DLT instead of crashing.

---

### How do you implement retry with backoff?

**Non-blocking retry (recommended) — `@RetryableTopic`:**
```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 16000)
)
@KafkaListener(topics = "orders")
public void handle(String message) { }
```
Retry delays: 1s → 2s → 4s → DLT

**Blocking retry — `DefaultErrorHandler`:**
```java
@Bean
public DefaultErrorHandler errorHandler() {
    return new DefaultErrorHandler(new ExponentialBackOffWithMaxRetries(3));
}
```
Blocks the consumer thread during backoff — not recommended for high-throughput scenarios.

---

### How do you avoid duplicate processing?

**Idempotent consumers:**
- Track processed message IDs in a database or cache
- Check before processing; skip if already processed

```java
public void handle(ConsumerRecord<String, String> record) {
    String messageId = record.headers().lastHeader("messageId").value().toString();
    if (processedIds.contains(messageId)) return;  // Skip duplicate

    processOrder(record.value());
    processedIds.add(messageId);
}
```

**Database-level idempotency:**
- Use `INSERT ... ON CONFLICT DO NOTHING` (PostgreSQL)
- Use unique constraints on business keys

**Kafka Transactions (exactly-once):**
- Use transactional producer + `isolation.level=read_committed` on consumer
- Guarantees exactly-once in Kafka-to-Kafka pipelines
