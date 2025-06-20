## Integration with Java Ecosystem

23. **How would you integrate Spring Boot with Kafka?**
    - Use spring-kafka dependency
    - Configure properties in application.properties/yml
    - Use @KafkaListener annotation for consumers
    - Use KafkaTemplate for producers
    ```java
    @KafkaListener(topics = "my-topic", groupId = "my-group")
    public void listen(String message) {
        // Process message
    }
    ```

24. **How would you implement error handling in Kafka consumers?**
    - Use error handler in consumer factory
    - Implement dead letter topics for failed messages
    - Retry mechanisms with exponential backoff
    - Circuit breaker patterns for downstream service failures

25. **How would you implement transactions in Kafka with Java?**
    ```java
    producer.initTransactions();
    try {
        producer.beginTransaction();
        // Send records
        producer.send(record1);
        producer.send(record2);
        // Commit transaction
        producer.commitTransaction();
    } catch (Exception e) {
        producer.abortTransaction();
        throw e;
    }
    ```

26. **How would you test Kafka applications in Java?**
    - Use embedded Kafka for unit tests
    - TestContainers for integration tests
    - Mock producers/consumers for isolated testing
    - Create test consumer to verify produced messages

27. **Explain how you would implement an event-driven architecture using Kafka in a microservices environment.**
    - Domain events published to Kafka topics
    - Services produce and consume events as needed
    - Use event sourcing patterns where appropriate
    - Implement CQRS for read/write separation
    - Handle eventual consistency challenges


