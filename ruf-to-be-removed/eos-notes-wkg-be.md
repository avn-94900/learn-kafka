
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
