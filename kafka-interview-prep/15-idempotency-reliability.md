# 15. Idempotency & Reliability (Distributed Systems)

---

### What are idempotency keys?

An **idempotency key** is a unique identifier attached to a request or message that allows the receiver to detect and safely ignore duplicate submissions.

**Example:** A payment request includes `idempotency-key: txn-abc-123`. If the client retries due to a timeout, the server recognizes the key and returns the original result instead of processing the payment again.

---

### Why are idempotency keys used?

In distributed systems, **retries are unavoidable** due to:
- Network timeouts (request sent but response lost)
- Service crashes after processing but before responding
- Client-side timeouts

Without idempotency keys, retries cause **duplicate operations** (double charges, duplicate orders, etc.).

**Idempotency keys ensure:** "No matter how many times you send this request, the effect happens exactly once."

---

### How do idempotency keys prevent duplicates?

**Server-side flow:**
1. Client sends request with `idempotency-key: <uuid>`
2. Server checks if key exists in a deduplication store (Redis, DB)
3. If **not found:** Process request, store result with the key, return result
4. If **found:** Return stored result immediately (skip processing)

```java
public Response processPayment(PaymentRequest request, String idempotencyKey) {
    Optional<Response> cached = idempotencyStore.get(idempotencyKey);
    if (cached.isPresent()) return cached.get();  // Duplicate — return cached result

    Response result = paymentService.charge(request);
    idempotencyStore.save(idempotencyKey, result, Duration.ofDays(1));
    return result;
}
```

**In Kafka:** Producer uses sequence numbers (PID + seq) as implicit idempotency keys; broker deduplicates automatically when `enable.idempotence=true`.

---

### What are retry strategies?

**Fixed Retry:**
- Retry after a constant delay
- Simple to implement
- Risk: thundering herd if many clients retry simultaneously

```java
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
public void callExternalService() { }
```

**Exponential Backoff:**
- Delay doubles with each retry: 1s → 2s → 4s → 8s
- Reduces load on recovering services
- Add **jitter** (random variation) to prevent synchronized retries

```java
@Retryable(
    maxAttempts = 5,
    backoff = @Backoff(delay = 500, multiplier = 2.0, maxDelay = 30000, random = true)
)
public void callExternalService() { }
```

**Retry with circuit breaker:**
- After N consecutive failures, open the circuit (stop retrying)
- Periodically probe to check if service recovered
- Prevents cascading failures

---

### What is the role of timeouts and deadlines?

**Timeout:** Maximum time to wait for a single operation before giving up.

**Deadline:** Absolute point in time by which the entire operation (including retries) must complete.

**Why they matter:**
- Without timeouts, threads block indefinitely → resource exhaustion
- Deadlines prevent retrying past the point where the result is still useful

**Example:**
```java
// Timeout per attempt
restTemplate.setRequestFactory(new SimpleClientHttpRequestFactory());
factory.setConnectTimeout(2000);   // 2s to connect
factory.setReadTimeout(5000);      // 5s to read response

// Deadline — don't retry after 30s total
Instant deadline = Instant.now().plusSeconds(30);
while (Instant.now().isBefore(deadline)) {
    try { return callService(); }
    catch (Exception e) { Thread.sleep(backoff); }
}
throw new DeadlineExceededException();
```

---

### How do retries impact system reliability?

**Positive impacts:**
- Recover from transient failures automatically
- Improve availability without manual intervention
- Handle brief network blips transparently

**Negative impacts (if not designed carefully):**
- **Thundering herd:** All clients retry simultaneously, overwhelming a recovering service
- **Cascading failures:** Retries amplify load on an already struggling service
- **Duplicate side effects:** Without idempotency, retries cause duplicate operations
- **Increased latency:** Retries add delay to the overall operation

**Best practices:**
- Always use **exponential backoff with jitter**
- Set a **max retry limit** and **deadline**
- Use **circuit breakers** to stop retrying when service is clearly down
- Ensure all retried operations are **idempotent**
- Monitor retry rates as a signal of system health
