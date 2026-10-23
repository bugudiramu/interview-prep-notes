# Level 14 — Redis, Kafka & Distributed Systems
## Part 1: Apache Kafka — Core Architecture & Fundamentals — Revision Notes

**Status:** In progress  
**Source:** [Apache Kafka Crash Course | KodeKloud](https://youtu.be/cNFAP9OnJjo)  
**Focus:** Event-driven architecture, brokers, topics, partitioning, replication, and interview patterns

---

# 1. Event Streaming & Kafka Fundamentals

### Core Definitions
- **Event:** A record of an occurrence or happening in a system, often represented as a **key-value pair** in Kafka.
- **Event Streaming:** The continuous flow and processing of event data in real time, enabling immediate analysis or reaction.
- **Apache Kafka:** A **distributed event streaming platform** used to build real-time data pipelines and streaming applications. It acts as a centralized **hub (or backbone) for data movement** between producers and consumers.

### High-Level Architecture

```text
Producers (Web, Microservices, IoT, Mobile Apps)
                      ↓
        +---------------------------+
        |       Apache Kafka        |
        |  (Central Data Movement   |
        |      Hub / Backbone)      |
        +---------------------------+
                      ↓
Consumers (Microservices, Analytics, Databases)
```

### Pitfalls of Traditional Event-Driven Architecture (Why Kafka?)

Without a central streaming platform like Kafka, direct service-to-service event communication runs into five major issues at scale:

| Pitfall | Problem | Real-World Impact |
|---|---|---|
| **Tight Coupling** | Services become too intertwined and dependent on each other. | Updating an order system could break a tightly coupled payment system during peak sales. |
| **Reduced Scalability** | Systems struggle to handle growing data and user demands due to interdependencies. | Traffic spikes (e.g., millions of tweets during major events) cause services to buckle. |
| **Single Point of Failure** | If one downstream service goes down, the whole chain crumbles like dominoes. | A crashed recommendation engine component halting the entire user browsing flow. |
| **No Message Persistence** | Events aren't stored over time; if a consumer is down, data vanishes. | Losing financial transactions, stock trading signals, or customer orders permanently. |
| **Limited Functionality** | Lacks built-in real-time analytics, replayability, or standardized error recovery. | Logistics systems losing package tracking updates without recovery mechanisms. |

### Advantages of Apache Kafka

| Advantage | Meaning |
|---|---|
| **High Throughput** | Handles massive event streams (thousands to millions of events/sec) effortlessly. |
| **Fault Tolerance** | Distributed architecture guarantees resilience and high availability even during node failures. |
| **Scalability** | Scales horizontally by adding more broker nodes without service disruption. |
| **Real-Time Processing** | Low-latency stream processing enables instant data insights and up-to-the-second reactions. |

---

# 2. Kafka Brokers — The Foundation of Message Storage

A **Broker** is a physical machine (or container/VM) with disks attached to it where the messages sent to Kafka are physically stored and managed.

> **Mental Model:** A Kafka broker is a single server node in the Kafka cluster responsible for receiving, storing on disk, and serving messages.

### Key Characteristics of Brokers

| Feature | Description |
|---|---|
| **Message Management** | Brokers handle storing, retrieving, and distributing messages across consumers. |
| **Cluster Node** | Each broker is an autonomous node in the Kafka cluster that communicates to prevent single-point bottlenecks. |
| **Scalability** | Brokers enable horizontal scaling by distributing topic partitions across nodes as volume grows. |
| **Fault Tolerance** | Redundancy across brokers ensures message durability during broker hardware failures. |
| **Dynamic Membership** | Brokers can join or leave clusters dynamically without downtime. |

**Default Port:** `9092`

---

# 3. Kafka Topics — Organizing Your Data Streams

A **Topic** is a logical way of organizing and categorizing Kafka messages. Think of it as a container or table that holds messages of a similar type (e.g., `transaction_topic`, `ev_charging_topic`, `station_metric_topic`).

- **Broker vs. Topic:** A broker is a *physical* concept (machine/disk), whereas a topic is a *logical* concept.

### Five Pillars of Kafka Topics

| Pillar | Description |
|---|---|
| **Message Categorization** | Topics group related messages logically (emails, logs, IoT sensor data, payment events). |
| **Immutable Log** | Messages in a topic are stored sequentially in an append-only log. Once written, a message **cannot be modified**. |
| **Multi-Consumer Access** | Multiple independent consumers (e.g., fraud detection AND reporting dashboard) can read from the same topic simultaneously. |
| **Decoupled Communication** | Producers and consumers interact strictly through topics. Producers push and move on; consumers poll at their own pace. |
| **Replication** | Topics guarantee data availability via replication across multiple cluster brokers. |

### Important Topic Insights
- **Retention Policy:** Because logs are immutable, Kafka manages disk space via retention settings configured either by **time** (e.g., `7 days` / `retention.ms`) or by **size** (e.g., `1 GB`).
- **Offsets:** Each consumer tracks its own reading position in the topic log sequentially using an **offset**.
- **Metadata Overhead:** Kafka imposes no hard limit on topics, but having tens of thousands of topics increases cluster metadata coordination.

---

# 4. Partitioning — Parallelism, Scale & Ordering

**Partitioning** is the process of breaking up a topic's log and spreading the parts across multiple brokers to avoid single-broker CPU/RAM/disk bottlenecks and achieve parallelism.

```text
Topic: charging_station_topic
 ├── Partition 01 (Stored on Broker 1)  <-- Key: Station A
 ├── Partition 02 (Stored on Broker 2)  <-- Key: Station B
 └── Partition 03 (Stored on Broker 3)  <-- Key: Station C
```

### Core Benefits of Partitioning

| Benefit | How Partitioning Achieves It |
|---|---|
| **Distribution** | Distributes a topic's data across multiple brokers in a Kafka cluster so no single broker becomes a bottleneck. |
| **Parallelism** | Enables parallel message consumption by multiple worker processes, boosting total throughput. |
| **Scalability** | Allows a single topic to scale far beyond the storage or compute capacity of any single physical server. |
| **Fault Tolerance** | If one broker fails, partitions residing on other brokers remain operational. |
| **Ordering** | **Within a single partition**, messages are strictly ordered sequentially by offset. *(Ordering is NOT guaranteed across different partitions).* |

### Partition Design Considerations (Interview Essentials)

1. **Partition Key (Message Key):**
   - When producing messages, supplying a key (e.g., `user_id` or `account_id`) guarantees that all messages with the **same key always route to the same partition**.
   - This guarantees strict ordering for that specific entity and ensures **Data Locality**.
2. **Over-Partitioning vs. Under-Partitioning:**
   - **Over-partitioning:** Hundreds of partitions per topic introduce heavy metadata, replication latency, and leader election overhead during failovers.
   - **Under-partitioning:** Too few partitions limits consumer parallelism and causes throughput bottlenecks.
   - **Rule of Thumb:** Align the partition count with your peak throughput requirements and planned consumer instances.
3. **Consumer Group Size vs. Partitions:**
   - Within a single Consumer Group, **each partition is assigned to at most one consumer**.

| Scenario | Behavior |
|---|---|
| `Consumers < Partitions` | Some consumer instances read from multiple partitions. |
| `Consumers == Partitions` | **Optimal:** 1-to-1 mapping for balanced, maximum parallelism. |
| `Consumers > Partitions` | Surplus consumers **sit idle** (active as hot standbys). |

---

# 5. Replication — High Availability & Durability

**Important Distinction:** Partitioning *splits* data across brokers for throughput; it does **not** duplicate it. If a broker fails and replication is not configured, its partitions are temporarily or permanently lost. **Replication** solves this by maintaining backup copies across separate brokers.

### Core Benefits of Replication

| Benefit | How Replication Achieves It |
|---|---|
| **High Availability** | If a partition leader fails, a follower replica on another broker takes over immediately. |
| **Fault Tolerance** | Guards against data loss resulting from hardware failure, disk corruption, or network isolation. |
| **Scalability** | Replicas can distribute read traffic in modern versions for localized consumers. |
| **Data Durability** | Multiple disk copies ensure zero data loss when configured with appropriate acknowledgments (`acks=all`). |
| **Increased Throughput** | Enables distributed operations and predictable failovers without global cluster locks. |

### Leader & Follower Architecture (Replication Factor)

Example: Topic partition with **Replication Factor = 3** across 3 Brokers:

```text
Broker 1: [ Partition 01 (LEADER) ]   <-- Writes and standard reads happen here
                 |          |
        Replicates        Replicates
                 ↓          ↓
Broker 2: [ Partition 01 (Follower) ]
Broker 3: [ Partition 01 (Follower) ]
```

- **Execution Flow:**
  1. **Standard Operation:** All producer writes route to the designated **Leader** partition on Broker 1, which syncs data to **Followers** on Brokers 2 and 3.
  2. **Follower Failure:** If Broker 3 drops, the cluster continues operating with the remaining in-sync replicas (ISR) without consumer interruption.
  3. **Leader Failure (Failover):** If Broker 1 crashes, an in-sync follower (e.g., Broker 2) is elected as the new **Leader**, and producers/consumers seamlessly reconnect.

---

# 6. Practical CLI & Docker Cheat Sheet

### Docker Local Setup
Run Kafka alongside a cluster UI on a shared Docker bridge network to communicate over port `9092`.

### Essential Kafka CLI Commands (located in `/bin`)

```bash
# 1. Create a simple topic (default: 1 partition)
./kafka-topics.sh --create \
  --topic demo-topic \
  --bootstrap-server localhost:9092

# 2. Create a topic with specific partitions and replication factor
./kafka-topics.sh --create \
  --topic payment-topic \
  --partitions 3 \
  --replication-factor 3 \
  --bootstrap-server localhost:9092

# 3. Inspect / Describe topic metadata (leader, replicas, ISR, configs)
./kafka-topics.sh --describe \
  --topic payment-topic \
  --bootstrap-server localhost:9092

# 4. Alter topic configuration dynamically (e.g., set retention time to 7 days)
./kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name payment-topic \
  --alter --add-config retention.ms=604800000

# 5. Verify broker connectivity and API versions
./kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

---

# 7. Real-World System Design Use Cases

| Use Case | Producers | Topics | Consumers / Actions |
|---|---|---|---|
| **Ride-Sharing App (Uber/Ola)** | Rider App, Driver App, Telemetry GPS sensors | `ride-requests`, `driver-locations`, `ride-completed` | Matching algorithm, real-time map tracker, wallet billing service |
| **Financial Transactions** | Payment Gateways, ATM networks, Core Banking APIs | `payment-initiated`, `account-debited` | Fraud detection engine, compliance auditor, balance ledger, push notifications |
| **EV Charging Network** | Charger IoT hardware | `charger-status`, `charging-session`, `energy-consumption` | Mobile availability map, remote diagnostics monitor, invoicing service |

---

# 8. Pattern Recognition & Interview Cheat Sheet

| If the interviewer asks… | Think… |
|---|---|
| Why not use synchronous REST calls between microservices? | REST causes tight coupling, cascading failures (domino effect), and data loss if downstream services are offline. |
| How do we guarantee events for a specific `accountId` are processed strictly in order? | Set `accountId` as the **Partition Key** so all related messages go to the same partition, preserving sequence. |
| Consumer lag is growing rapidly; how do we scale up? | Increase topic **Partitions** and scale out the number of consumer instances up to that partition count. |
| We added 10 consumers to a topic with 4 partitions, but throughput didn't increase. Why? | Only 4 consumers are active; each partition within a group can only be read by one consumer. 6 consumers sit idle. |
| What happens if a broker disk fails permanently? | With **Replication Factor > 1**, an in-sync follower replica is elected Leader, maintaining uptime and durability. |
| How can multiple independent services read the same event without consuming/destroying it? | Kafka logs are **immutable**. Each service connects with its own **Consumer Group** and maintains its own **offset pointer**. |

---

# 9. Common Kafka Mistakes

- **Confusing Partitioning with Replication:** Partitioning distributes data for scale/parallelism; replication clones data across nodes for durability. Both are required in production.
- **Assuming Global Topic-Wide Ordering:** Kafka guarantees FIFO ordering **only per partition**, never across distinct partitions.
- **Overly Aggressive Retention Settings:** Setting retention limits too low causes unread data to be purged before slow consumers catch up.
- **Mismatching Partitions and Consumers:** Expecting more consumers than partitions to increase processing speed within a single consumer group.
- **Over-Partitioning:** Creating an excessive number of partitions increases Zookeeper/KRaft metadata overhead, socket connections, and leader election failover time.

---

# 10. Level 14 — Kafka Progress Tracker

| Topic | Status | Notes |
|---|---|---|
| **Producer → Topic → Partition → Broker** | Covered | Covered in Part 1 (this document) |
| **Ordering & Partition Keys** | Covered | Strict ordering guaranteed per partition via message keys |
| **Replication & High Availability** | Covered | Leader/Follower ISR election & failover dynamics |
| **Consumer Groups & Offsets** | Up Next | Group rebalancing protocols, automatic vs manual offset commits |
| **Retry & Dead-Letter Queues (DLQ)** | Not Started | Poison pill isolation, backoff retries, Spring Kafka error handlers |
| **Delivery Guarantees & Idempotency** | Not Started | At-least-once, idempotent producers, consumer de-duplication patterns |
| **Spring Boot Kafka Integration** | Not Started | `KafkaTemplate`, `@KafkaListener`, production configs |