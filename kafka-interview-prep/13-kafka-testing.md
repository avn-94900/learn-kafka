# 13. Kafka Testing

---

### How do you test Kafka producers?

**Unit test — mock KafkaTemplate:**
```java
@ExtendWith(MockitoExtension.class)
class OrderProducerTest {

    @Mock
    private KafkaTemplate<String, String> kafkaTemplate;

    @InjectMocks
    private OrderProducer producer;

    @Test
    void shouldSendMessage() {
        when(kafkaTemplate.send(anyString(), anyString(), anyString()))
            .thenReturn(mock(CompletableFuture.class));

        producer.sendOrder("order-123", "ORDER_CREATED");

        verify(kafkaTemplate).send("orders", "order-123", "ORDER_CREATED");
    }
}
```

**Integration test — EmbeddedKafka:**
```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"})
class OrderProducerIntegrationTest {

    @Autowired
    private OrderProducer producer;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    void shouldPublishToKafka() throws Exception {
        producer.sendOrder("order-123", "ORDER_CREATED");
        // Verify with a test consumer
    }
}
```

---

### How do you test Kafka consumers?

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"})
class OrderConsumerIntegrationTest {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private OrderConsumer consumer;  // Has a CountDownLatch or spy

    @Test
    void shouldConsumeMessage() throws Exception {
        kafkaTemplate.send("orders", "order-123", "ORDER_CREATED");

        // Wait for consumer to process
        assertTrue(consumer.getLatch().await(10, TimeUnit.SECONDS));
        assertEquals("ORDER_CREATED", consumer.getLastMessage());
    }
}
```

---

### What is Embedded Kafka?

**Embedded Kafka** is an in-process Kafka broker that runs inside your test JVM — no external Kafka installation needed.

Provided by `spring-kafka-test` via `@EmbeddedKafka` annotation.

**Benefits:**
- Fast test startup
- No external dependencies
- Isolated per test class
- Supports full Kafka functionality (topics, partitions, consumer groups)

---

### What is `spring-kafka-test`?

`spring-kafka-test` is a test support library for Spring Kafka that provides:
- `@EmbeddedKafka` — Starts an embedded Kafka broker for tests
- `KafkaTestUtils` — Utility methods for consumer/producer setup in tests
- `ConsumerRecord` matchers for assertions

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>
```

**Setup example:**
```java
@EmbeddedKafka(
    partitions = 3,
    topics = {"orders", "payments"},
    brokerProperties = {"log.dir=target/embedded-kafka"}
)
```

---

### How do you simulate message failures?

```java
@SpyBean
private OrderService orderService;

@Test
void shouldRetryOnFailure() throws Exception {
    // Make processing fail twice, then succeed
    doThrow(new RuntimeException("Transient error"))
        .doThrow(new RuntimeException("Transient error"))
        .doNothing()
        .when(orderService).process(any());

    kafkaTemplate.send("orders", "test-message");

    // Verify it was called 3 times (2 failures + 1 success)
    verify(orderService, timeout(10000).times(3)).process(any());
}
```

---

### How do you validate retry behavior?

```java
@RetryableTopic(attempts = "3", backoff = @Backoff(delay = 100))
@KafkaListener(topics = "orders")
public void handle(String message) {
    orderService.process(message);
}

@Test
void shouldRetryThreeTimes() throws Exception {
    doThrow(new RuntimeException()).when(orderService).process(any());

    kafkaTemplate.send("orders", "bad-message");

    // Verify 3 attempts total (1 original + 2 retries)
    verify(orderService, timeout(5000).times(3)).process(any());
}
```

---

### How do you test Dead Letter Topics?

```java
@Test
void shouldSendToDltAfterMaxRetries() throws Exception {
    doThrow(new RuntimeException("Always fails")).when(orderService).process(any());

    kafkaTemplate.send("orders", "poison-message");

    // Consume from DLT and verify
    ConsumerRecord<String, String> dltRecord = KafkaTestUtils.getSingleRecord(
        dltConsumer, "orders-dlt", Duration.ofSeconds(10)
    );

    assertNotNull(dltRecord);
    assertEquals("poison-message", dltRecord.value());
}
```

---

### What is the difference between unit testing and integration testing?

| Aspect | Unit Testing | Integration Testing |
|--------|-------------|---------------------|
| **Scope** | Single class/method in isolation | Multiple components working together |
| **Kafka** | Mocked (`KafkaTemplate` mock) | Real (EmbeddedKafka or TestContainers) |
| **Speed** | Very fast (milliseconds) | Slower (seconds — broker startup) |
| **Dependencies** | All mocked | Real Spring context, real Kafka |
| **Purpose** | Test business logic | Test end-to-end message flow |
| **Reliability** | High (no flakiness) | Can be flaky (timing, async) |

**Best practice:** Use unit tests for business logic, integration tests for verifying the full Kafka pipeline (produce → consume → process → commit).

**TestContainers alternative (real Kafka in Docker):**
```java
@Testcontainers
class KafkaContainerTest {
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
}
```
