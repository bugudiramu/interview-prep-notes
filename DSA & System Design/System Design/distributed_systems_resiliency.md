# Level 14 — Redis, Kafka & Distributed Systems
## Part 3: Distributed Systems Fundamentals & Resiliency Patterns — Revision Notes

**Status:** Completed  
**Language / Stack:** Java + Spring Boot (`Resilience4j`)  
**Focus:** CAP Theorem, Consistency models, Distributed Locking, Idempotency, and network resiliency patterns (Circuit Breaker, Retry, Bulkhead, Backpressure)

---

# 1. Core Distributed Systems Theory

When you move from a monolithic Spring Boot application talking to a single PostgreSQL database to a distributed architecture, network unreliability becomes the primary engineering challenge.

### The CAP Theorem
The CAP Theorem states that a distributed data store can only simultaneously provide **two of the following three** guarantees:

| Guarantee | Meaning |
|---|---|
| **Consistency (C)** | Every read receives the most recent write or an error. All nodes see the exact same data at the exact same time. |
| **Availability (A)** | Every request receives a non-error response, but without the guarantee that it contains the most recent write. |
| **Partition Tolerance (P)** | The system continues to operate despite an arbitrary number of messages being dropped or delayed by the network between nodes. |

**The Reality of CAP:**
Because networks *will* fail, **Partition Tolerance (P) is not a choice—it is a requirement**. Therefore, when a network partition occurs, a distributed system must choose between:
*   **CP (Consistency + Partition Tolerance):** Cancel the operation to ensure data isn't corrupted or stale (e.g., MongoDB, ZooKeeper, HBase).
*   **AP (Availability + Partition Tolerance):** Return the local data you have, even if it might be stale (e.g., Cassandra, DynamoDB).

### Consistency Models
*   **Strong Consistency:** Once a write is acknowledged, every subsequent read will reflect that write. Required for financial transactions, inventory counts, and passwords. (Achieved via distributed locks or consensus protocols like Raft).
*   **Eventual Consistency:** If no new updates are made to a given data item, eventually all accesses to that item will return the last updated value. Required for high-availability systems (social media feeds, DNS, search indexes, likes/views).

---

# 2. Resiliency Patterns (Designing for Failure)

In microservices, if Service A calls Service B, and Service B is slow or failing, it can consume all of Service A's threads, causing Service A to crash. These patterns protect your systems from cascading failures.

In Spring Boot, these are typically implemented using **Resilience4j**.

## 1. Timeout
The most basic and critical pattern. Never make a network call over HTTP or to a database without a strict upper time bound.
*   **Problem:** Service B is deadlocked and takes 45 seconds to respond. Service A's Tomcat thread pool fills up waiting for Service B.
*   **Solution:** Set a strict timeout (e.g., 2 seconds). If B doesn't respond, cut the connection and throw an exception or return a fallback.

## 2. Retry & Exponential Backoff
When a timeout or transient error (e.g., `HTTP 503` or a brief network blip) occurs, the client automatically attempts the request again.

*   **Exponential Backoff:** Retrying immediately often worsens the problem if the downstream service is overloaded. Instead, increase the wait time between retries: $1\text{s} \rightarrow 2\text{s} \rightarrow 4\text{s} \rightarrow 8\text{s}$.
*   **Jitter (Critical):** If 1,000 clients all fail and backoff for exactly 2 seconds, they will all hammer the server again at the exact same millisecond. **Jitter** adds randomness (e.g., $2\text{s} \pm 500\text{ms}$) to spread out the retry spikes.

```java
// Spring Boot + Resilience4j Example
@Retry(name = "paymentService", fallbackMethod = "fallbackPayment")
public PaymentResponse processPayment() {
    return restTemplate.getForObject("/api/payments", PaymentResponse.class);
}
```

## 3. Circuit Breaker
If a downstream service is completely down, continuously retrying wastes resources and prevents the downstream service from recovering. A Circuit Breaker acts like an electrical switch.

| State | Behavior |
|---|---|
| **CLOSED** | Normal operation. Requests flow freely. The breaker counts failure rates. |
| **OPEN** | If the failure rate exceeds a threshold (e.g., 50% over 10 calls), the breaker opens. **All requests fail fast immediately** without attempting the network call, giving the downstream service time to recover. |
| **HALF-OPEN** | After a wait duration, the breaker allows a limited number of "test" requests through. If they succeed, it transitions to **CLOSED**. If they fail, it trips back to **OPEN**. |

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "defaultInventory")
public Inventory checkInventory(String productId) {
    // If open, this code isn't even executed; it goes straight to defaultInventory
}
```

## 4. Bulkhead
Named after the watertight compartments of a ship. If one compartment floods, the ship doesn't sink.
*   **Problem:** Your service has APIs for `/checkout` and `/reports`. The `/reports` API makes a slow DB query. If a traffic spike hits `/reports`, it consumes all 200 Tomcat threads, leaving 0 threads for the critical `/checkout` API.
*   **Solution:** Partition your thread pools or concurrent call limits. Allocate max 20 threads to `/reports` and 180 to `/checkout`.

## 5. Backpressure
When a publisher is sending data faster than a consumer can process it, the consumer will eventually run out of memory (OOM) and crash.
*   **Backpressure Strategies:**
    1.  **Buffer:** Queue the excess messages (Kafka topics).
    2.  **Drop:** Drop the oldest or newest messages (Metrics logging).
    3.  **Pushback (Control):** The consumer signals the producer to slow down (Reactive Streams / Project Reactor in Spring WebFlux).

---

# 3. State & Coordination

## 1. Idempotency (Safe Retries)
Because network responses can be lost *after* a successful operation (e.g., the payment succeeds, but the HTTP 200 OK drops), clients will retry.
*   **Idempotency:** An operation is idempotent if executing it once has the exact same effect as executing it multiple times.
*   **Implementation:** The client generates a unique `Idempotency-Key` (UUID) header. The server checks a database or Redis cache to see if this key has already been processed. If yes, it returns the cached successful response without executing the business logic again.

## 2. Distributed Locking
*(Covered in-depth in Redis notes)*
Ensures that across a fleet of microservices, only one node can execute a specific critical section of code at a time (e.g., generating a daily financial report, or refunding a specific transaction). Requires a TTL/Lease and an owner UUID to prevent deadlocks if the lock holder crashes.

---

# 4. Pattern Recognition & Interview Cheat Sheet

| If the interviewer asks… | The Solution is… |
|---|---|
| "How do you prevent a slow microservice from bringing down the services that call it?" | **Circuit Breaker** to fail fast, plus strict **Timeouts**. |
| "A downstream service occasionally drops connections due to network blips. How do we handle this?" | **Retry with Exponential Backoff and Jitter**. |
| "Users are double-clicking the 'Submit Order' button and creating two orders." | Generate an **Idempotency Key** on the frontend and deduplicate on the backend via a database constraint or Redis cache. |
| "A heavily requested reporting endpoint is crashing our main transaction API." | **Bulkhead Pattern**. Isolate the thread pools or limit concurrent calls for the reporting endpoint. |
| "We are reading social media posts from a replica database, and sometimes a user's own post doesn't show up immediately after they post it." | Explain **Eventual Consistency** and replication lag. Fix it by doing a "read-your-own-writes" cache lookup for the first 5 seconds. |
| "How do you count unique visitors across a cluster of 10 nodes without double-counting?" | This requires coordination. Use **Redis (HyperLogLog)** or route by IP hash to specific nodes. |

---

# 5. Revision Checklist

- [x] Define CAP Theorem and explain why CP vs AP is the actual choice during a network partition.
- [x] Contrast Strong Consistency with Eventual Consistency.
- [x] Explain how a Circuit Breaker transitions between Closed, Open, and Half-Open.
- [x] Explain why Jitter is mandatory when using Exponential Backoff.
- [x] Describe the Bulkhead pattern for thread isolation.
- [x] Explain Idempotency and how to implement it using a unique key.