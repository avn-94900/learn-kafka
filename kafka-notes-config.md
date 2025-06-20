
##  Kafka's message retention policy and configuration
###  Valid Combined Configuration (no error):

```properties
log.retention.ms=604800000       # Retain messages for 7 days
log.retention.bytes=1073741824   # Retain up to 1 GB of log data
log.cleanup.policy=delete        # Use time/size-based deletion (default)
```

---

### How Kafka Applies These Together:

Kafka uses **both time and size** thresholds:

* If **retention.ms** is reached → Kafka starts deleting old log segments.
* If **retention.bytes** is reached → Kafka deletes oldest segments to keep size under the limit.
* If **both are configured**, **whichever condition is met first** will trigger cleanup.

> This is totally safe and **recommended for production** to control disk usage and data lifecycle.

---

###  Notes:

* These apply to **all topics** unless overridden with per-topic configs.
* You must **restart the Kafka broker** after changing `server.properties` for the changes to take effect.

---

## Kafka server Backpressure Handling:


### 🔹 What This Means:

```properties
buffer.memory=67108864       # 64 MB buffer in the producer
max.block.ms=30000           # Wait up to 30 seconds if buffer is full
```

---

### 🔄 Step-by-step Behavior:

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

---

###  Example Exception:

```java
org.apache.kafka.common.errors.TimeoutException: 
Failed to allocate memory within the configured max blocking time 30000 ms
```

### 🔁 Kafka Handles It Like This:

```text
[Producer] --> [Buffer (buffer.memory)] --> [Kafka Broker]

               ^         ↑            ↓
               |         |            |
         (if full)   wait (max.block.ms)   then send to broker
```

---

###  Summary:

| Setting                       | Purpose                                       |
| ----------------------------- | --------------------------------------------- |
| `buffer.memory`               | How much data the producer can buffer locally |
| `max.block.ms`                | How long to wait when buffer is full          |
| If buffer full + time exceeds | ➡️ Throw error (backpressure response)        |

---






##  Kafka Consumer Lag Management  :

* Producer wrote messages with offsets: `100 to 200`
* Consumer has only read up to offset `150`

  **Consumer Lag = 200 - 150 = 50 messages**

---

#### 🔧 How to Manage Consumer Lag in Kafka

---

###  1. **Monitor Lag**

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

---

###  2. **Tune Consumer Settings**

Key configs:

| Property               | Description                                           |
| ---------------------- | ----------------------------------------------------- |
| `max.poll.records`     | Controls how many messages to fetch per poll.         |
| `max.poll.interval.ms` | If processing takes too long, this must be increased. |
| `fetch.max.bytes`      | Max bytes fetched per request.                        |
| `session.timeout.ms`   | Time after which Kafka marks consumer as dead.        |

---

###  3. **Scale Consumers**

* If lag is consistently growing, add more consumer **instances** to the **consumer group**.
* Each consumer in the group handles **one or more partitions**, so adding more reduces the load per consumer.

---

###  4. **Optimize Processing Logic**

* If a consumer is slow due to business logic:

  * Offload to **async processing**
  * Use **worker threads or a thread pool**
  * Avoid long blocking calls

---

###  5. **Increase Topic Retention**

If you expect consumers to lag:

```properties
log.retention.ms=604800000   # Retain data for 7 days
```

This ensures that **messages are not deleted** before slow consumers read them.

---

##  Why You Should Care?

If consumer lag is high:

* Real-time applications may become **outdated**
* Risk of **data loss** (if retention time is exceeded)
* You’ll have **high disk usage** on Kafka brokers

---














## **message delivery semantics in Kafka** — `at-most-once`, `at-least-once`, and `exactly-once` :
---

##  1. **At-Most-Once (Fastest, May Lose Messages)**

> 📝 Messages are delivered **zero or one time**. If failure happens after send, **message is lost**.

###  Producer Config (`application.yml`)

```yaml
spring:
  kafka:
    producer:
      acks: 0                          # No acknowledgment from broker (fire-and-forget)
      retries: 0                       # No retries if send fails
      enable-idempotence: false        # No deduplication logic
```

###  Consumer Config

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: true         # Offsets auto-committed (even before processing)
      auto-commit-interval: 1000       # Commit offset every 1 sec (default)
```

---

##  2. **At-Least-Once (Reliable, May Duplicate)**

> 📝 Messages are delivered **one or more times**. Duplicates can occur due to retries.

###  Producer Config

```yaml
spring:
  kafka:
    producer:
      acks: all                        # Wait for all in-sync replicas to acknowledge
      retries: 5                       # Retry sending if broker fails
      enable-idempotence: false        # No deduplication; may cause duplicates
```

###  Consumer Config

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false        # Prevent auto-offset commit
      auto-offset-reset: earliest      # Start from earliest if no offset found
```

> ⚠️ You must **manually commit offsets** after processing, using:

```java
consumer.commitSync();
```

---

##  3. **Exactly-Once (Reliable + No Duplicates or Loss)**

> 📝 Messages are **delivered once and only once**, even with retries and failures.

###  Producer Config

```yaml
spring:
  kafka:
    producer:
      acks: all                                 # Strong durability: wait for all replicas
      retries: 5                                # Retry send on failure
      enable-idempotence: true                  # Enable deduplication for safe retries
      transactional-id-prefix: txn-id           # Enables transactional producer with prefix
```

###  Consumer Config

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

---

##  Summary Table

| Semantics     | Duplicates | Loss  | Config Key Points                                                             |
| ------------- | ---------- | ----- | ----------------------------------------------------------------------------- |
| At-most-once  | ❌ No       | ✅ Yes | `acks: 0`, `enable-idempotence: false`, `auto-commit: true`                   |
| At-least-once | ✅ Yes      | ❌ No  | `acks: all`, `retries: 5`, `enable-idempotence: false`, `manual commit`       |
| Exactly-once  | ❌ No       | ❌ No  | `acks: all`, `enable-idempotence: true`, `transactional-id`, `read_committed` |

---