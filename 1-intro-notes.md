# Apache Kafka Core Concepts

Apache Kafka is a distributed streaming platform designed to handle high-throughput, fault-tolerant data pipelines. Understanding the following core concepts is essential for working with Kafka effectively.
- async msg model(non-blocking networks call bw services)
- pub/sub model 
- mediates communication bw services

## 1. Kafka Cluster

A Kafka cluster is a distributed system consisting of multiple servers working together. Each server in the cluster is called a broker.

- **Definition**: A collection of multiple Kafka brokers operating together as a single system
- **Purpose**: Provides scalability, fault tolerance, and high availability
- **Minimum Configuration**: Typically 3 brokers for production environments

## 2. Kafka Broker

A broker is an individual Kafka server instance that acts as an intermediary between data producers and consumers.

- **Definition**: A Kafka server that stores data and serves client requests
- **Role**: Mediates communication between producers and consumers without direct interaction
- **Function**: Receives, stores, and allows retrieval of messages

## 3. Kafka Producer

Producers are applications that publish data to Kafka topics.

- **Definition**: An application that sends messages to Kafka
- **Behavior**: Sends messages to Kafka brokers, not directly to consumers
- **Responsibility**: Determines which topic and partition to send messages to

## 4. Kafka Consumer

Consumers are applications that subscribe to topics and process the published messages.

- **Definition**: An application that reads messages from Kafka
- **Behavior**: Requests and reads data from Kafka brokers
- **Capability**: Can consume messages from any producer (with appropriate permissions)

## 5. Kafka Topic

Topics are logical channels or categories to which messages are published.

- **Definition**: A named feed or category of messages
- **Analogy**: Similar to a table in a database or a folder in a file system
- **Structure**: Each topic consists of one or more partitions
- **Flexibility**: You can create any number of topics based on your needs

## 6. Kafka Partitions

Partitions are the units of parallelism in Kafka that allow data to be distributed across multiple brokers.

- **Definition**: Ordered, immutable sequence of records within a topic
- **Purpose**: Enables horizontal scaling and parallel processing
- **Distribution**: Partitions are distributed across multiple brokers for load balancing
- **Characteristic**: Each partition is an ordered, immutable sequence of messages

    - Inside a **single partition**, messages are **always ordered by offset** (a unique ID for each message).
    - Once a message is written, it is **never changed** (immutable).
    - Consumers **read messages in order**, one by one.

## 7. Kafka Offsets

Offsets are sequential identifiers assigned to messages within a partition.

- **Definition**: A unique, sequential identifier for each message in a partition
- **Assignment**: First message in a partition gets offset 0, next gets 1, and so on
- **Immutability**: Once assigned, an offset never changes
- **Usage**: Consumers use offsets to track their position in a partition

## 8. Kafka Consumer Group

Consumer groups allow multiple consumers to collaborate on processing messages from topics.

- **Definition**: A collection of consumers working together to process messages
- **Load Balancing**: Partitions are distributed among consumers in a group
- **Scalability**: Adding consumers to a group (up to the number of partitions) increases throughput
- **Fault Tolerance**: If a consumer fails, its partitions are reassigned to other group members

---

## Kafka Message Anatomy
Each Kafka message (called a *record*) includes the following components:

* **Topic** – The logical category or name to which the message belongs
* **Key** (optional) – Used to determine the partition; helps in grouping related messages
* **Value** – The actual business data or payload
* **Timestamp** – The time when the message was produced
* **Partition** (optional) – The specific partition within the topic where the message is stored; if not provided, Kafka decides automatically

---

## What is a Serializer?
Kafka stores all data as byte arrays.
A *Serializer* is responsible for converting keys and values (such as strings, integers, or JSON objects) into byte arrays before sending them to Kafka.

Kafka provides built-in serializers for common data types, including:

* String
* Integer
* Long
* Byte array

Custom serializers can be created for more complex types.

---

## What is a Partitioner?
When a producer sends a message to Kafka, a *Partitioner* determines which partition within a topic the message should go to.

Kafka's partitioning logic:

* If a **key is provided**, Kafka applies a hash function to the key to select a partition (`hash(key) % number of partitions`)
* If **no key is provided**, Kafka uses a round-robin approach to distribute messages evenly across partitions
* You can also implement a **custom partitioner** to apply specific routing logic based on your application's needs

