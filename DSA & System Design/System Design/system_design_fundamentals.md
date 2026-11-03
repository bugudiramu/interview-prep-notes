# Level 12 — System Design Fundamentals

**Status:** Completed  
**Focus:** Core system design terminology, scaling strategies, infrastructure components, database selection, and foundational architecture patterns.

---

# 1. System Requirements

Every system design interview begins by clarifying what you are building. Never start designing without locking these down first.

| Requirement Type | Definition | Examples |
| :--- | :--- | :--- |
| **Functional Requirements** | **What** the system must do. The core business features from a user's perspective. | "Users can upload a photo."<br>"Users can send a message."<br>"System generates a short URL." |
| **Non-Functional Requirements** | **How** the system performs. The architectural qualities and constraints. | "System must be highly available (99.99%)."<br>"Latency must be < 200ms."<br>"System is read-heavy (100:1 read/write ratio)." |

---

# 2. Core Metrics & System Characteristics

### Latency vs. Throughput
*   **Latency:** The time it takes for a *single request* to yield a response. Measured in milliseconds (ms). (e.g., "It takes 50ms to load the profile page.")
*   **Throughput:** The *volume of requests* a system can handle over a specific timeframe. Measured in Requests Per Second (RPS) or Queries Per Second (QPS). (e.g., "Our API handles 10,000 QPS.")
*   *Analogy:* Latency is how fast a car drives; Throughput is how many lanes the highway has.

### Scalability, Availability, and Reliability
*   **Scalability:** The ability of a system to handle increased load by adding resources without degrading performance.
*   **Reliability:** The probability that a system will produce correct, expected results (it doesn't lose your data or calculate the wrong balance).
*   **Availability:** The percentage of time a system is operational and accessible.
    *   *Two 9s (99%):* ~3.6 days of downtime per year.
    *   *Three 9s (99.9%):* ~8.7 hours of downtime per year.
    *   *Four 9s (99.99%):* ~52 minutes of downtime per year. (Standard for enterprise systems).
    *   *Five 9s (99.999%):* ~5 minutes of downtime per year. (Telecomm / pacemakers).

---

# 3. Scaling Strategies & Load Balancing

## Horizontal vs. Vertical Scaling

| Feature | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
| :--- | :--- | :--- |
| **Definition** | Adding more CPU/RAM to a single existing server. | Adding more physical or virtual servers to a pool. |
| **Limits** | Has a hard hardware limit. | Virtually infinite (Cloud computing). |
| **Availability** | Single Point of Failure (SPOF). | Highly Available (if one node dies, others take over). |
| **Complexity** | Extremely simple (no code changes needed). | Complex (requires Load Balancers, distributed state). |
| **Use Case** | Small internal tools, monolithic databases. | Modern web applications, microservices. |

## Stateless Services
To successfully scale horizontally, your application servers **must be stateless**.
*   **Stateful:** Server 1 remembers that User A logged in. If User A's next request goes to Server 2, they appear logged out.
*   **Stateless:** No server stores local session data. Session data is pushed down to a centralized, external store (like **Redis**). Any server can handle any request from any user at any time.

## Load Balancing
A Load Balancer (LB) sits in front of your servers and routes incoming traffic to healthy instances.
*   **Algorithms:** Round Robin (sequential), Least Connections (sends to the least busy server), IP Hashing (routes same user to same server).
*   **L4 vs L7:** Layer 4 LBs route based on IP/TCP ports (extremely fast). Layer 7 LBs route based on HTTP headers/URL paths (smarter routing, e.g., `/api/payments` goes to the Payment Service cluster).

---

# 4. Core Infrastructure Components

### 1. API Gateway & Service Discovery
*   **API Gateway:** The single entry point for all clients. Handles cross-cutting concerns: **Routing, Rate Limiting, Authentication/Authorization, and SSL Termination**.
*   **Service Discovery:** A registry (like Consul or Eureka) that keeps track of the dynamic IP addresses of all your microservices as they scale up and down, allowing the API Gateway and other services to find them.

### 2. Caching & CDN (Content Delivery Network)
*   **Caching:** Storing frequently accessed, computationally expensive, or slow-to-read data in fast RAM (e.g., Redis, Memcached). Reduces database load and improves latency.
*   **CDN:** A globally distributed network of proxy servers. It caches **static assets** (Images, Videos, HTML, CSS, JS) geographically close to the user. (e.g., An Indian user downloads a video from a Mumbai server instead of a US server).

### 3. Message Queues (Asynchronous Processing)
*   **What they do:** Decouple components so they don't have to wait for each other synchronously. (e.g., Apache Kafka, RabbitMQ, Amazon SQS).
*   **Why use them?** To absorb traffic spikes (buffering), enable retry mechanisms, and allow background workers to process heavy tasks (like video processing or sending 100,000 emails) without making the user wait.

---

# 5. Database Selection: SQL vs. NoSQL

Choosing the right database is the most critical decision in a system design interview.

### SQL (Relational Databases)
*   **Examples:** PostgreSQL, MySQL, Oracle.
*   **Characteristics:** Strict schema, tabular data, ACID compliance (Strong consistency), relates data via Foreign Keys.
*   **Scaling:** Generally scales *vertically* well. Hard to scale horizontally (requires Sharding).
*   **Best For:** Financial transactions, highly structured data, when relationships between data entities are complex and strict correctness is required.

### NoSQL (Non-Relational Databases)
*   **Examples:** MongoDB (Document), Cassandra (Column-family), DynamoDB (Key-Value), Neo4j (Graph).
*   **Characteristics:** Flexible or schema-less, optimized for specific access patterns, often favors Eventual Consistency (AP in the CAP theorem) over Strong Consistency.
*   **Scaling:** Designed from the ground up for seamless *horizontal* scaling.
*   **Best For:** Massive read/write throughput, unstructured data, rapidly evolving schemas, leaderboards, time-series data, chat histories.

---

# 6. The Standard Core Architecture (Mental Model)

Memorize this flow. Almost every system design interview starts by drawing this exact architecture on the whiteboard before diving into specific components.

```text
       [ Clients (Mobile / Web) ]
                  |
             (DNS Lookup)
                  |
                  v
          [ CDN (Static Assets) ]
                  |
            (Dynamic Traffic)
                  |
                  v
           [ Load Balancer ]
                  |
                  v
            [ API Gateway ]  <--- Auth, Rate Limiting
                  |
       +----------+----------+
       |                     |
[ App Server 1 ]      [ App Server 2 ]  <--- Stateless Microservices
       |                     |
       +----------+----------+
                  |
    +-------------+-------------+
    |             |             |
[ Redis ]   [ PostgreSQL ]  [ Kafka ]  
 (Cache)  (Primary Database) (Message Queue)
                                |
                                v
                       [ Async Workers ] <--- Background processing
```

---

# 7. Pattern Recognition & Interview Playbook

When defining your architecture, use these default choices as your baseline, deviating only when requirements demand it:

| Component | Default Choice | Why? |
| :--- | :--- | :--- |
| **Compute / API** | Java Spring Boot + REST | Industry standard, robust, great ecosystem. |
| **Relational DB** | PostgreSQL | Feature-rich, highly reliable, supports JSONB. |
| **Distributed Cache** | Redis | Extremely fast, versatile data structures. |
| **Asynchronous Messaging** | Apache Kafka | High throughput, durable, replayable events. |
| **Static Assets / Media** | AWS S3 + CloudFront (CDN) | Infinite storage, globally distributed edge caching. |
| **Search Engine** | Elasticsearch | Unmatched text search and log aggregation. |

### The 12-Level System Design Framework
Whenever you are given a design problem (e.g., "Design Twitter"), follow this exact sequence:
1. Clarify Requirements (Functional & Non-Functional)
2. Back-of-the-envelope Capacity Estimation (QPS, Storage)
3. Define Core API Endpoints
4. Define Data Model / Schema
5. Draw High-Level Architecture
6. Dive into Detailed Components (How does the newsfeed generate?)
7. Scaling strategies (Bottlenecks)
8. Caching strategies
9. Asynchronous processing (Queues)
10. Failure handling & Redundancy
11. Monitoring & Observability
12. Trade-offs discussed explicitly

---

# 8. Level 12 Progress Tracker

| Topic | Status | Notes |
| :--- | :--- | :--- |
| **Functional vs Non-functional** | Covered | What vs How (Availability, Latency). |
| **Scalability, Availability, Reliability** | Covered | 9s of availability, fault tolerance. |
| **Latency vs Throughput** | Covered | Time per request vs Volume (QPS). |
| **Horizontal vs Vertical Scaling** | Covered | Scaling out vs Scaling up; SPOF. |
| **Load Balancing & Stateless Services** | Covered | L4/L7 routing, externalizing session state. |
| **Caching & CDN** | Covered | Edge caching static assets, Redis memory caching. |
| **Database Selection (SQL vs NoSQL)** | Covered | ACID/Relational vs Schema-less/Distributed. |
| **Message Queues** | Covered | Decoupling, async processing, buffering. |
| **API Gateway & Service Discovery** | Covered | Routing, rate limiting, auth. |
| **Core Architecture Diagram** | Covered | Client → LB → App → Cache/DB/MQ. |