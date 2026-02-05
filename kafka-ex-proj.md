
## 9. Code Examples and Projects

### Kafka Producer Service

#### POM.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.5</version>
        <relativePath/>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>kafka-producer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>kafka-producer</name>
    <description>Kafka Producer Service</description>
    
    <properties>
        <java.version>11</java.version>
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
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

#### application.yml

```yaml
spring:
  application:
    name: kafka-producer
  kafka:
    producer:
      bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        spring.json.add.type.headers: false

kafka:
  topic:
    employee: employee-topic

server:
  port: 8080
```

#### Main Application

```java
package com.example.kafkaproducer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaProducerApplication {

    public static void main(String[] args) {
        SpringApplication.run(KafkaProducerApplication.class, args);
    }
}
```

#### Kafka Config

```java
package com.example.kafkaproducer.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaConfig {

    @Value("${kafka.topic.employee}")
    private String employeeTopic;

    public String getEmployeeTopic() {
        return employeeTopic;
    }
}
```

#### Employee Model

```java
package com.example.kafkaproducer.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Employee implements Serializable {
    private Long id;
    private String name;
    private String department;
    private Double salary;
}
```

#### Producer Service

```java
package com.example.kafkaproducer.service;

import com.example.kafkaproducer.config.KafkaConfig;
import com.example.kafkaproducer.model.Employee;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class EmployeeProducerService {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final KafkaConfig kafkaConfig;

    public void sendEmployeeMessage(Employee employee) {
        log.info("Sending employee to Kafka: {}", employee);
        kafkaTemplate.send(kafkaConfig.getEmployeeTopic(), String.valueOf(employee.getId()), employee);
        log.info("Employee message sent successfully");
    }
}
```

#### REST Controller

```java
package com.example.kafkaproducer.controller;

import com.example.kafkaproducer.model.Employee;
import com.example.kafkaproducer.service.EmployeeProducerService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
@RequestMapping("/api/employees")
@RequiredArgsConstructor
public class EmployeeController {

    private final EmployeeProducerService producerService;

    @PostMapping
    public ResponseEntity<String> createEmployee(@RequestBody Employee employee) {
        log.info("Received employee creation request: {}", employee);
        producerService.sendEmployeeMessage(employee);
        return ResponseEntity.ok("Employee sent to Kafka successfully");
    }
}
```

### Consumer Service

[Similar structure for consumer]

### Manual Offset Commit Project

#### Understanding Manual Offset Management

This project demonstrates manual offset management in Kafka consumers, providing better control over message processing guarantees.

**Key Differences from Auto-commit:**
- Auto-commit is disabled (`enable-auto-commit: false`)
- Offsets are committed only after successful processing
- Provides at-least-once delivery guarantees
- Better for critical data processing

#### Project Structure

```
kafka-manual-offset/
├── src/
│   ├── main/
│   │   ├── java/com/example/kafka/
│   │   │   ├── KafkaManualOffsetApplication.java
│   │   │   ├── config/KafkaConsumerConfig.java
│   │   │   └── consumer/ManualOffsetKafkaConsumer.java
│   └── resources/
│       └── application.yml
├── pom.xml
```

#### Dependencies (pom.xml)

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>kafka-manual-offset</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.kafka</groupId>
      <artifactId>spring-kafka</artifactId>
    </dependency>
  </dependencies>

  <properties>
    <java.version>17</java.version>
    <spring.boot.version>3.2.0</spring.boot.version>
  </properties>
</project>
```

#### application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-manual-group
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

#### Main Application

```java
package com.example.kafka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaManualOffsetApplication {
    public static void main(String[] args) {
        SpringApplication.run(KafkaManualOffsetApplication.class, args);
    }
}
```

#### Kafka Consumer Config

```java
package com.example.kafka.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties.AckMode;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "my-manual-group");
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // manual commit
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

#### Manual Offset Consumer

```java
package com.example.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Component
public class ManualOffsetKafkaConsumer {
    
    private static final Logger logger = LoggerFactory.getLogger(ManualOffsetKafkaConsumer.class);

    @KafkaListener(topics = "my-topic", groupId = "my-manual-group", 
                  containerFactory = "kafkaListenerContainerFactory")
    public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            logger.info("Consumed message: {}", record.value());
            
            // Simulate message processing
            processMessage(record);
            
            // After successful processing, manually commit the offset
            ack.acknowledge();
            logger.info("Offset committed for message: {}", record.offset());
        } catch (Exception e) {
            logger.error("Processing failed for message: {}", record.value(), e);
            // Do not acknowledge - will be retried later
        }
    }
    
    private void processMessage(ConsumerRecord<String, String> record) {
        try {
            // Simulate actual processing time
            Thread.sleep(1000);
            
            // Add your actual message processing logic here
            
            logger.info("Message successfully processed: {}", record.value());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Message processing interrupted", e);
        }
    }
}
```

---
