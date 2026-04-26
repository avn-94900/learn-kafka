# 7. Kafka Connect & Integration

---

### What is Kafka Connect?

Kafka Connect is a **framework for streaming data between Kafka and external systems** (databases, file systems, cloud services, etc.) without writing custom producer/consumer code.

It provides:
- Pre-built connectors for common systems (JDBC, S3, Elasticsearch, etc.)
- Scalable, fault-tolerant distributed execution
- Automatic offset management and schema handling
- REST API for managing connectors

---

### How does Kafka Connect differ from Kafka Streams?

| Aspect | Kafka Connect | Kafka Streams |
|--------|---------------|---------------|
| **Purpose** | Data integration (ETL) | Stream processing |
| **Architecture** | Connector framework | Library/API |
| **Deployment** | Standalone or distributed cluster | Embedded in application |
| **Data Sources** | External systems (DB, files, cloud) | Kafka topics only |
| **Processing** | Simple transformations (SMTs) | Complex stream processing |
| **Scalability** | Horizontal via workers | Application-level scaling |
| **Configuration** | JSON-based connector configs | Java/Scala code |
| **Use Cases** | Data ingestion/export | Real-time analytics, aggregations |

---

### What are source and sink connectors?

**Source Connector:** Reads data from an external system and **writes into Kafka**
- Examples: JDBC Source (database → Kafka), Debezium CDC (DB change events → Kafka), FileStream Source

**Sink Connector:** Reads data from Kafka and **writes to an external system**
- Examples: JDBC Sink (Kafka → database), S3 Sink (Kafka → S3), Elasticsearch Sink

**Example source connector config (JDBC):**
```json
{
  "name": "jdbc-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://localhost:5432/mydb",
    "table.whitelist": "orders",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "db-"
  }
}
```

---

### How do you handle schema evolution?

**Challenges:** Producers and consumers may be deployed independently; schema changes must not break existing consumers.

**Strategies:**
- **Backward compatible changes:** Add optional fields with defaults — old consumers ignore new fields
- **Forward compatible changes:** Remove optional fields — new consumers handle missing fields
- **Full compatibility:** Both backward and forward compatible
- **Avoid:** Renaming fields, changing field types, removing required fields

**Best practices:**
- Always add new fields as optional with default values
- Never remove or rename fields in use
- Version your schemas
- Test compatibility before deploying

---

### How does Avro + Schema Registry help schema management?

**Schema Registry** is a centralized service that stores and enforces Avro (or JSON/Protobuf) schemas.

**How it works:**
1. Producer registers schema with Schema Registry on first use
2. Schema Registry assigns a schema ID
3. Producer serializes message as: `[magic byte][schema ID][avro bytes]`
4. Consumer fetches schema by ID from registry and deserializes

**Benefits:**
- Enforces schema compatibility rules (backward, forward, full)
- Eliminates need to embed schema in every message (just the ID)
- Centralized schema versioning and governance
- Automatic serialization/deserialization

**Implementation:**
```java
// Producer
props.put("schema.registry.url", "http://localhost:8081");
props.put("value.serializer", KafkaAvroSerializer.class);

// Consumer
props.put("schema.registry.url", "http://localhost:8081");
props.put("value.deserializer", KafkaAvroDeserializer.class);
props.put("specific.avro.reader", "true");
```

**Compatibility types:**

| Compatibility | Add Fields | Remove Fields |
|---------------|-----------|---------------|
| **BACKWARD** | ✓ (with defaults) | ✓ (if optional) |
| **FORWARD** | ✓ (optional) | ✓ (if not required) |
| **FULL** | ✓ (optional with defaults) | ✓ (if optional) |
