# mTLS — Folder 2: Client Configuration

## Learning Path
msk-mtls-learning/
├── folder-1-mtls-authorization-design.md
├── folder-2-mtls-client-configuration.md   ← YOU ARE HERE
└── folder-3-mtls-acls-and-production.md

---

## 1. What the Client Needs

| File              | Purpose                                          |
|-------------------|--------------------------------------------------|
| Client certificate (.pem / .crt) | Proves identity to the broker      |
| Client private key (.key)        | Signs the TLS handshake            |
| CA certificate (.pem)            | Trusts the broker's server cert    |
| Keystore (.jks / .p12)           | Bundles cert + key for Java clients|
| Truststore (.jks)                | Holds the CA cert for Java clients |

---

## 2. Create Keystore and Truststore (One-Time Setup)

```bash
# Step 1 — Import client cert + key into a PKCS12 keystore
openssl pkcs12 -export \
  -in client-cert.pem \
  -inkey client-key.pem \
  -out client-keystore.p12 \
  -name client \
  -passout pass:keystorepassword

# Step 2 — Convert to JKS (Java KeyStore)
keytool -importkeystore \
  -srckeystore client-keystore.p12 \
  -srcstoretype PKCS12 \
  -destkeystore client-keystore.jks \
  -deststoretype JKS \
  -srcstorepass keystorepassword \
  -deststorepass keystorepassword

# Step 3 — Import CA cert into truststore
keytool -import -trustcacerts \
  -alias ca-cert \
  -file ca-cert.pem \
  -keystore client-truststore.jks \
  -storepass truststorepassword \
  -noprompt
```

---

## 3. Java Client Properties — Producer

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094

# mTLS — port 9094
security.protocol=SSL

# Keystore — client identity
ssl.keystore.location=/app/certs/client-keystore.jks
ssl.keystore.password=keystorepassword
ssl.key.password=keystorepassword

# Truststore — broker CA trust
ssl.truststore.location=/app/certs/client-truststore.jks
ssl.truststore.password=truststorepassword

key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
```

> Port 9094 is the mTLS listener. Port 9098 is IAM. Port 9092 is plaintext.

---

## 4. Java Client Properties — Consumer

```properties
bootstrap.servers=b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094
security.protocol=SSL

ssl.keystore.location=/app/certs/client-keystore.jks
ssl.keystore.password=keystorepassword
ssl.key.password=keystorepassword

ssl.truststore.location=/app/certs/client-truststore.jks
ssl.truststore.password=truststorepassword

group.id=orders-consumer-group
auto.offset.reset=earliest
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

> The group.id must match the ACL principal + group name set in Kafka ACLs.

---

## 5. CLI Tools with mTLS

Create a `client.properties` file:

```properties
security.protocol=SSL
ssl.keystore.location=/app/certs/client-keystore.jks
ssl.keystore.password=keystorepassword
ssl.key.password=keystorepassword
ssl.truststore.location=/app/certs/client-truststore.jks
ssl.truststore.password=truststorepassword
```

Produce:

```bash
kafka-console-producer.sh \
  --broker-list b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --topic orders-topic \
  --producer.config client.properties
```

Consume:

```bash
kafka-console-consumer.sh \
  --bootstrap-server b-1.my-cluster.abc123.c2.kafka.us-east-1.amazonaws.com:9094 \
  --topic orders-topic \
  --group orders-consumer-group \
  --from-beginning \
  --consumer.config client.properties
```

---

## 6. Storing Certs Securely in Production

Never store keystore files on disk in containers. Use Secrets Manager:

```bash
# Store keystore as a binary secret
aws secretsmanager create-secret \
  --name /msk/orders-producer/keystore \
  --secret-binary fileb://client-keystore.jks

# Retrieve at container startup
aws secretsmanager get-secret-value \
  --secret-id /msk/orders-producer/keystore \
  --query SecretBinary \
  --output text | base64 --decode > /tmp/client-keystore.jks
```

---

## 7. Common Mistakes

| Mistake                                    | Result                                  |
|--------------------------------------------|-----------------------------------------|
| Using port 9092 with SSL config            | Connection refused                      |
| Truststore missing broker CA               | SSL handshake failure                   |
| Keystore CN doesn't match ACL principal    | Auth succeeds but all operations denied |
| Expired client certificate                 | TLS handshake rejected by broker        |
| Private key password mismatch              | Keystore load failure at startup        |

---

## Key Takeaway

The keystore holds your identity (cert + key). The truststore holds your trust (CA cert).
Both are required. A missing or misconfigured truststore causes broker SSL rejection before any ACL check even runs.
