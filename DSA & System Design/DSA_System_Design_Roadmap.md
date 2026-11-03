# Backend/Full-Stack Career Roadmap

**Goal:** Transition from Senior Frontend Engineer → Senior Backend / Backend-leaning Full-Stack Engineer

Your existing Java + Spring Boot roadmap is now the foundation.

```text
JAVA + SPRING BOOT
       |
       ▼
  DSA / Coding
       |
       +-----------------------+
       ▼                       ▼
 System Design           Backend Depth
       |                       |
       +-----------+-----------+
                   |
                   ▼
          Distributed Systems
                   |
                   ▼
          Production Project
                   |
                   ▼
         Interview Preparation
                   |
                   ▼
      Senior Backend / Full-Stack
```

---

## New Roadmap

### Level 11 - DSA & Problem Solving
**Status:** 🟡 In Progress (Currently on Part 5: Graphs)

**Focus:**
*   Big-O analysis
*   Arrays
*   HashMap / HashSet
*   Strings
*   Two Pointers
*   Sliding Window
*   Prefix Sum
*   Stack / Monotonic Stack
*   Queue / Deque
*   Binary Search
*   Linked List
*   Trees / BST
*   Heap / Priority Queue
*   Graphs (BFS/DFS, Topological Sort, Union Find)
*   Recursion / Backtracking
*   Greedy
*   Dynamic Programming
*   Intervals

**Goal:** Not: "Solve 500 LeetCode problems." Instead: Recognize the pattern and solve a medium problem confidently in Java.

**Learning format:**
Pattern → Intuition → Java implementation → Time/Space complexity → 2-4 problems → Recognition cues

---

### Level 12 - System Design Fundamentals
**Status:** 🔴 Not Started

**Focus:**
*   Functional vs non-functional requirements
*   Scalability / Availability / Reliability
*   Latency / Throughput
*   Load balancing
*   Horizontal vs vertical scaling
*   Stateless services
*   Caching / CDN
*   Database selection (SQL vs NoSQL)
*   Message queues
*   API Gateway / Service discovery
*   Basic microservices

**Design problems to start with:**
1. URL Shortener
2. Rate Limiter
3. File Storage
4. Notification System
5. Job Scheduler
6. Chat System
7. Food Delivery
8. Payment System

---

### Level 13 - Advanced PostgreSQL & Database Engineering
**Status:** 🟢 Completed *(SQL Optimization Techniques)*

**Focus:**
*   B-Tree indexes / Composite indexes
*   Covering indexes / Partial indexes
*   Query optimization (`EXPLAIN`, `EXPLAIN ANALYZE`)
*   Sequential vs Index Scan
*   Joins / Query planner
*   Transactions / ACID / Isolation levels
*   Row locks / Deadlocks
*   Optimistic locking / Pessimistic locking
*   MVCC
*   Connection pooling
*   Read replicas / Partitioning / Sharding
*   Database migrations

**Important practical question:** Why is this query slow even though I have an index? (You should be able to investigate this yourself).

---

### Level 14 - Redis, Kafka & Distributed Systems
**Status:** 🟢 Completed *(Redis & Kafka done; Distributed Systems pending)*

**Redis (Completed):**
*   Caching / TTL / Cache invalidation
*   Cache-aside
*   Distributed locks
*   Counters / Rate limiting
*   Sessions
*   Pub/Sub

**Kafka (Completed):**
*   Producer → Topic → Partition → Consumer Group → Consumer
*   Ordering / Offset
*   Retry / Dead-letter queues
*   At-least-once delivery / Idempotency
*   Event-driven architecture

**Distributed systems (Pending):**
*   CAP theorem
*   Consistency / Eventual consistency
*   Distributed locking
*   Idempotency / Retry / Exponential backoff
*   Circuit breaker
*   Bulkhead / Timeout / Backpressure

---

### Level 15 - Production Spring Boot
**Status:** 🟢 Completed

**Focus:**
"Can I run and maintain this application in production?"
*   Spring Security / JWT / OAuth2 concepts / RBAC
*   API versioning / Validation / Global exception handling
*   Transactions
*   Async processing / Scheduling
*   Configuration management / Profiles
*   Actuator / Logging / Metrics
*   Distributed tracing (OpenTelemetry)
*   Testing (Unit, Integration, E2E, Testcontainers)
*   Docker

---

### Level 16 - Production Backend Project
**Status:** 🟢 Completed

**Focus:** Build a serious Transaction / Payment Platform.

**Features:**
*   Authentication / Authorization
*   Users / Accounts / Transfers / Transaction history
*   Idempotency / Concurrency control / Database transactions
*   Redis caching / Kafka events / Retry
*   Audit logs / Pagination / Filtering / Rate limiting
*   Docker / Monitoring / Tests

This becomes your portfolio + interview project.

---

### Level 17 - System Design Interview Preparation
**Status:** 🔴 Not Started

**Focus:** Move from "What is Kafka?" to "Why would I use Kafka here?"
Every design should follow:
1. Requirements
2. Capacity estimation
3. API design
4. Data model
5. High-level architecture
6. Detailed components
7. Scaling
8. Caching
9. Async processing
10. Failure handling
11. Monitoring
12. Trade-offs

**Problems:** URL Shortener, Instagram, WhatsApp, Uber, Swiggy/Zomato, YouTube, Netflix, Payment system, Notification system, Distributed job scheduler, File storage.

---

### Level 18 - Senior Backend / Full-Stack Interview Preparation
**Status:** 🔴 Not Started

**Focus:**
*   **DSA:** Timed problems, medium-heavy preparation, pattern recognition, mock rounds.
*   **Backend:** Questions around Spring Boot, Java, PostgreSQL, Redis, Kafka, Transactions, Concurrency, REST, Security, Testing.
*   **System Design:** Timed 45-60 minute designs.
*   **Behavioral:** Prepare stories around architecture decisions, performance improvements, production incidents, difficult technical decisions, mentoring, ownership, conflict, project failures, cross-team collaboration.

*(Your existing frontend experience becomes a strength here, not something to hide).*

---

### Level 19 - Full-Stack Architecture & Career Positioning
**Status:** 🔴 Not Started

**Focus:** Deliberately maintain your frontend advantage.
*   React architecture / API integration / Authentication flows
*   Frontend caching / SSR/CSR concepts / Web performance
*   Backend-for-Frontend / API contracts
*   End-to-end architecture (Browser → Frontend → API Gateway → Spring Boot → Redis/PostgreSQL/Kafka → Workers)
*   Frontend/backend observability
*   Full-stack deployment

---

## Final Progress Tracker

**Java + Spring Foundation**
| Area | Status |
| :--- | :--- |
| Level 1: Java Basics | 🟢 Completed |
| Level 2: Collections & Streams | 🟢 Completed |
| Level 3: OOP / Core Java | 🟢 Completed |
| Level 4: Exceptions / Generics | 🟢 Completed |
| Level 5: Modern Java | 🟢 Completed |
| Level 6: Concurrency | 🟢 Completed |
| Level 7: JVM Internals | 🟢 Completed |
| Level 8: Backend Fundamentals | 🟢 Completed |
| Level 9: Spring Boot | 🟢 Completed |
| Level 10: Production Backend Project | 🟢 Completed |

**New Track**
| Area | Status |
| :--- | :--- |
| Level 11: DSA | 🟡 In Progress |
| Level 12: System Design Fundamentals | 🔴 Not Started |
| Level 13: PostgreSQL / DB Internals | 🟢 Completed |
| Level 14: Redis / Kafka / Distributed | 🟢 Completed (Dist. Sys Pending) |
| Level 15: Production Spring Boot | 🟢 Completed |
| Level 16: Production Backend Project | 🟢 Completed |
| Level 17: System Design Interviews | 🔴 Not Started |
| Level 18: Senior Interview Prep | 🔴 Not Started |
| Level 19: Full-Stack Architecture | 🔴 Not Started |