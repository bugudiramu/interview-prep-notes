# Level 14 — Redis, Kafka & Distributed Systems
## Part 2: Redis — Core Patterns, Distributed Caching & System Design — Revision Notes

**Status:** In progress  
**Language / Stack:** Java + Spring Boot (`Spring Data Redis` / `Redisson`)  
**Focus:** Caching strategies, TTL, invalidation, distributed locks, rate limiting, sessions, Pub/Sub, and interview patterns

---

# 1. Redis Fundamentals

**Redis** (Remote Dictionary Server) is an open-source, **in-memory key-value data structure store** used as a database, cache, message broker, and streaming engine.

### Why is Redis so fast?
1. **In-Memory Storage:** Data lives in RAM rather than on spinning disks or SSDs, yielding sub-millisecond latency ($< 1\text{ ms}$).
2. **Single-Threaded Event Loop:** Uses an I/O multiplexing model (epoll/kqueue) to process commands sequentially on a single main thread. This eliminates context-switching and lock-contention overhead.
3. **Optimized Low-Level Data Structures:** Built in C using custom structures like Simple Dynamic Strings (SDS), Skip Lists, and Ziplists.

### Core Data Types & Complexity

| Data Type | Common Use Case | Key Commands | Time Complexity |
|---|---|---|---|
| **String** | Caching JSON/HTML, atomic counters, distributed locks | `GET`, `SET`, `INCR`, `DECR` | $\mathcal{O}(1)$ |
| **Hash** | Storing flat objects (e.g., User profile fields, cart items) | `HGET`, `HSET`, `HGETALL` | $\mathcal{O}(1)$ per field; $\mathcal{O}(N)$ for all |
| **List** | Activity feeds, simple task queues (FIFO/LIFO) | `LPUSH`, `RPUSH`, `LPOP` | $\mathcal{O}(1)$ at ends |
| **Set** | Unique visitors, tags, friend connections, IP allowlists | `SADD`, `SISMEMBER`, `SINTER` | $\mathcal{O}(1)$ add/check |
| **Sorted Set (ZSet)** | Real-time leaderboards, sliding-window rate limiters, priority queues | `ZADD`, `ZRANGE`, `ZRANK` | $\mathcal{O}(\log N)$ |

---

# 2. Caching Strategies

Choosing how your application interacts with Redis and PostgreSQL is one of the most common backend interview topics.

## Pattern 1: Cache-Aside (Lazy Loading) — *Most Common*

The application is responsible for talking to both the cache and the database.

```text
Read Path:
1. App checks Redis
2. Cache Hit?  --> Return data immediately
3. Cache Miss? --> Query PostgreSQL --> Write to Redis (with TTL) --> Return data
```

```java
public UserDto getUserById(Long userId) {
    String key = "user:" + userId;

    // 1. Check Cache
    UserDto cachedUser = (UserDto) redisTemplate.opsForValue().get(key);
    if (cachedUser != null) {
        return cachedUser; // Cache Hit
    }

    // 2. Cache Miss -> Load from DB
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
    UserDto dto = UserDto.from(user);

    // 3. Populate Cache with TTL
    redisTemplate.opsForValue().set(key, dto, Duration.ofMinutes(15));

    return dto;
}
```

- **Pros:** Resilient to Redis failures (falls back to DB); only caches data that is actually requested.
- **Cons:** First request always suffers a **Cache Miss penalty** (3 network hops); data can become stale if updated in DB without invalidating the cache.

## Other Caching Patterns Comparison

| Pattern | How it Works | Pros | Cons |
|---|---|---|---|
| **Cache-Aside** | App reads/writes cache & DB explicitly | Only requested data is cached; resilient | Cache miss penalty on cold start |
| **Read-Through** | Cache sits inline; cache itself loads from DB on miss | Cleaner application code | Requires cache provider plugin support |
| **Write-Through** | App writes to Cache, and Cache synchronously writes to DB | Cache is never stale | Higher write latency (writes hit both systems) |
| **Write-Behind (Write-Back)** | App writes to Cache; Cache flushes to DB asynchronously in batches | Extremely fast writes; absorbs write spikes | Risk of data loss if Redis crashes before DB flush |

---

# 3. TTL, Eviction Policies & Cache Invalidation

### TTL (Time-To-Live)
Every cached key in production should almost always have a **TTL** (`EXPIRE` / `PEXPIRE`).
- Acts as a safety net against stale data.
- Prevents unbounded memory growth (`OutOfMemory` errors).

### Cache Invalidation on Writes
When data changes in PostgreSQL (e.g., `updateUser`), what should you do with the Redis key?

> **Key Rule:** **Delete** the cache key rather than updating it (`redisTemplate.delete(key)`).

**Why delete instead of update?**
If two concurrent requests update the database in order `A -> B`, network jitter could cause their cache updates to arrive at Redis in order `B -> A`, leaving the cache permanently out of sync with the database until TTL expires. Deleting forces the next read to fetch fresh state via Cache-Aside.

### Production Cache Failure Modes & Mitigations

| Failure Mode | What Happens | Solution / Mitigation |
|---|---|---|
| **Cache Stampede (Thundering Herd)** | A popular key expires; thousands of concurrent requests miss the cache and hammer PostgreSQL simultaneously. | 1. **Distributed Lock / Mutex** on cache rebuild.<br>2. **Logical early expiration** (refresh slightly before TTL expires). |
| **Cache Penetration** | Requests query for non-existent IDs (e.g., `user:-999`). Misses Redis and hits PostgreSQL every time. | 1. **Cache Null Values** with a short TTL (e.g., 30–60s).<br>2. Use a **Bloom Filter** in front of Redis. |
| **Cache Avalanche** | Thousands of keys set with the exact same TTL expire at the exact same second, overwhelming the DB. | **Add Random Jitter** to TTLs: $\text{TTL} = \text{Base\_TTL} + \text{random}(0, 300\text{s})$. |
| **Hot Key Problem** | A single key (e.g., celebrity post) receives millions of reads/sec, overloading a single Redis shard. | 1. Local in-memory cache (Caffeine) with short TTL.<br>2. Replicate key across multiple shards (`key#1`, `key#2`). |

### Memory Eviction Policies (`maxmemory-policy`)
When Redis reaches its RAM limit:
- **`allkeys-lru` / `volatile-lru`:** Evicts Least Recently Used keys (standard for caching).
- **`allkeys-lfu` / `volatile-lfu`:** Evicts Least Frequently Used keys.
- **`noeviction`:** Returns errors on writes when memory is full (used when Redis acts as a primary store or queue).

---

# 4. Atomic Counters & Rate Limiting

Because Redis executes commands sequentially on a single thread, `INCR` and `DECR` are **100% atomic** without needing database row locks.

### Use Cases for Counters
- Real-time video view counts / post likes.
- Inventory stock reservation counters.
- API rate limiting per user or IP.

## Pattern 1: Fixed-Window Rate Limiter

Limit a user to $N$ requests per minute.

```java
public boolean isAllowedFixedWindow(String userId, int maxRequests) {
    long currentMinute = System.currentTimeMillis() / 60000;
    String key = "rate_limit:" + userId + ":" + currentMinute;

    Long count = redisTemplate.opsForValue().increment(key);

    if (count != null && count == 1) {
        // First request in this window -> set expiry so keys clean up automatically
        redisTemplate.expire(key, Duration.ofSeconds(65));
    }

    return count != null && count <= maxRequests;
}
```
- **Trade-off:** Boundary burst issue—a user can send $N$ requests at `00:59` and another $N$ requests at `01:01` ($2N$ requests in 2 seconds).

## Pattern 2: Sliding-Window Log Rate Limiter (Using Sorted Sets)

Solves the boundary burst problem by tracking exact timestamps in a Redis **ZSet**.

1. `ZREMRANGEBYSCORE key 0 (now - windowSize)`: Remove timestamps older than the sliding window.
2. `ZCARD key`: Count remaining requests in the current window.
3. If `count < limit`, add current request: `ZADD key now uniqueRequestId` and set `EXPIRE`.

> **Production Note:** In multi-step Redis operations like Sliding Window or Token Bucket, wrap the steps in a **Lua Script** so Redis executes all commands atomically in a single network round-trip.

---

# 5. Distributed Locks

When running multiple instances of a Spring Boot microservice behind a load balancer, Java's `synchronized` or `ReentrantLock` only locks threads inside a **single JVM**. To prevent two servers from processing the same payment or cron job simultaneously, you need a **Distributed Lock**.

## Core Pattern: `SET key value NX PX milliseconds`

- **`NX` (Not Exists):** Only acquire the lock if the key does not already exist.
- **`PX` (Milliseconds TTL):** Automatically release the lock after a lease timeout so a crashed server doesn't hold the lock forever (deadlock prevention).
- **`value` (`UUID`):** Must be a unique token per thread/request so a server only unlocks its **own** lock.

```java
public boolean acquireLock(String lockKey, String lockOwnerUuid, long leaseMillis) {
    Boolean acquired = redisTemplate.opsForValue().setIfAbsent(
            lockKey,
            lockOwnerUuid,
            Duration.ofMillis(leaseMillis)
    );
    return Boolean.TRUE.equals(acquired);
}
```

### Safe Lock Release (Why a Lua Script is Mandatory)

**Never** do `GET` in Java followed by `DEL` in Java to release a lock:
```text
Thread A lock expires at t=10s due to GC pause.
Thread B acquires lock at t=10.1s.
Thread A wakes up at t=10.2s and calls DEL --> Thread A just deleted Thread B's active lock!
```

Instead, use an atomic Lua script to verify ownership before deleting:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### Production Standard: Redisson
In Spring Boot production projects, use **Redisson** (`RLock`) instead of hand-rolling locks:
- Provides a **Watchdog mechanism** that automatically renews the lock's TTL while your thread is still alive and working.
- Implements pub/sub lock waiting rather than aggressive busy-spinning.

---

# 6. Distributed Sessions

In a stateless microservice architecture, storing session state in a single server's memory (`HttpSession`) breaks when a load balancer routes the user's next request to a different server (unless sticky sessions are enabled).

```text
User Request --> Load Balancer --> Spring Boot Instance 1, 2, or 3
                                           ↓
                                  Central Redis Cluster
                                 (Session Token / State)
```

### Two Common Auth/Session Patterns with Redis

1. **Server-Side Sessions (`Spring Session Data Redis`):**
   - Browser holds an opaque `SESSIONID` cookie.
   - Redis stores user principals, roles, and metadata in a Redis Hash with sliding TTL (`30m`).
   - Instant logout: simply delete the session key in Redis.
2. **JWT Denylist / Revocation + Refresh Tokens:**
   - Access tokens (JWT) are short-lived (e.g., 5–15 mins) and verified statelessly.
   - Refresh tokens are stored in Redis with TTL matching the refresh period (e.g., 7 days).
   - If a user logs out or gets banned before their JWT expires, store `blacklist:jti:<token_id>` in Redis with a TTL equal to the JWT's remaining lifetime.

---

# 7. Redis Pub/Sub vs. Apache Kafka

Redis includes a lightweight **Publish/Subscribe** messaging system where publishers send messages to channels and subscribers receive them immediately.

```text
Publisher --> Redis Channel ("chat:room:42") --> Active Subscriber 1
                                             --> Active Subscriber 2
```

### Critical Interview Comparison: Redis Pub/Sub vs. Kafka

| Feature | Redis Pub/Sub | Apache Kafka |
|---|---|---|
| **Message Persistence** | **None (Fire-and-Forget).** Stored only in RAM buffers briefly during delivery. | **Persistent on Disk** (Immutable append-only log with configurable retention). |
| **Offline Consumers** | If a subscriber is disconnected when a message is published, **the message is lost forever**. | Consumers track offsets and read missed messages whenever they come back online. |
| **Delivery Semantic** | Push-based (at-most-once) | Pull-based (at-least-once / exactly-once) |
| **Consumer Groups** | No load-balancing across group members; **every** subscriber gets a copy (fan-out). *(Note: Redis Streams adds consumer groups).* | Built-in Consumer Groups for partitioned load balancing + fan-out across groups. |
| **Throughput & Backlog** | Degrades if slow subscribers cause output buffers to overflow. | Handles massive backlogs (terabytes of data) without impacting broker memory. |
| **Best Use Case** | Ephemeral real-time events: WebSocket chat fan-out, live presence, cache invalidation broadcasts. | Financial transactions, order pipelines, audit logs, durable event-driven microservices. |

---

# 8. Persistence & High Availability (Brief Architecture Overview)

Even though Redis is in-memory, it supports disk persistence and cluster replication so caches don't start completely cold after a reboot.

### Persistence Options
1. **RDB (Redis Database Snapshot):** Point-in-time binary snapshots at intervals (e.g., every 5 mins). Fast restarts, compact files, but loses data between snapshots.
2. **AOF (Append-Only File):** Logs every write operation sequentially (`fsync` every second or every write). Much higher durability, slower restarts, larger files.
3. **Hybrid (RDB + AOF):** Production standard when persistence is required.

### High Availability Topologies
- **Redis Sentinel:** Primary-Replica setup with automatic failover monitoring (great when data fits on a single machine's RAM).
- **Redis Cluster:** Horizontal sharding across multiple nodes using **16,384 hash slots** ($\text{CRC16}(\text{key}) \bmod 16384$). Scales both memory capacity and write throughput.

---

# 9. Pattern Recognition & Interview Cheat Sheet

| If the interviewer asks… | Think… |
|---|---|
| Our PostgreSQL database CPU is at 95% due to repetitive read queries on product pages. | **Cache-Aside pattern** with Redis + randomized **TTL** to prevent Cache Avalanche. |
| How do you design a real-time gaming leaderboard for 10 million players? | Redis **Sorted Set (`ZSet`)**: `ZADD` for updating scores in $\mathcal{O}(\log N)$, `ZREVRANGE` for top $K$, `ZREVRANK` for player's rank. |
| How do you prevent a user from calling an expensive API more than 100 times per minute? | Redis **Rate Limiter** (Fixed Window via `INCR`+`EXPIRE`, or Sliding Window via `ZSet` + Lua script). |
| How do we prevent two instances of our Payment Worker from double-charging or running a job twice? | **Redis Distributed Lock** (`SET NX PX` with unique owner UUID + Lua script unlock, or **Redisson** + database idempotency key). |
| What happens if a celebrity's cached profile expires and 50,000 requests hit the DB at once? | **Cache Stampede**. Solve with a distributed mutex lock on cache recomputation or logical early refresh. |
| Should we use Redis Pub/Sub or Kafka for our payment transaction events? | **Kafka**. Redis Pub/Sub is fire-and-forget; if the payment worker restarts, events are permanently lost. |
| How do we broadcast WebSocket messages across 10 chat servers when two users are on different servers? | **Redis Pub/Sub**. Each chat server subscribes to user/room channels to route real-time ephemeral frames. |

---

# 10. Common Redis Mistakes

- **Setting Keys Without a TTL:** Slowly leaks memory until Redis hits `maxmemory` and starts evicting critical keys or throwing `OOM` errors.
- **Using `KEYS *` in Production:** `KEYS *` scans the entire keyspace in $\mathcal{O}(N)$ on Redis's single thread, **freezing all other requests** across your entire application. Always use `SCAN` instead.
- **Updating Cache Instead of Deleting on DB Write:** Leads to race conditions where concurrent writes leave stale data in Redis. Prefer invalidation (`DEL`).
- **Unlocking a Distributed Lock Without Checking the Owner Value:** Allows a delayed thread whose lock expired to accidentally delete another thread's active lock.
- **Treating Redis Distributed Locks as 100%Foolproof Without Idempotency:** If a JVM experiences a long Stop-The-World GC pause longer than the lock TTL, the lock expires early. Always pair distributed locks with **database-level idempotency / optimistic locking (`@Version`)** for financial transactions.
- **Storing Massive Blobs (Big Keys):** Storing 10 MB JSON blobs in a single key blocks the single-threaded event loop during network serialization/deserialization.

---

# 11. Revision Checklist

## Caching & Core Data Structures
- [ ] Explain why Redis is fast (RAM, single-threaded event loop, I/O multiplexing).
- [ ] Implement the **Cache-Aside** pattern in Spring Boot.
- [ ] Explain why deleting a cache key is safer than updating it on writes.
- [ ] Differentiate **Cache Stampede**, **Cache Penetration**, and **Cache Avalanche** and state their fixes.
- [ ] Choose the right Redis data structure (`String`, `Hash`, `Set`, `ZSet`).

## Distributed Coordination & System Design
- [ ] Implement a Fixed-Window and Sliding-Window **Rate Limiter**.
- [ ] Explain how a **Distributed Lock** works (`SET NX PX`, unique token, Lua unlock, Redisson watchdog).
- [ ] Explain distributed session management and JWT revocation using Redis.
- [ ] Contrast **Redis Pub/Sub** with **Apache Kafka** in system design interviews.

---

# 12. Level 14 Progress Tracker

| Part | Topic | Status |
|---|---|---|
| **Part 1** | **Apache Kafka — Core Architecture & Fundamentals** | Covered (Brokers, Topics, Partitions, Replication) |
| **Part 2** | **Redis — Caching, Locks, Rate Limiting & Pub/Sub** | Covered (this document) |
| **Part 3** | **Kafka Advanced — Consumer Groups, Offsets, DLQ & Idempotency** | Up Next |
| **Part 4** | **Distributed Systems — CAP, Consistency, Resiliency Patterns** | Not Started (Circuit Breaker, Retry, Backoff, Bulkhead, Backpressure) |