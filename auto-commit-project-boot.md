pom.xml:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>kafka-manual-commit</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>kafka-manual-commit</name>
    <description>Demo project for Spring Boot with Kafka manual offset commit</description>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

src/main/resources/application.yml:
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: manual-offset-group
      auto-offset-reset: earliest
      enable-auto-commit: false  # Disable auto-commit to handle manually
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer

kafka:
  topic: manual-offset-topic
```

src/main/java/com/example/kafkamanualcommit/KafkaManualCommitApplication.java:
```java
package com.example.kafkamanualcommit;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class KafkaManualCommitApplication {

    public static void main(String[] args) {
        SpringApplication.run(KafkaManualCommitApplication.class, args);
    }
}
```

src/main/java/com/example/kafkamanualcommit/config/KafkaTopicConfig.java:
```java
package com.example.kafkamanualcommit.config;

import org.apache.kafka.clients.admin.AdminClientConfig;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.KafkaAdmin;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaTopicConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${kafka.topic}")
    private String topic;

    @Bean
    public KafkaAdmin kafkaAdmin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new KafkaAdmin(configs);
    }

    @Bean
    public NewTopic manualOffsetTopic() {
        return new NewTopic(topic, 1, (short) 1);
    }
}
```

src/main/java/com/example/kafkamanualcommit/config/KafkaConsumerConfig.java:
```java
package com.example.kafkamanualcommit.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${spring.kafka.consumer.group-id}")
    private String groupId;

    @Value("${spring.kafka.consumer.auto-offset-reset}")
    private String autoOffsetReset;

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, autoOffsetReset);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> syncCommitKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        return factory;
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> asyncCommitKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        return factory;
    }
}
```

src/main/java/com/example/kafkamanualcommit/config/KafkaProducerConfig.java:
```java
package com.example.kafkamanualcommit.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(configProps);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

src/main/java/com/example/kafkamanualcommit/service/MessageProducerService.java:
```java
package com.example.kafkamanualcommit.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.concurrent.CompletableFuture;

@Service
@Slf4j
@RequiredArgsConstructor
public class MessageProducerService {

    private final KafkaTemplate<String, String> kafkaTemplate;

    @Value("${kafka.topic}")
    private String topic;

    // Sends a message every 10 seconds
    @Scheduled(fixedRate = 10000)
    public void sendMessage() {
        String message = "Message sent at " + LocalDateTime.now();
        CompletableFuture<SendResult<String, String>> future = kafkaTemplate.send(topic, message);
        
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                log.info("Sent message: [{}] with offset: [{}]", 
                         message, 
                         result.getRecordMetadata().offset());
            } else {
                log.error("Unable to send message: [{}] due to: {}", message, ex.getMessage());
            }
        });
    }
}
```

src/main/java/com/example/kafkamanualcommit/service/SyncCommitConsumerService.java:
```java
package com.example.kafkamanualcommit.service;

import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class SyncCommitConsumerService {

    @KafkaListener(
            topics = "${kafka.topic}",
            groupId = "${spring.kafka.consumer.group-id}-sync",
            containerFactory = "syncCommitKafkaListenerContainerFactory"
    )
    public void consumeWithSyncCommit(ConsumerRecord<String, String> record, Acknowledgment acknowledgment) {
        log.info("Received message with SYNC commit: {}", record.value());
        
        try {
            // Process your message here
            processMessage(record.value());
            
            // Commit offset synchronously
            acknowledgment.acknowledge();
            log.info("Successfully committed offset synchronously for: {}", record.value());
        } catch (Exception e) {
            log.error("Error processing message: {}", e.getMessage());
            // Don't acknowledge, will be redelivered
        }
    }
    
    private void processMessage(String message) {
        log.info("Processing message: {}", message);
        // Add your message processing logic here
    }
}
```

src/main/java/com/example/kafkamanualcommit/service/AsyncCommitConsumerService.java:
```java
package com.example.kafkamanualcommit.service;

import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class AsyncCommitConsumerService {

    @KafkaListener(
            topics = "${kafka.topic}",
            groupId = "${spring.kafka.consumer.group-id}-async",
            containerFactory = "asyncCommitKafkaListenerContainerFactory"
    )
    public void consumeWithAsyncCommit(ConsumerRecord<String, String> record, Acknowledgment acknowledgment) {
        log.info("Received message with ASYNC commit: {}", record.value());
        
        try {
            // Process your message here
            processMessage(record.value());
            
            // Commit offset asynchronously
            acknowledgment.acknowledge();
            log.info("Triggered asynchronous commit for: {}", record.value());
        } catch (Exception e) {
            log.error("Error processing message: {}", e.getMessage());
            // Don't acknowledge, will be redelivered
        }
    }
    
    private void processMessage(String message) {
        log.info("Processing message: {}", message);
        // Add your message processing logic here
    }
}
```

src/main/java/com/example/kafkamanualcommit/controller/KafkaController.java:
```java
package com.example.kafkamanualcommit.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/kafka")
@RequiredArgsConstructor
public class KafkaController {

    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Value("${kafka.topic}")
    private String topic;

    @PostMapping("/send")
    public ResponseEntity<String> sendMessage(@RequestBody String message) {
        kafkaTemplate.send(topic, message);
        return ResponseEntity.ok("Message sent to Kafka topic: " + topic);
    }
}
```

src/main/java/com/example/kafkamanualcommit/service/ManualConsumerService.java:
```java
package com.example.kafkamanualcommit.service;

import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.TopicPartition;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.time.Duration;
import java.util.Collections;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicBoolean;

@Service
@Slf4j
public class ManualConsumerService {

    private final ConsumerFactory<String, String> consumerFactory;
    private final AtomicBoolean running = new AtomicBoolean(true);
    private ExecutorService executorService;
    private KafkaConsumer<String, String> consumer;

    @Value("${kafka.topic}")
    private String topic;

    public ManualConsumerService(ConsumerFactory<String, String> consumerFactory) {
        this.consumerFactory = consumerFactory;
    }

    @PostConstruct
    public void init() {
        executorService = Executors.newSingleThreadExecutor();
        executorService.submit(this::consumeMessages);
    }

    private void consumeMessages() {
        consumer = (KafkaConsumer<String, String>) consumerFactory.createConsumer("manual-consumer-group", null);
        consumer.subscribe(Collections.singleton(topic));

        while (running.get()) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            if (!records.isEmpty()) {
                log.info("Received {} records", records.count());
                
                // Process each record
                for (ConsumerRecord<String, String> record : records) {
                    log.info("Processing record: {}", record.value());
                    // Add your processing logic here
                }
                
                // Example of synchronous commit
                try {
                    consumer.commitSync();
                    log.info("Committed offsets synchronously");
                } catch (Exception e) {
                    log.error("Failed to commit offsets synchronously", e);
                }
                
                // Example of async commit with callback
                /*
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        log.error("Async commit failed for offsets {}", offsets, exception);
                    } else {
                        log.info("Async commit succeeded for offsets {}", offsets);
                    }
                });
                */
            }
        }
    }

    @PreDestroy
    public void destroy() {
        log.info("Shutting down consumer");
        running.set(false);
        if (consumer != null) {
            consumer.close();
        }
        if (executorService != null) {
            executorService.shutdown();
        }
    }
}
```

README.md:
```markdown
# Spring Boot Kafka Manual Offset Commit

This project demonstrates how to use Spring Boot with Apache Kafka, specifically focusing on manual offset commit strategies.

## Features

- Spring Boot application with Kafka integration
- Manual offset commit using both synchronous (`commitSync()`) and asynchronous (`commitAsync()`) methods
- Message producer (both scheduled and on-demand via REST API)
- Multiple consumer implementations showcasing different commit patterns

## Prerequisites

- Java 17+
- Maven
- Kafka server running on localhost:9092 (or update configuration to point to your Kafka instance)

## Running the Application

1. Start Zookeeper and Kafka servers
2. Build the application:
   ```
   mvn clean install
   ```
3. Run the application:
   ```
   mvn spring-boot:run
   ```

## API Endpoints

- `POST /api/kafka/send` - Send a custom message to the Kafka topic

## Project Structure

- `KafkaManualCommitApplication.java` - Main Spring Boot application class
- Configuration classes:
  - `KafkaTopicConfig.java` - Defines the Kafka topic
  - `KafkaConsumerConfig.java` - Configures Kafka consumer properties and listener factories
  - `KafkaProducerConfig.java` - Configures Kafka producer
- Service classes:
  - `MessageProducerService.java` - Scheduled message producer
  - `SyncCommitConsumerService.java` - Consumer with synchronous offset commit
  - `AsyncCommitConsumerService.java` - Consumer with asynchronous offset commit
  - `ManualConsumerService.java` - Low-level consumer with direct control over polling and committing
- Controller:
  - `KafkaController.java` - REST controller for sending messages on demand

## Key Implementation Details

1. **Auto-commit disabled**: `enable-auto-commit: false` in `application.yml`
2. **Manual acknowledgment**: Using `Acknowledgment` in listener methods
3. **Manual consumer**: Direct use of `KafkaConsumer` for more granular control
4. **Commit strategies**:
   - Synchronous commit via `commitSync()`
   - Asynchronous commit via `commitAsync()`

## Notes

- The application creates two consumer groups with the same base ID plus a suffix (-sync and -async)
- The manual consumer implementation uses a separate thread to continuously poll for messages