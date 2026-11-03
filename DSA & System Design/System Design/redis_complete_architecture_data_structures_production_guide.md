# Level 14 — Redis, Kafka & Distributed Systems
## Part 2: Redis — Complete Architecture, Data Structures, Caching & Spring Boot — Revision Notes

**Status:** Completed  
**Language / Stack:** Java + Spring Boot (`Spring Data Redis` / `Redisson` / `Lettuce`)  
**Source:** Level 14 Roadmap + [Redis Course - In-Memory Database Tutorial | freeCodeCamp](https://www.youtube.com/watch?v=XCsS_NVAa1g)  
**Focus:** Core data structures, HyperLogLog, Geospatial indexes, caching strategies, TTL, invalidation, distributed locks, rate limiting, sessions, Pub/Sub, Transactions (`WATCH`), Lua scripting, and Spring Boot production configs

---

# 1. Redis Fundamentals & Architecture

**Redis** (Remote Dictionary Server) is an open-source, **in-memory key-value data structure store** used as a database, distributed cache, message broker, and coordination engine.

### Why is Redis so fast?
1. **In-Memory Storage:** Data lives in RAM rather than on spinning disks or SSDs, yielding sub-millisecond latency ($< 1\text{ ms}$) and ~110,000 writes/sec or ~81,000 reads/sec on commodity hardware.
2. **Single-Threaded Event Loop:** Uses an I/O multiplexing model (`epoll`/`kqueue`) to process commands sequentially on a single main thread. This guarantees that individual commands execute **atomically** without context-switching or lock-contention overhead.
3. **Optimized Low-Level Data Structures:** Built in C using custom structures like Simple Dynamic Strings (SDS), Skip Lists, and Ziplists/Listpacks.
4. **Pipelining Support:** Clients can batch multiple commands in a single network round-trip to eliminate RTT latency bottlenecks.

**Default Port:** `6379`

---

# 2. The 5 Core Data Structures & CLI Cheat Sheet

| Data Type | Common Use Case | Key Commands | Time Complexity |
|---|---|---|---|
| **String** | Caching JSON/HTML, atomic counters, distributed locks | `GET`, `SET`, `MGET`, `INCR`, `SETEX` | $\mathcal{O}(1)$ |
| **List** | Activity feeds, blocking worker queues (FIFO/LIFO) | `LPUSH`, `RPUSH`, `LPOP`, `BLPOP`, `LRANGE` | $\mathcal{O}(1)$ at ends; $\mathcal{O}(N)$ range |
| **Set** | Unique tags, mutual friends, IP allowlists | `SADD`, `SISMEMBER`, `SINTER`, `SDIFF`, `SUNION` | $\mathcal{O}(1)$ add/check |
| **Sorted Set (ZSet)** | Real-time leaderboards, sliding-window rate limiters, priority queues | `ZADD`, `ZRANGE`, `ZREVRANGE`, `ZINCRBY` | $\mathcal{O}(\log N)$ |
| **Hash** | Storing structured domain objects (e.g., User profile fields, cart items) | `HSET`, `HGET`, `HMGET`, `HGETALL`, `HINCRBY` | $\mathcal{O}(1)$ per field; $\mathcal{O}(N)$ for all |

---

## 1. Strings (Text, Integers, Floats, Binary)
Can store strings, integers, floats, or serialized byte arrays (up to 512 MB).

| Command | Example | Behavior |
|---|---|---|
| `SET` / `GET` | `SET name "shabbir"` / `GET name` | Basic key-value read/write. |
| `GETRANGE` | `GETRANGE email 0 4` | Substring extraction (0-indexed inclusive). |
| `MSET` / `MGET` | `MSET lang "en" tech "redis"` | Atomic multi-key set/get in a single network round-trip. |
| `STRLEN` | `STRLEN name` | Returns string length (returns `0` if key doesn't exist). |
| `INCR` / `DECR` | `INCR count` / `DECR count` | Atomic $+1$ or $-1$ on integer strings. |
| `INCRBY` / `DECRBY` | `INCRBY count 10` | Atomic increment/decrement by integer delta. |
| `INCRBYFLOAT` | `INCRBYFLOAT pi 0.001` | Atomic increment for floating-point numbers. |
| `EXPIRE` / `TTL` | `EXPIRE a 10` / `TTL a` | Sets expiration in seconds. `TTL` returns remaining seconds, `-1` if no expiry set, or `-2` if key expired/deleted. |
| `SETEX` | `SETEX b 10 "val"` | Atomically sets value and TTL in one command. |

---

## 2. Lists (Doubly Linked Lists / Queues / Stacks)
Ordered sequence of strings allowing duplicates. Ideal for **task queues**, **recent activity feeds**, and **blocking worker queues**.

```text
LPUSH / LPOP  <-->  [ "UK", "USA", "India", "Australia" ]  <-->  RPUSH / RPOP
                    (Head / Index 0)          (Tail / Index -1)
```

| Command | Example | Behavior |
|---|---|---|
| `LPUSH` / `RPUSH` | `LPUSH country "India"` | Push element to head (left) or tail (right). |
| `LPUSHX` / `RPUSHX` | `LPUSHX movies "Avatar"` | Pushes **only if the list already exists** (returns `0` if key doesn't exist). |
| `LRANGE` | `LRANGE country 0 -1` | Returns elements from `start` to `stop` (`0 -1` returns all). |
| `LPOP` / `RPOP` | `LPOP country` | Removes and returns element from head or tail. |
| `BLPOP` / `BRPOP` | `BLPOP jobs 15` | **Blocking Pop:** Blocks client connection until an element is pushed to the list or timeout (`15s`) expires. Great for lightweight worker queues! |
| `LLEN` / `LINDEX` | `LLEN country` / `LINDEX country 3` | Get list length or element at specific index. |
| `LSET` / `LINSERT` | `LINSERT country BEFORE "USA" "UAE"` | Update index (`LSET`) or insert `BEFORE`/`AFTER` a pivot value. |
| `SORT` | `SORT country ALPHA DESC` | Sorts list numbers or strings (`ALPHA`) in `ASC`/`DESC` order. |

---

## 3. Sets (Unordered Unique Elements)
Stores unique strings only; duplicate additions are ignored and return `0`. Supports fast server-side set algebra.

| Command | Example | Behavior |
|---|---|---|
| `SADD` / `SMEMBERS` | `SADD tech "Java" "Redis"` | Adds unique members / lists all members. |
| `SCARD` / `SISMEMBER` | `SISMEMBER tech "Java"` | Returns cardinality (count) (`SCARD`) or `1`/`0` membership check in $\mathcal{O}(1)$. |
| `SDIFF` / `SDIFFSTORE` | `SDIFF tech frontend` | Elements in `tech` that are **not** in `frontend` (`SDIFFSTORE` saves result into a new set). |
| `SINTER` / `SINTERSTORE` | `SINTER user1:friends user2:friends` | **Intersection:** Common elements across sets (e.g., **Mutual Friends**). |
| `SUNION` / `SUNIONSTORE` | `SUNION tech frontend` | **Union:** Merges unique elements across multiple sets. |

---

## 4. Sorted Sets / ZSets (Score-Ordered Unique Elements)
Every unique member is associated with a 64-bit floating-point **score**. Implemented via a **Hash Table + Skip List**, keeping elements sorted automatically from lowest to highest score in $\mathcal{O}(\log N)$.
- **Primary System Design Use Cases:** Real-time **Leaderboards**, **Sliding-Window Rate Limiters**, and **Delayed Job Schedulers**.

| Command | Example | Behavior |
|---|---|---|
| `ZADD` | `ZADD leaderboard 100 "Shabbir" 250 "Alex"` | Adds members with their scores. |
| `ZRANGE` | `ZRANGE leaderboard 0 -1 WITHSCORES` | Returns members ordered lowest-to-highest rank (optionally with scores). |
| `ZREVRANGE` | `ZREVRANGE leaderboard 0 9 WITHSCORES` | Reverses order (highest-to-lowest rank — **Top 10 Leaderboard query**). |
| `ZREVRANGEBYSCORE` | `ZREVRANGEBYSCORE leaderboard 500 100` | Returns members in reverse order filtered by `max` and `min` score. |
| `ZCARD` / `ZCOUNT` | `ZCOUNT leaderboard 100 500` | Total count (`ZCARD`) or count of members within a `[min, max]` score range (`ZCOUNT`). |
| `ZSCORE` / `ZINCRBY` | `ZINCRBY leaderboard 50 "Shabbir"` | Gets member's score (`ZSCORE`) or atomically increments score (`ZINCRBY`). |
| `ZRANK` / `ZREVRANK` | `ZREVRANK leaderboard "Shabbir"` | Returns 0-based rank of a player (`ZREVRANK` gives rank from highest score down). |
| `ZREM` / `ZREMRANGEBYSCORE` | `ZREMRANGEBYSCORE rate_log 0 1700000` | Removes specific member or bulk removes members within a score/timestamp range. |

---

## 5. Hashes (Field-Value Maps for Domain Objects)
A map of string fields to string values—ideal for representing domain objects (`User`, `Account`, `Cart`) so you can read or update individual fields without deserializing and re-serializing an entire JSON blob.

| Command | Example | Behavior |
|---|---|---|
| `HSET` / `HSETNX` | `HSET user:101 name "Shabbir" tier "PRO"` | Sets fields (`HSETNX` sets **only if field does not exist**). |
| `HMGET` / `HGET` | `HMGET user:101 name tier` | Gets single (`HGET`) or multiple (`HMGET`) object fields in one command. |
| `HGETALL` / `HKEYS` / `HVALS` | `HGETALL user:101` | Returns all field-value pairs, only field names, or only values. |
| `HEXISTS` / `HLEN` | `HEXISTS user:101 email` | Checks if field exists (`1`/`0`) or counts number of fields. |
| `HINCRBY` / `HINCRBYFLOAT` | `HINCRBY user:101 login_count 1` | Atomically increments a numeric field inside a hash. |
| `HDEL` | `HDEL user:101 temp_token` | Deletes a specific field from the hash. |

---

# 3. Specialized Data Structures: HyperLogLog & Geospatial

## 1. HyperLogLog (Probabilistic Cardinality Estimation)

**Problem:** Count unique items (e.g., unique daily active IP addresses visiting YouTube, unique search queries) across **500 million events** without storing every ID in a massive `SET` that consumes gigabytes of RAM.

**Solution:** **HyperLogLog (HLL)** is a probabilistic data structure that estimates the cardinality (unique count) of a dataset with ~99.2% accuracy (0.81% standard error) using a tiny, fixed **12 KB of memory** per key.

```bash
# Add elements to HyperLogLog
PFADD page:home:visitors "user_1" "user_2" "user_3"
PFADD page:about:visitors "user_3" "user_4"

# Get estimated unique count of one or multiple HLLs
PFCOUNT page:home:visitors                        # Returns 3
PFCOUNT page:home:visitors page:about:visitors    # Returns 4 (combined unique union)

# Merge multiple HLLs (e.g., hourly unique visitors -> daily unique visitors)
PFMERGE page:daily:visitors page:home:visitors page:about:visitors
PFCOUNT page:daily:visitors
```

---

## 2. Geospatial Indexes (The Uber / Swiggy Proximity Pattern)

**How Redis Stores Locations Under the Hood:**
- Redis stores longitude and latitude coordinates inside a **Sorted Set (`ZSET`)**.
- Coordinates are converted into a **52-bit integer Geohash**, which becomes the `score` of the member in the Sorted Set.
- Uses the Haversine formula (spherical Earth model) with up to a ~0.5% error margin, measuring straight-line distance. Because it is backed by a `ZSET`, standard commands like `ZREM` work directly on geospatial keys.

```bash
# 1. Add locations: GEOADD <key> <longitude> <latitude> <member>
# CRITICAL: Order is strictly LONGITUDE first, then LATITUDE (X, Y)
GEOADD drivers:active 72.5714 23.0225 "driver_101" 72.8777 19.0760 "driver_202"

# 2. Get Geohash string or Lat/Long coordinates
GEOHASH drivers:active "driver_101"
GEOPOS drivers:active "driver_101"

# 3. Calculate straight-line distance between two members (Units: m, km, mi, ft)
GEODIST drivers:active "driver_101" "driver_202" km

# 4. Find nearby drivers within 5 km of a rider's coordinates (sorted closest first)
GEORADIUS drivers:active 72.5714 23.0225 5 km WITHDIST WITHCOORD ASC
# (Note: In Redis 6.2+, GEORADIUS is succeeded by GEOSEARCH)
```

---

# 4. Caching Strategies (Level 14 Roadmap Core)

Choosing how your Spring Boot service coordinates reads and writes between Redis and PostgreSQL is a staple backend interview discussion.

## Pattern 1: Cache-Aside (Lazy Loading) — *Production Default*

The application code explicitly orchestrates both the cache and the database.

```text
Read Path:
1. App checks Redis
2. Cache Hit?  --> Return data immediately (< 1ms)
3. Cache Miss? --> Query PostgreSQL --> Write to Redis (with TTL) --> Return data
```

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;

    public UserDto getUserById(Long userId) {
        String key = "user:" + userId;

        // 1. Check Cache
        UserDto cachedUser = (UserDto) redisTemplate.opsForValue().get(key);
        if (cachedUser != null) {
            return cachedUser; // Cache Hit
        }

        // 2. Cache Miss -> Load from PostgreSQL
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new EntityNotFoundException("User not found"));
        UserDto dto = UserDto.from(user);

        // 3. Populate Cache with TTL + Random Jitter (prevents Cache Avalanche)
        long jitterSeconds = ThreadLocalRandom.current().nextLong(0, 120);
        redisTemplate.opsForValue().set(key, dto, Duration.ofMinutes(15).plusSeconds(jitterSeconds));

        return dto;
    }
}
```

### Declarative Spring Cache Alternative (`@Cacheable` & `@CacheEvict`)
```java
@Cacheable(value = "users", key = "#userId", unless = "#result == null")
public UserDto getUserById(Long userId) {
    return userRepository.findById(userId).map(UserDto::from).orElse(null);
}

@Transactional
@CacheEvict(value = "users", key = "#userId")
public UserDto updateUser(Long userId, UpdateUserRequest request) {
    // Updates PostgreSQL and automatically evicts "users::<userId>" from Redis!
}
```

## Caching Patterns Comparison

| Pattern | How it Works | Pros | Cons |
|---|---|---|---|
| **Cache-Aside** | App reads/writes cache & DB explicitly | Only requested data is cached; resilient if Redis goes down | Cache miss penalty on cold start (3 network hops) |
| **Read-Through** | Cache sits inline; cache provider loads from DB on miss | Cleaner application code | Requires cache provider plugin support |
| **Write-Through** | App writes to Cache, and Cache synchronously writes to DB | Cache is never stale | Higher write latency (every write hits both systems); caches unread data |
| **Write-Behind (Write-Back)** | App writes to Cache; Cache flushes to DB asynchronously in batches | Extremely fast writes; absorbs massive write spikes | **Risk of data loss** if Redis crashes before flushing to PostgreSQL |

---

# 5. TTL, Eviction Policies & Cache Invalidation

### 1. TTL (Time-To-Live)
Every cached key in production must have a **TTL** (`EXPIRE` / `PEXPIRE`).
- Acts as a self-healing boundary against stale data.
- Prevents unbounded RAM growth (`OutOfMemory` errors).

### 2. Cache Invalidation on Writes: Why Delete Instead of Update?

> **Golden Rule:** On a database write (`UPDATE` / `DELETE`), **delete (`DEL`)** the cache key rather than updating it.

**Why?** Suppose Request A and Request B concurrently update the same user:
```text
1. Request A updates PostgreSQL (Balance = $100)
2. Request B updates PostgreSQL (Balance = $250)  <-- Latest truth in DB is $250
3. Due to network jitter, Request B updates Redis FIRST (Redis = $250)
4. Request A's Redis update arrives SECOND (Redis = $100)  <-- Cache is now permanently stale!
```
Deleting the key (`DEL user:101`) avoids race-condition overwrites and forces the next read to cleanly repopulate from PostgreSQL.

### 3. Production Cache Failure Modes & Mitigations

| Failure Mode | What Happens | Solution / Mitigation |
|---|---|---|
| **Cache Stampede (Thundering Herd)** | A popular key expires; 10,000 concurrent requests miss the cache at the exact same millisecond and hammer PostgreSQL simultaneously. | 1. **Distributed Mutex Lock** on cache rebuild (only 1 thread queries DB, others wait).<br>2. **Logical early expiration** (refresh asynchronously before TTL expires). |
| **Cache Penetration** | Malicious or buggy requests query non-existent IDs (`user:-999`). Misses Redis and hits PostgreSQL every single time. | 1. **Cache Null/Empty sentinel values** with a short TTL (`30–60s`).<br>2. Place a **Bloom Filter** in front of Redis to reject non-existent IDs in memory. |
| **Cache Avalanche** | Thousands of keys loaded during a batch job with the exact same `15m` TTL expire at the exact same second, overwhelming the DB. | **Add Random Jitter** to every TTL: $\text{TTL} = \text{Base\_TTL} + \text{random}(0, 300\text{s})$. |
| **Hot Key Problem** | A single key (e.g., celebrity tweet or flash-sale item) receives 500,000 reads/sec, saturating the network NIC of a single Redis shard. | 1. Add a local JVM L1 cache (**Caffeine**) with a 2–5 second TTL in front of Redis.<br>2. Replicate the key across multiple shards (`item:101#1`, `item:101#2`). |

### 4. Memory Eviction Policies (`maxmemory-policy`)
- **`allkeys-lru` / `volatile-lru`:** Evicts Least Recently Used keys (standard for general caching).
- **`allkeys-lfu` / `volatile-lfu`:** Evicts Least Frequently Used keys (retains consistently popular keys).
- **`noeviction`:** Returns errors on writes when RAM is full (used when Redis stores non-evictable state like locks or queues).

---

# 6. Atomic Counters & Rate Limiting

Because Redis executes commands sequentially on its single-threaded event loop, `INCR` and `DECR` are **100% atomic** without database row lock contention.

## Pattern 1: Fixed-Window Rate Limiter

Limit a user to $N$ requests per minute using `INCR` + `EXPIRE`.

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
- **Pros:** $\mathcal{O}(1)$ time and space; tiny memory footprint (1 integer per user per minute).
- **Trade-off (Boundary Burst):** A user can send $100$ requests at `12:00:59` and another $100$ requests at `12:01:01` ($200$ requests across $2$ seconds).

---

## Pattern 2: Sliding-Window Log Rate Limiter (Sorted Set + Lua Script)

Solves the boundary burst problem by storing exact request timestamps inside a Redis **Sorted Set (`ZSET`)** and executing all steps atomically via a **Lua Script**:

```lua
-- KEYS[1] = rate limit key (e.g., "rate_limit:sliding:user_42")
-- ARGV[1] = current timestamp in ms (now)
-- ARGV[2] = window size in ms (e.g., 60000)
-- ARGV[3] = max allowed requests (limit)
-- ARGV[4] = unique request UUID

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local reqId = ARGV[4]

-- 1. Evict timestamps older than (now - window)
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- 2. Count remaining requests inside the current sliding window
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    -- 3. Allowed: Add current timestamp and reset key TTL
    redis.call('ZADD', key, now, reqId)
    redis.call('PEXPIRE', key, window)
    return 1
else
    -- 4. Rejected (429 Too Many Requests)
    return 0
end
```
- **Pros:** 100% accurate rolling window; zero boundary burst.
- **Cons:** Higher memory footprint ($\mathcal{O}(N)$ entries stored per user in the ZSet). For massive scale, **Token Bucket** (storing `tokens_left` and `last_refill_ts` in a Redis Hash via Lua) is also popular.

---

# 7. Transactions (`WATCH`) & Lua Scripting Engine

## 1. Redis Transactions & Optimistic Locking (`MULTI`, `EXEC`, `DISCARD`, `WATCH`)

- **`MULTI`:** Enters transaction mode. Subsequent commands are **not executed immediately**—they are placed in a `QUEUED` state.
- **`EXEC`:** Executes all queued commands sequentially and atomically in a single step.
- **`DISCARD`:** Flushes the transaction queue and exits transaction mode without executing anything.

### Important Distinction: Redis Transactions vs. PostgreSQL ACID Transactions
> **Critical Interview Nuance:** Redis transactions **do NOT support rollbacks**! If command #2 in a `MULTI/EXEC` block fails at runtime (e.g., running `INCR` on a list key), command #1 remains committed and command #3 still executes.

### Optimistic Locking with `WATCH` (Check-And-Set / CAS)
What if another client modifies a `balance` key after we read it, before our `MULTI/EXEC` runs?
- Calling `WATCH key` before `MULTI` monitors the key for modifications by any other client.
- If another client modifies `key` before our client calls `EXEC`, the **entire transaction aborts cleanly and `EXEC` returns `(nil)`**.

```text
Client 1                                  Client 2
--------                                  --------
WATCH wallet:101
val = GET wallet:101 ($100)
MULTI                                     SET wallet:101 $50  <-- Modified by Client 2!
SET wallet:101 $80 -> QUEUED
EXEC  ---> Returns (nil)! Transaction aborted safely; Client 1 can retry.
```

---

## 2. Server-Side Lua Scripting (`EVAL` & `EVALSHA`)

### Why Prefer Lua Scripts Over `WATCH/MULTI/EXEC` in Production?
1. **True Conditional Atomicity:** In `MULTI/EXEC`, you cannot inspect the result of step 1 (`GET`) inside the queue to decide whether to run step 2 (`SET`). In a **Lua Script**, you can read keys, run `if/else` logic, and write keys atomically on the server.
2. **Zero Contention Retries:** Unlike optimistic locking (`WATCH`), which aborts and forces retries under high contention, a Lua script executes serially without aborting.
3. **Single Network Round-Trip:** Sending a cached SHA1 hash via `EVALSHA` executes multi-step workflows in 1 hop.

```bash
# Load script into Redis cache once to get a SHA1 digest
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# Execute cached script by SHA1
EVALSHA "4e6d8fc8bb01276962cce5371fa795a7763657ae" 1 user:101
```
- **Production Rule:** Because Lua scripts block the single-threaded Redis event loop while running, keep scripts tiny and fast (default `lua-time-limit` timeout is **5 seconds**).

---

# 8. Distributed Locks & Concurrency Control

When running 5 instances of a Spring Boot microservice behind a load balancer, Java's `synchronized` or `ReentrantLock` only coordinates threads inside a **single JVM**. To prevent two pods from executing the same cron job or processing the same transfer concurrently, use a **Redis Distributed Lock**.

## 1. Low-Level Pattern: `SET key value NX PX milliseconds`

- **`NX` (Not Exists):** Only acquire the lock if the key does not already exist.
- **`PX` (Milliseconds TTL):** Automatically expire the lock after a lease timeout so a crashed pod never holds the lock forever (deadlock prevention).
- **`value` (`UUID`):** Must be a unique token per thread/request so a pod only unlocks its **own** lock.

### Safe Lock Release (Why a Lua Script is Mandatory)
**Never** do `GET` in Java followed by `DEL` in Java to release a lock:
```text
1. Pod A acquires lock with 10s TTL.
2. Pod A hits a 12s GC pause or slow network call -> Lock expires at t=10s!
3. Pod B acquires the lock at t=10.1s.
4. Pod A wakes up at t=12s and calls DEL -> Pod A just deleted Pod B's active lock!
```
Always use an atomic Lua script to verify the owner `UUID` before deleting:
```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

---

## 2. Production Spring Boot Implementation with Redisson (`RLock`)

In production Spring Boot systems, use **Redisson** rather than hand-rolling `SET NX` locks:
- **Watchdog Auto-Renewal:** If a thread is still alive and working, Redisson's background watchdog automatically extends the lock's TTL every 10 seconds (default 30s lease), preventing premature expiration during slow processing—while still expiring automatically if the pod crashes!

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TransferLockService {

    private final RedissonClient redissonClient;
    private final TransferService transferService;

    public void executeLockedTransfer(String accountId, TransferRequest request) {
        RLock lock = redissonClient.getLock("lock:account:" + accountId);

        try {
            // Wait up to 2s to acquire lock; leaseTime = -1 enables the Watchdog auto-renewal!
            boolean acquired = lock.tryLock(2, -1, TimeUnit.SECONDS);
            if (!acquired) {
                throw new ConcurrentModificationException("Account is currently busy");
            }

            // Critical Section (still enforce DB-level @Version optimistic locking / idempotency!)
            transferService.processTransfer(request);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Lock acquisition interrupted", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

> **Senior Architecture Warning (Fencing Tokens & GC Pauses):**  
> Redis replication is **asynchronous**—if a Redis Master crashes right after granting a lock before replicating it to a Replica, a second client could acquire the same lock from the newly promoted Master. For financial ledgers, treat Redis Distributed Locks as an **efficiency optimization** to reduce contention, and **always enforce final correctness at the PostgreSQL layer** using ACID transactions, `@Version` optimistic locking, or unique `idempotency_key` constraints!

---

# 9. Distributed Sessions & Authentication State

In a stateless microservice architecture, storing session state in local server RAM (`HttpSession`) breaks when a load balancer routes a user's next request to a different pod.

```text
User Request --> Load Balancer --> Spring Boot Pod 1, 2, or 3
                                           ↓
                                  Central Redis Cluster
                             (Session Hash / JWT Denylist)
```

### Two Production Auth Patterns with Redis
1. **Stateful Server Sessions (`Spring Session Data Redis`):**
   - Browser holds an `HttpOnly`, `Secure` cookie containing an opaque `SESSIONID`.
   - Redis stores user principals, roles, and CSRF metadata in a Redis Hash with a sliding TTL (`30m`).
   - **Instant Revocation:** Deleting the session key in Redis immediately logs the user out across all devices.
2. **Stateless JWT + Redis Refresh Token & Denylist (Hybrid Pattern):**
   - **Access Token (JWT):** Short-lived (`5–15 mins`), verified statelessly via cryptographic signature.
   - **Refresh Token:** Stored in Redis (`refresh_token:<userId>:<deviceId>`) with a `7d` TTL, rotated on every use.
   - **Immediate JWT Revocation (Logout / Ban):** When a user logs out or compromises their token, write `jwt:denylist:<jti>` to Redis with a TTL equal to the remaining seconds before the JWT naturally expires. Every API Gateway / Security Filter checks `EXISTS jwt:denylist:<jti>` in $< 1\text{ ms}$.

---

# 10. Redis Pub/Sub vs. Apache Kafka

Redis provides a lightweight **Publish/Subscribe** messaging broker supporting direct channels (`SUBSCRIBE`, `PUBLISH`) and glob-style wildcard pattern subscriptions (`PSUBSCRIBE news*`, `h?llo`, `b[ai]ll`) plus introspection (`PUBSUB CHANNELS`, `PUBSUB NUMSUB`, `PUBSUB NUMPAT`).

### Critical System Design Comparison: Redis Pub/Sub vs. Apache Kafka

| Feature | Redis Pub/Sub | Apache Kafka |
|---|---|---|
| **Message Persistence** | **None (Fire-and-Forget).** Pushed directly through RAM sockets; never written to disk. | **Persistent on Disk** (Immutable append-only log with configurable retention). |
| **Offline Consumers** | If a subscriber disconnects for 2 seconds, **all messages published during those 2 seconds are lost forever**. | Consumers track offsets and read all missed messages whenever they come back online. |
| **Delivery Guarantee** | At-most-once (Push-based) | At-least-once / Effectively-once (Pull-based) |
| **Consumer Routing** | **Pure Fan-out Broadcast:** Every active subscriber on a channel gets a copy. *(Note: Redis Streams adds consumer groups).* | **Consumer Groups** support both partitioned load-balancing (intra-group) and broadcast fan-out (inter-group). |
| **Backpressure & Backlog** | Slow subscribers cause Redis output buffers to swell and get forcefully disconnected. | Handles terabytes of backlog on disk without hurting broker RAM. |
| **Best System Design Use Case** | **Ephemeral real-time events:** WebSocket chat routing across pods, live typing indicators, broadcasting L1 cache invalidation signals. | **Durable business events:** Payment processing, order pipelines, audit logs, microservice choreography. |

---

# 11. Persistence, High Availability, Clustering & Security

## 1. Persistence Options (How In-Memory Data Survives Restarts)

| Mechanism | How It Works | Trade-offs |
|---|---|---|
| **RDB (Snapshot)** | Forks a background child process (`BGSAVE`) to write a compact binary point-in-time snapshot (`dump.rdb`) every $X$ minutes/writes. | Fast restarts, tiny file size, zero impact on main thread I/O; **loses data written since the last snapshot** (e.g., last 5 mins). |
| **AOF (Append-Only File)** | Logs every write command sequentially to disk (`appendonly.aof`), typically synced every 1 second (`appendfsync everysec`). | **High durability** (at most 1 second of data loss); larger files and slower boot replay unless compacted (`BGREWRITEAOF`). |
| **RDB + AOF Hybrid** | Combines an RDB base preamble with incremental AOF tail logs. | **Production Standard** when Redis is used for persistent state (sessions, rate limits, queues). |

---

## 2. High Availability & Scaling Topologies

1. **Redis Sentinel (High Availability without Sharding):**
   - 1 Primary (Master) + $N$ Read Replicas + Sentinel monitor nodes.
   - If the Primary fails, Sentinel nodes reach quorum, promote a Replica to Primary, and notify Spring Boot clients (`Lettuce`) automatically.
   - Use when your entire dataset fits comfortably inside a single machine's RAM (e.g., $< 32\text{ GB}$).
2. **Redis Cluster (Horizontal Sharding + High Availability):**
   - Automatically partitions data across multiple Primary shards using **16,384 Hash Slots**:
     $$\text{Hash Slot} = \text{CRC16}(\text{key}) \bmod 16384$$
   - Each Primary shard has its own Replicas for failover.
   - **Multi-Key Hash Tags (`{...}`):** In Redis Cluster, multi-key operations (`MGET`, Lua scripts, transactions) require all keys to live on the **same shard**. Use curly braces to force related keys into the same hash slot: e.g., `{user:101}:profile` and `{user:101}:settings` both hash only the substring `user:101`!
   - **Logical DB Note:** Standalone Redis supports 16 isolated logical namespaces (`SELECT 0`..`15`), whereas **Redis Cluster strictly supports only `SELECT 0`**.

---

## 3. Administration, Security & Benchmarking Commands

- **Health & Client Inspection:** `PING` (`PONG`), `CLIENT LIST`, `CLIENT SETNAME <pod-name>`, `CLIENT KILL ID <id>`.
- **Authentication:** `CONFIG SET requirepass <secret>` and `AUTH <secret>` (or Redis 6+ ACLs with role-based user permissions).
- **Load Benchmarking (`redis-benchmark`):**
  ```bash
  # Simulate 200 concurrent clients (-c), 100,000 requests (-n), 1KB payload (-d)
  redis-benchmark -h 127.0.0.1 -p 6379 -c 200 -n 100000 -d 1000
  ```

---

# 12. Spring Boot Production Redis Configuration

By default, `RedisTemplate` uses `JdkSerializationRedisSerializer`, which writes unreadable binary payloads and breaks across language/class changes. **Always configure `GenericJackson2JsonRedisSerializer` and `StringRedisSerializer` in production!**

```java
@Configuration
@EnableCaching
public class RedisProductionConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        StringRedisSerializer stringSerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer jsonSerializer = new GenericJackson2JsonRedisSerializer();

        // Keys as clean UTF-8 strings; values as readable JSON
        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

---

# 13. Pattern Recognition & Interview Cheat Sheet

| If the interviewer asks… | Think… |
|---|---|
| Our PostgreSQL database CPU is at 95% due to repetitive read queries on product pages. | **Cache-Aside pattern** with Redis + randomized **TTL jitter** to prevent Cache Avalanche. |
| What happens if a celebrity's cached profile expires and 50,000 requests hit the DB at once? | **Cache Stampede (Thundering Herd)**. Solve with a **Distributed Mutex Lock** (`Redisson`) on cache rebuild or logical early refresh. |
| Attackers are querying random non-existent `userId`s, bypassing Redis and overloading PostgreSQL. | **Cache Penetration**. Cache `null` sentinel values with a 30s TTL or place a **Bloom Filter** in front of Redis. |
| How do you design a real-time gaming leaderboard for 10 million players? | Redis **Sorted Set (`ZSET`)**: `ZINCRBY` for updating scores in $\mathcal{O}(\log N)$, `ZREVRANGE 0 9` for top 10, `ZREVRANK` for a player's exact rank. |
| How do you count unique daily visitors across 500M page views using minimal memory? | Redis **HyperLogLog** (`PFADD`, `PFCOUNT`, `PFMERGE`) — estimates unique cardinality in just **12 KB** of RAM. |
| How do you find nearby drivers or restaurants within 5 km in real time (Uber/Swiggy)? | Redis **Geospatial Index** (`GEOADD` with Longitude then Latitude, `GEORADIUS` / `GEOSEARCH`) backed by a 52-bit Geohash Sorted Set. |
| How do you prevent a user from calling an API more than 100 times per minute? | Redis **Rate Limiter**: Fixed Window (`INCR` + `EXPIRE`) or Sliding Window (`ZSET` + atomic **Lua script**). |
| How do we prevent two pods of our Payment Worker from running a job or transfer simultaneously? | **Redis Distributed Lock** (`SET NX PX` + UUID + Lua unlock, or **Redisson `RLock` with Watchdog**) paired with **PostgreSQL idempotency**! |
| How do we revoke a compromised JWT immediately before its 15-minute expiration? | Store `jwt:denylist:<jti>` in Redis with a TTL matching the token's remaining lifetime. |
| Should we use Redis Pub/Sub or Kafka for payment/order events? | **Kafka**. Redis Pub/Sub is fire-and-forget in RAM; if a worker pod restarts, events are permanently lost. |
| How do we route real-time WebSocket chat messages when User A is connected to Pod 1 and User B is on Pod 4? | **Redis Pub/Sub**. Pod 1 publishes to `chat:userB`, and Pod 4 (subscribed to `chat:userB`) pushes the frame down User B's WebSocket. |

---

# 14. Common Redis Mistakes

- **Setting Keys Without a TTL:** Slowly leaks RAM until Redis hits `maxmemory` and starts evicting critical keys or throwing `OOM` errors.
- **Using `KEYS *` in Production:** `KEYS *` scans the entire keyspace in $\mathcal{O}(N)$ on Redis's single thread, **freezing all other requests across your entire company** for seconds! Always use cursor-based `SCAN` instead.
- **Leaving Default Java Serialization in `RedisTemplate`:** Writes bloated binary keys (`\xac\xed\x00\x05...`) that cannot be inspected via `redis-cli` or shared across services. Always configure `StringRedisSerializer` + JSON.
- **Updating Cache Instead of Deleting on DB Write:** Leads to concurrent write race conditions that leave stale data in Redis until TTL expires. Always **delete (`DEL` / `@CacheEvict`)** on writes.
- **Assuming `MULTI/EXEC` Rolls Back on Error like SQL:** Redis transactions do **not** roll back prior commands when a runtime error occurs mid-batch.
- **Unlocking a Distributed Lock Without Checking the Owner UUID:** Allows a delayed thread whose lock expired to accidentally delete another thread's active lock.
- **Storing Massive Blobs ("Big Keys"):** Storing a 20 MB JSON array or a `SET` with 5 million items blocks the single-threaded event loop during network serialization or deletion (use `UNLINK` instead of `DEL` for asynchronous background memory freeing of large keys!).

---

# 15. Revision Checklist

## Core Data Structures & Specialized Types
- [x] Explain why Redis is fast (RAM, single-threaded event loop, I/O multiplexing, custom C structures).
- [x] Use all 5 core data structures (`String`, `List` with `BLPOP`, `Set` algebra, `Sorted Set` leaderboards, `Hash` objects).
- [x] Explain **HyperLogLog** (`PFADD`/`PFCOUNT`) for 12 KB unique visitor counting.
- [x] Explain **Geospatial Indexes** (`GEOADD`/`GEORADIUS`) and how Geohash maps onto a `ZSET`.

## Caching, Coordination & Production Spring Boot
- [x] Implement **Cache-Aside** in Spring Boot and explain why deleting a cache key on write beats updating it.
- [x] Differentiate **Cache Stampede**, **Cache Penetration**, **Cache Avalanche**, and **Hot Keys** and state their fixes.
- [x] Implement Fixed-Window (`INCR`+`EXPIRE`) and Sliding-Window (`ZSET`+Lua) **Rate Limiters**.
- [x] Contrast Redis **`WATCH/MULTI/EXEC`** optimistic locking with atomic **Lua Scripts** (`EVAL`/`EVALSHA`).
- [x] Implement a safe **Distributed Lock** (`SET NX PX` + Lua unlock, or **Redisson Watchdog**) and explain why DB idempotency is still required.
- [x] Contrast **Redis Pub/Sub** vs. **Apache Kafka**, **RDB** vs. **AOF**, and **Sentinel** vs. **Redis Cluster** (16,384 hash slots).

---

# 16. Level 14 — Redis Progress Tracker

| Topic | Status | Notes |
|---|---|---|
| **Core Data Structures & CLI** | Covered | Strings, Lists (`BLPOP`), Sets (`SINTER`), ZSets (Leaderboards), Hashes |
| **HyperLogLog & Geospatial** | Covered | `PFADD`/`PFCOUNT` (12 KB cardinality), `GEOADD`/`GEORADIUS` (52-bit Geohash ZSet) |
| **Caching, TTL & Cache-Aside** | Covered | Lazy loading, Write-Through/Behind, TTL jitter, `@Cacheable` / `@CacheEvict` |
| **Cache Invalidation & Failure Modes** | Covered | Delete-on-write rule, Stampede, Penetration, Avalanche, Hot Keys, `UNLINK` |
| **Atomic Counters & Rate Limiting** | Covered | Fixed Window (`INCR`) vs. Sliding Window (`ZSET` + atomic Lua script) |
| **Transactions (`WATCH`) & Lua Scripts** | Covered | `MULTI`/`EXEC` no-rollback caveat, `WATCH` CAS, `EVAL`/`EVALSHA` atomicity |
| **Distributed Locks** | Covered | `SET NX PX` + UUID + Lua release, Redisson `RLock` Watchdog, Fencing/Idempotency |
| **Distributed Sessions & JWT Revocation** | Covered | `Spring Session Data Redis` & stateless JWT denylist pattern |
| **Redis Pub/Sub vs. Kafka** | Covered | `PUBLISH`/`PSUBSCRIBE` wildcards, ephemeral fan-out vs. durable Kafka logs |
| **Persistence, Cluster & Spring Boot Config** | Covered | RDB vs. AOF, Sentinel vs. Cluster (16,384 slots & `{hash_tags}`), `RedisTemplate` JSON config |