# Kafka Error Handling & Testing — Interview Q&A

> **Target:** 1 year Kafka experience · Focuses on depth, not definitions

---

## Error Handling

### Q1. What are the common Kafka error scenarios and how are they handled?

| Scenario | What Happens | Handling |
|---|---|---|
| Deserialization failure | Exception before listener runs | `ErrorHandlingDeserializer` + DLT |
| Processing exception | Listener throws | Retry → DLT via `@RetryableTopic` |
| Consumer lag spike | Slow processing or downstream failure | Scale consumers, async processing |
| Broker failure | Partition leader changes | Producer/consumer retry with backoff |
| Offset commit failure | Duplicate processing on restart | Idempotent processing |
| Network partition | Timeouts | Exponential backoff, circuit breaker |

---

### Q2. What is a poison message? How do you handle it?

- A message the consumer **cannot process** regardless of retry count (bad data, schema mismatch, corrupt bytes)
- Retrying forever causes the consumer to **stall** — no other messages in that partition progress
- Solutions:
  - Route to DLT after N retries using `@RetryableTopic`
  - Use `ErrorHandlingDeserializer2` to catch deserialization errors and forward to DLT without blocking
  - Alert + manual inspection of DLT

---

### Q3. What happens on deserialization failure? How do you prevent it from blocking a partition?

**Without protection:**
- `ClassCastException` or `SerializationException` in the consumer → infinite retry → partition stalled

**With `ErrorHandlingDeserializer`:**
```java
consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
consumerProps.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
```
- Wraps the real deserializer; on failure, sets a header and passes `null` to the listener
- Combine with `@RetryableTopic(exclude = DeserializationException.class, dltStrategy = DltStrategy.ALWAYS_RETRY_ON_ERROR)` to bypass retries and go straight to DLT

---

### Q4. How do you avoid duplicate processing?

Three layers:
1. **Idempotent producer** — prevents duplicate writes to Kafka
2. **Manual offset commit** — commit only after successful processing
3. **Idempotent consumer** — make your processing logic safe for re-runs:
   - Use **idempotency keys** (message ID / event ID) with DB upsert or `INSERT ... ON CONFLICT DO NOTHING`
   - Check processed state before acting (dedupe table / Redis cache)

---

### Q5. When should you retry vs send directly to DLT?

```
Retry:
  - Transient: DB timeout, downstream HTTP 503, lock contention
  - Expected to succeed within seconds/minutes

DLT immediately:
  - Deserialization failure (data is broken, retrying is pointless)
  - Business validation failure (invalid state machine transition)
  - Non-recoverable external dependency error

Never retry:
  - Poison messages (causes infinite loop without DLT)
```

In Spring Kafka: use `include`/`exclude` on `@RetryableTopic` to control which exceptions retry.

---

## Testing

### Q6. How do you test Kafka producers and consumers with Embedded Kafka?

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"orders", "orders-dlt"},
    brokerProperties = {"auto.create.topics.enable=true"}
)
class OrderConsumerTest {

    @Autowired
    private KafkaTemplate<String, Order> kafkaTemplate;

    @Test
    void shouldConsumeAndProcess() throws Exception {
        kafkaTemplate.send("orders", new Order("123", "PAID"));

        // Use CountDownLatch or Awaitility to wait for async consumption
        await().atMost(5, SECONDS).until(() -> orderRepository.findById("123").isPresent());
    }
}
```

- `@EmbeddedKafka` starts an in-memory Kafka broker — no Docker needed
- Inject `@Value("${spring.embedded.kafka.brokers}")` for bootstrap server config

---

### Q7. How do you test DLT behavior?

```java
@Test
void shouldSendToDltAfterRetryExhaustion() throws Exception {
    // Send a message that will fail processing (e.g., mock throws exception)
    kafkaTemplate.send("orders", new Order("BAD_ID", null));

    // Listen to DLT topic
    ConsumerRecord<String, Order> dltRecord = KafkaTestUtils.getSingleRecord(dltConsumer, "orders-dlt");

    assertThat(dltRecord).isNotNull();
    assertThat(dltRecord.value().getId()).isEqualTo("BAD_ID");
}
```

- Create a test consumer pointing at the DLT topic
- Use `KafkaTestUtils.getSingleRecord()` with a timeout
- Verify the DLT message has the expected headers (retry count, original topic, exception message)

---

### Q8. How do you validate retry behavior in tests?

```java
// Mock the service to fail N times then succeed
when(orderService.process(any()))
    .thenThrow(new RuntimeException("DB down"))
    .thenThrow(new RuntimeException("DB down"))
    .thenReturn(true); // succeeds on 3rd attempt

// Verify retry count via invocation count
verify(orderService, times(3)).process(any());
```

- Use `Mockito.when().thenThrow().thenReturn()` to simulate transient failures
- Combine with `Awaitility` for async verification

---

### Q9. Unit testing vs Integration testing for Kafka — when to use which?

| | Unit Test | Integration Test |
|---|---|---|
| Scope | Single listener/service method | Full pipeline with Embedded Kafka |
| Speed | Fast (milliseconds) | Slower (seconds) |
| What to test | Business logic, exception routing decisions | Offset commits, retry flow, DLT routing, serialization |
| Tools | Mockito, no Kafka | `@EmbeddedKafka`, `spring-kafka-test` |

**Rule of thumb:** Unit test the processing logic; integration test the Kafka wiring (serialization, error routing, retry counts, offset behavior).
