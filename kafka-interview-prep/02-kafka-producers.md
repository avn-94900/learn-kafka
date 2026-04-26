# 2. Kafka Producers

---

### What are producer acknowledgment (ACKS) modes?

| ACK Setting | Behavior | Durability | Performance |
|-------------|----------|------------|-------------|
| `acks=0` | No acknowledgment waited | Lowest — fire and forget | Highest throughput |
| `acks=1` | Leader acknowledges write | Medium — leader may fail before replication | Medium |
| `acks=all` / `acks=-1` | All in-sync replicas acknowledge | Highest — no data loss if ISR ≥ min.insync.replicas | Lowest throughput |

---

### Explain Kafka ACKS and retries mechanism.

**ACK flow:**
1. Producer sends a batch to the partition leader
2. Leader writes to its local log
3. Depending on `acks`, leader waits for follower acknowledgments before responding
4. Producer receives success or error response

**Retry configuration:**
```properties
retries=2147483647
retry.backoff.ms=100
request.timeout.ms=30000
delivery.timeout.ms=120000
max.in.flight.requests.per.connection=5
```

**Retry behavior:**
- **Retriable errors:** Network timeouts, `LeaderNotAvailableException`
- **Non-retriable errors:** `InvalidTopicException`, message too large
- **Ordering with retries:** Set `max.in.flight.requests.per.connection=1` for strict ordering, or use `enable.idempotence=true` (allows up to 5 in-flight)

---

### What happens if a Kafka producer fails?

- **Before send:** Message never leaves the producer buffer — no impact on Kafka
- **During send (no ack received):** Producer retries based on `retries` and `delivery.timeout.ms`
- **After send (ack received):** Message is safely in Kafka; producer failure has no effect
- **After send (no ack, retries exhausted):** `TimeoutException` or `KafkaException` thrown; message may or may not be in Kafka (risk of duplicate if it was written but ack lost)

**Mitigation:** Use `enable.idempotence=true` to safely retry without duplicates.

---

### What is idempotency in Kafka producers?

Idempotency ensures that retrying a failed send does **not** produce duplicate messages, even if the original send actually succeeded on the broker.

**How it works:**
- Broker assigns each producer a unique **Producer ID (PID)**
- Producer attaches a **sequence number** to every message
- Broker deduplicates messages with the same PID + sequence number
- Works within a single producer session (PID resets on restart unless using transactions)

**Configuration:**
```properties
enable.idempotence=true
acks=all
retries=2147483647
max.in.flight.requests.per.connection=5
```

---

### Why is idempotency important?

- Eliminates duplicate messages caused by producer retries
- Allows setting `retries` to a high value safely
- Provides exactly-once producer semantics without application-level deduplication
- Required foundation for Kafka transactions (exactly-once end-to-end)

---

### How do you implement a custom serializer and deserializer?

**Custom Serializer:**
```java
public class OrderSerializer implements Serializer<Order> {
    private final ObjectMapper mapper = new ObjectMapper();

    @Override
    public byte[] serialize(String topic, Order data) {
        if (data == null) return null;
        try {
            return mapper.writeValueAsBytes(data);
        } catch (JsonProcessingException e) {
            throw new SerializationException("Error serializing Order", e);
        }
    }
}
```

**Custom Deserializer:**
```java
public class OrderDeserializer implements Deserializer<Order> {
    private final ObjectMapper mapper = new ObjectMapper();

    @Override
    public Order deserialize(String topic, byte[] data) {
        if (data == null) return null;
        try {
            return mapper.readValue(data, Order.class);
        } catch (IOException e) {
            throw new SerializationException("Error deserializing Order", e);
        }
    }
}
```

**Producer config:**
```properties
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=com.example.OrderSerializer
```

**Consumer config:**
```properties
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=com.example.OrderDeserializer
```
