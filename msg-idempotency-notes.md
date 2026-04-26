Here’s a **clean, compact note** you can use for quick revision:

---

## 📝 Idempotent Producer & Deduplication in Kafka

In **Apache Kafka**:

### 🔹 Core Configuration

```properties
enable.idempotence=true
```

---

## 🔹 What this does

* Kafka assigns:

  * **Producer ID (PID)**
  * **Sequence numbers (per partition)**

* These are used to:

  * Detect **duplicate retries**
  * Ensure each message is written **only once**

---

## 🔹 How deduplication works

* Each message → sent with a **sequence number**
* If retry happens → same sequence number is reused
* Broker checks:

  * If already received → **drops duplicate**
  * Else → **writes message**

👉 Ensures: **No duplicate messages due to retries**

---

## 🔹 Why `acks=all` is required

* Producer writes to **leader**
* Message must be replicated to **in-sync replicas (ISR)** before ACK

### ✔ With `acks=all`:

* All replicas have the message
* Retry → safely detected → **no duplication**

### ❌ With `acks=1` or `acks=0`:

* Message may not be replicated
* Leader crash → data loss
* Retry → **duplicate can be written**

---

## 🔹 Key guarantees

* No duplicates **within a partition**
* Safe retries
* Ordering preserved

---

## 🔹 Important notes

* No separate “deduplication config”
* Idempotence = built-in dedup mechanism
* Works at **broker level**, not based on message content

---

## 🔹 One-line summary

> `enable.idempotence=true` + `acks=all` ensures **retry-safe, duplicate-free, and durable message writes** in Kafka.

---

If you want, I can shrink this into a **2–3 line interview answer** as well.
