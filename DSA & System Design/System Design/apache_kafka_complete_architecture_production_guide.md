# Level 14 — Redis, Kafka & Distributed Systems
## Apache Kafka — Complete Architecture, Reliability & Spring Boot — Revision Notes

**Status:** Completed  
**Language / Stack:** Java + Spring Boot (`Spring for Apache Kafka`)  
**Focus:** Brokers, topics, partitioning, replication, consumer groups, offsets, retries, DLQ, delivery guarantees, idempotency, and production Spring Boot configs

---

# 1. Event Streaming & Kafka Fundamentals

### Core Definitions
- **Event:** A record of an occurrence or state change in a system, represented in Kafka as a message containing a **Key, Value, Timestamp, and optional Headers**.
- **Event Streaming:** The continuous flow and processing of event data in real time, enabling immediate analysis or reaction.
- **Apache Kafka:** A **distributed event streaming platform** used to build real-time data pipelines and event-driven microservices. It acts as a centralized **hub (or backbone) for data movement** between producers and consumers.

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

Without a central streaming platform like Kafka, direct service-to-service communication runs into five major issues at scale:

| Pitfall | Problem | Real-World Impact |
|---|---|---|
| **Tight Coupling** | Services become intertwined and directly dependent on each other's APIs and availability. | Updating an order system could break a tightly coupled payment or notification service during peak sales. |
| **Reduced Scalability** | Systems struggle to handle sudden traffic spikes due to synchronous bottlenecks. | Flash-sale traffic overwhelms downstream databases instead of being buffered smoothly. |
| **Single Point of Failure** | If one downstream service goes down, the whole chain crumbles like dominoes. | A crashed analytics or email service halting the core user checkout flow. |
| **No Message Persistence** | Events aren't stored over time; if a receiver is offline, data vanishes. | Losing financial transactions, audit trails, or order events permanently. |
| **Limited Functionality** | Lacks built-in replayability, stream fan-out, or standardized error recovery. | Inability to replay past events to rebuild a corrupted read database or train a new ML service. |

### Advantages of Apache Kafka

| Advantage | Meaning |
|---|---|
| **High Throughput** | Handles massive event streams (hundreds of thousands to millions of events/sec) via sequential disk I/O and zero-copy networking. |
| **Fault Tolerance** | Distributed architecture guarantees resilience and high availability even during node failures. |
| **Scalability** | Scales horizontally by adding more broker nodes and partitions without service disruption. |
| **Real-Time Processing** | Low-latency stream delivery enables near-instantaneous reactions across microservices. |

---

# 2. Kafka Brokers — The Foundation of Message Storage

A **Broker** is a physical server (or container/VM) with disks attached to it where the messages sent to Kafka are physically stored and managed.

> **Mental Model:** A Kafka broker is a single stateless-compute / stateful-disk server node in the Kafka cluster responsible for receiving, appending to disk logs, replicating, and serving messages.

### Key Characteristics of Brokers

| Feature | Description |
|---|---|
| **Message Management** | Brokers handle storing, retrieving, and distributing messages across consumers. |
| **Cluster Node** | Each broker is an autonomous node in the Kafka cluster coordinated via **KRaft** (Kafka Raft metadata mode, replacing legacy ZooKeeper). |
| **Scalability** | Brokers enable horizontal scaling by distributing topic partitions across nodes as volume grows. |
| **Fault Tolerance** | Redundancy across brokers ensures message durability during broker hardware failures. |
| **Dynamic Membership** | Brokers can join or leave clusters dynamically without cluster downtime. |

**Default Port:** `9092`

---

# 3. Kafka Topics — Organizing Your Data Streams

A **Topic** is a logical way of organizing and categorizing Kafka messages. Think of it as an append-only ledger or table that holds events of a similar domain type (e.g., `payment-initiated`, `order-created`, `charger-telemetry`).

- **Broker vs. Topic:** A broker is a *physical* concept (machine/disk), whereas a topic is a *logical* concept spread across brokers.

### Five Pillars of Kafka Topics

| Pillar | Description |
|---|---|
| **Message Categorization** | Topics group related messages logically (emails, logs, IoT sensor data, payment events). |
| **Immutable Log** | Messages in a topic are stored sequentially in an append-only log in $\mathcal{O}(1)$ time. Once written, a message **cannot be modified**. |
| **Multi-Consumer Access** | Multiple independent systems (e.g., Fraud Detection AND Analytics) can read from the same topic simultaneously. |
| **Decoupled Communication** | Producers and consumers interact strictly through topics. Producers push and move on; consumers poll at their own pace (natural **backpressure** buffer). |
| **Replication** | Topics guarantee data availability via partition replication across multiple cluster brokers. |

### Important Topic Insights
- **Retention Policy:** Because logs are immutable and reading a message **does not delete it**, Kafka reclaims disk space via retention policies configured by **time** (default `7 days` / `retention.ms`) or **size** (`retention.bytes`), or via **Log Compaction** (keeping only the latest value per key).
- **Metadata Overhead:** Kafka imposes no hard limit on topics, but tens of thousands of topics/partitions increase cluster metadata overhead.

---

# 4. Partitioning — Parallelism, Scale & Ordering

**Partitioning** breaks a topic's log into smaller independent append-only logs spread across multiple brokers to avoid single-broker CPU/RAM/disk bottlenecks and achieve consumer parallelism.

```text
Topic: payment-events
 ├── Partition 0 (Stored on Broker 1)  <-- Key: account_101 (Offsets: 0, 1, 2...)
 ├── Partition 1 (Stored on Broker 2)  <-- Key: account_205 (Offsets: 0, 1, 2...)
 └── Partition 2 (Stored on Broker 3)  <-- Key: account_309 (Offsets: 0, 1, 2...)
```

### Core Benefits of Partitioning

| Benefit | How Partitioning Achieves It |
|---|---|
| **Distribution** | Distributes a topic's storage and I/O across multiple brokers so no single machine becomes a bottleneck. |
| **Parallelism** | Enables parallel message consumption by multiple worker instances, boosting total throughput. |
| **Scalability** | Allows a single topic to scale far beyond the storage or network capacity of any single physical server. |
| **Fault Tolerance** | If one broker fails, partitions led by other brokers remain unaffected. |
| **Ordering** | **Within a single partition**, messages are strictly ordered sequentially by offset. *(Ordering is NOT guaranteed across different partitions).* |

### Partition Routing Mechanics
1. **With a Message Key (Standard in Backend Systems):**
   - Kafka hashes the key: $\text{Partition} = \text{murmur2}(\text{key}) \bmod \text{numPartitions}$.
   - Guarantees all events for the **same `accountId` or `orderId` always route to the exact same partition**, preserving strict chronological order for that entity.
   - **Warning:** Adding new partitions later changes $\text{numPartitions}$, which changes the modulo hash mapping for existing keys! Size partitions thoughtfully upfront.
2. **Without a Message Key (`key = null`):**
   - Uses the **Sticky Partitioner** (fills a micro-batch for one partition before switching to the next) to maximize batching throughput, sacrificing ordering.

---

# 5. Replication — High Availability & Durability

**Important Distinction:** Partitioning *splits* data across brokers for throughput; it does **not** copy it. **Replication** maintains backup copies of each partition across separate brokers so hardware failures never cause data loss.

### Leader, Followers & ISR (In-Sync Replicas)

Example: Topic partition with **Replication Factor = 3** across 3 Brokers:

```text
Broker 1: [ Partition 0 (LEADER) ]   <-- Producers write here; Consumers typically read here
                 |          |
        Replicates        Replicates
                 ↓          ↓
Broker 2: [ Partition 0 (Follower - ISR) ]
Broker 3: [ Partition 0 (Follower - ISR) ]
```

- **Leader:** Handles all writes (and normally reads) for that specific partition.
- **Followers:** Passively fetch messages from the Leader in exact offset order and write them to their own local disk logs.
- **ISR (In-Sync Replicas):** The subset of replicas (Leader + caught-up Followers) that are fully up-to-date within `replica.lag.time.max.ms`. Only an **ISR member** is eligible to be elected Leader during a clean failover, preventing data loss.

### Producer Acknowledgment Settings (`acks`)

| Setting | How It Works | Durability vs. Latency Trade-off |
|---|---|---|
| `acks=0` | Producer fires and forgets without waiting for broker response. | Lowest latency, **highest data loss risk** (good only for lossy clickstream/metrics). |
| `acks=1` | Leader writes to its local disk and responds immediately *before* followers replicate. | Balanced, **can lose data** if Leader crashes right after acknowledging but before replicating. |
| `acks=all` (`-1`) | Leader waits until **all In-Sync Replicas (ISR)** write the message before acknowledging. | **Zero data loss** when paired with `replication.factor=3` and `min.insync.replicas=2` (**Production Standard**). |

> **Why `RF=3` + `min.insync.replicas=2` + `acks=all`?**  
> It guarantees every acknowledged write exists on **at least 2 separate disks**, while still allowing **1 broker to go down** for maintenance or failure without halting producer writes!

---

# 6. Consumer Groups, Offsets & Rebalancing

## 1. Consumer Groups — Combining Queue & Pub/Sub Semantics

A **Consumer Group** (`group.id`) is a logical cluster of consumer instances cooperating to process a topic.

```text
Topic: order-created (3 Partitions: P0, P1, P2)
 │
 ├──> Consumer Group A: "payment-service" (3 instances -> 1 partition each: Load Balanced)
 │       ├── Payment-Worker-1 <-- P0
 │       ├── Payment-Worker-2 <-- P1
 │       └── Payment-Worker-3 <-- P2
 │
 └──> Consumer Group B: "analytics-service" (1 instance -> reads ALL 3 partitions: Fan-out)
         └── Analytics-Worker-1 <-- P0, P1, P2
```

- **Within the SAME Consumer Group (Queue / Load Balancing):** Each partition is assigned to **at most one consumer instance** in the group.
- **Across DIFFERENT Consumer Groups (Pub/Sub Broadcast):** Every consumer group independently receives a full copy of all messages in the topic.

| Partition-to-Consumer Ratio (Single Group) | Behavior |
|---|---|
| `Consumers < Partitions` | Some consumer instances are assigned multiple partitions. |
| `Consumers == Partitions` | **Optimal:** 1-to-1 mapping for maximum balanced parallelism. |
| `Consumers > Partitions` | Surplus consumers **sit completely idle** (act as warm standbys for failover). |

---

## 2. Offsets & Commit Strategies

An **Offset** is a monotonically increasing 64-bit integer ($0, 1, 2, \dots$) uniquely identifying a record's position within a specific partition.
- Consumers commit their processed offsets to an internal Kafka topic named **`__consumer_offsets`**.
- **`auto.offset.reset`:** Controls what happens when a *brand-new* consumer group connects (no committed offset exists yet):
  - `earliest`: Start reading from the very beginning of the retained partition log.
  - `latest` *(default)*: Only read new messages arriving after the consumer joins.

### Automatic vs. Manual Offset Commits (Critical Interview Topic)

| Strategy | Configuration | Risks & Behavior |
|---|---|---|
| **Auto-Commit** | `enable.auto.commit=true` (every 5s by default) | **Risk of Data Loss!** Consumer polls 100 records at $t=0\text{s}$; background timer auto-commits offset 100 at $t=5\text{s}$ on the next poll; JVM crashes at $t=6\text{s}$ while only 40 records were actually saved to DB. Records 41–100 are **skipped forever**. |
| **Manual Sync/Batch Commit** *(Production Standard)* | `enable.auto.commit=false` + Spring `AckMode.RECORD` or `MANUAL_IMMEDIATE` | Offset is committed **only after** business logic and database writes succeed. If JVM crashes mid-processing, uncommitted records are re-read upon restart (**At-Least-Once**). |

---

## 3. Consumer Group Rebalancing

**Rebalancing** is the protocol coordinated by the **Group Coordinator** broker to redistribute partition ownership across consumers in a group.

### What Triggers a Rebalance?
1. A new consumer instance joins the group (scaling up pods).
2. An existing consumer shuts down cleanly or crashes (`session.timeout.ms` expires without heartbeats).
3. **Slow Processing Trap (`max.poll.interval.ms`):** A consumer takes too long processing a batch (e.g., slow external HTTP call $> 5\text{ mins}$) and fails to call `poll()` in time. Kafka assumes the consumer is dead/hung, evicts it, and triggers a rebalance!
4. Topic partition count is increased.

### Rebalance Protocols
- **Eager Rebalancing (Legacy Default):** *"Stop-the-World."* **All** consumers revoke **all** their partitions, pause processing completely, and wait for partitions to be reassigned. Causes latency spikes.
- **Cooperative Sticky Rebalancing (`CooperativeStickyAssignor` — Modern Best Practice):** Incremental rebalancing. Consumers keep processing unaffected partitions and **only revoke the specific partitions** that need to move to a newly joined consumer.

---

# 7. Delivery Guarantees & Idempotency

## 1. The Three Delivery Semantics

| Guarantee | Meaning | How It Happens | Use Case |
|---|---|---|---|
| **At-Most-Once** | Messages are delivered $0$ or $1$ time. **May lose data, never duplicates.** | Commit offset *before* processing message (or `acks=0`). | IoT temperature readings, non-critical log metrics. |
| **At-Least-Once** *(99% of Production Systems)* | Messages are delivered $1$ or more times. **Zero data loss, may produce duplicates.** | Producer `acks=all` + retries; Consumer commits offset *after* DB write. | Payments, orders, notifications (**paired with Idempotent Consumer!**). |
| **Exactly-Once (EOS)** | Each message affects downstream state **strictly once**, even during crashes. | Kafka Transactions (`transactional.id` + `isolation.level=read_committed`) for **Kafka-to-Kafka** pipelines. | Kafka Streams financial aggregations. |

> **Senior Backend Insight:** Kafka's native "Exactly-Once" only spans *Kafka-to-Kafka* workflows. When your Spring Boot consumer reads from Kafka and writes to **PostgreSQL** or calls **Stripe**, native Kafka EOS cannot roll back an external HTTP call or separate DB without distributed transactions. Therefore, **production backend systems rely on At-Least-Once delivery + Idempotent Producers + Idempotent Consumers** (which achieves *"Effectively-Once"* processing!).

---

## 2. Idempotent Producer (`enable.idempotence=true`)

**The Duplicate Write Problem:**
```text
1. Producer sends PaymentEvent to Broker Leader.
2. Broker writes PaymentEvent to disk (Offset 42) and replicates to ISR.
3. Network glitch drops the Broker's ACK response back to Producer!
4. Producer times out, assumes failure, and RETRIES sending PaymentEvent.
5. Without idempotence -> Broker writes a DUPLICATE record at Offset 43!
```

**How Kafka Solves It (`enable.idempotence=true`, enabled by default in Kafka 3.0+):**
- Each producer is assigned a unique **Producer ID (`PID`)** and attaches a monotonically increasing **Sequence Number** per partition to every batch.
- If the broker receives `PID=99, Seq=5` a second time, it recognizes it already persisted `Seq=5`, **deduplicates it in memory without writing a second record**, and simply returns the success `ACK`!

---

## 3. Idempotent Consumer (De-duplication Patterns)

**The Duplicate Read Problem:**
```text
1. Consumer polls PaymentEvent (Offset 42).
2. Consumer deducts $100 in PostgreSQL and commits DB transaction.
3. Consumer JVM crashes (OOM / Pod eviction) BEFORE committing Offset 42 to Kafka!
4. Another Consumer takes over Partition -> reads Offset 42 AGAIN!
```

Without consumer idempotency, the user gets charged **$\$200$ instead of $\$100$**!

### Production Pattern: Database Idempotency Key inside `@Transactional`

Include a deterministic unique `eventId` (UUID) in every Kafka payload and record processed IDs in the **exact same ACID database transaction** as your business state change:

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

```java
@Transactional
public void processTransferEvent(TransferEvent event) {
    // 1. Atomic check-and-insert (or INSERT ... ON CONFLICT DO NOTHING)
    if (processedEventRepository.existsById(event.eventId())) {
        log.info("Duplicate event {} detected. Skipping safely.", event.eventId());
        return;
    }

    // 2. Execute business mutation
    accountRepository.debit(event.fromAccountId(), event.amount());
    accountRepository.credit(event.toAccountId(), event.amount());

    // 3. Save idempotency record in the SAME PostgreSQL transaction!
    processedEventRepository.save(new ProcessedEvent(event.eventId()));
}
```
- Because Step 2 and Step 3 share one PostgreSQL transaction, they either **both commit** or **both roll back**. Even if Kafka redelivers the message 10 times, the balance only changes **once**.

---

## 4. Bonus: The Dual-Write Problem & Transactional Outbox Pattern

**Problem:** In an Order API, you save an `Order` to PostgreSQL and then call `kafkaTemplate.send("order-created", event)`. What if PostgreSQL commits, and the server crashes $1\text{ ms}$ later before `kafkaTemplate.send()` completes? Your DB says the order exists, but downstream services never hear about it!

**Solution — Transactional Outbox Pattern:**
1. Inside the single `@Transactional` DB call, insert into **both** the `orders` table and an `outbox_events` table in PostgreSQL.
2. A background relay worker (or **Debezium CDC** reading PostgreSQL WAL logs) asynchronously publishes rows from `outbox_events` to Kafka and marks them published.

---

# 8. Retry Strategies & Dead-Letter Queues (DLQ)

## 1. The "Poison Pill" Problem

A **Poison Pill** is a malformed message (corrupted JSON, unexpected `null` field, violated business invariant) that can **never** succeed no matter how many times you retry it.
- If a consumer throws an unhandled exception in an infinite loop without advancing the offset, **that entire partition is completely blocked** for all subsequent valid messages!

### Transient vs. Non-Transient Errors

| Error Category | Examples | Correct Action |
|---|---|---|
| **Transient (Recoverable)** | DB connection pool spike, HTTP `503 Service Unavailable`, optimistic lock conflict | **Retry with Exponential Backoff** ($1\text{s} \rightarrow 2\text{s} \rightarrow 4\text{s}$). |
| **Non-Transient (Fatal / Poison Pill)** | `DeserializationException`, `JsonParseException`, invalid `accountId` | **Skip retries immediately** and route straight to the **Dead-Letter Queue (DLQ)**. |

---

## 2. Blocking Retries vs. Non-Blocking Retries (Retry Topics)

### Approach A: Blocking In-Place Retries (`DefaultErrorHandler`)
- Pauses the partition consumer in memory, retries $N$ times with backoff, and if all fail, `DeadLetterPublishingRecoverer` publishes the failed message to `<topic>.DLT` and commits the offset.
- **Pros:** Preserves **strict partition ordering** (subsequent events for that key wait).
- **Cons:** Slow backoff delays block healthy events sitting behind the failing message in the same partition.

### Approach B: Non-Blocking Retries (`@RetryableTopic` in Spring Kafka)

```text
Main Topic ("payment-events")
   │ (Fails -> Immediately commits offset & forwards to retry topic!)
   ↓
"payment-events-retry-0" (Delay: 2s)
   │ (Still fails)
   ↓
"payment-events-retry-1" (Delay: 4s)
   │ (Max attempts exhausted)
   ↓
Dead-Letter Topic: "payment-events-dlt"  --> PagerDuty Alert + Manual Inspection / Replay
```

- **Pros:** Main partition **never blocks**! Healthy traffic keeps flowing at full speed while temporary failures wait in dedicated backoff topics.
- **Trade-off:** **Breaks strict per-key ordering** for the failing entity while its failed message detours through retry topics! *(Do not use non-blocking retries if Event #2 strictly depends on Event #1 succeeding first).*

### What Should Go Into a DLQ Message?
Spring Kafka automatically attaches diagnostic **Kafka Headers** to DLQ records:
- Original topic, partition, offset, and timestamp.
- Exception class name, error message, and stack trace.

---

# 9. Spring Boot Kafka Integration (Production Patterns)

## 1. Production `application.yml` Blueprint

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                        # Wait for all in-sync replicas
      retries: 2147483647              # Let delivery.timeout.ms govern retries
      properties:
        enable.idempotence: true       # Prevent duplicate writes on retry
        max.in.flight.requests.per.connection: 5
        delivery.timeout.ms: 120000    # 2 min upper bound for send + retries
        linger.ms: 5                   # Wait up to 5ms to batch records for throughput

    consumer:
      group-id: payment-processor-group
      auto-offset-reset: earliest
      enable-auto-commit: false        # CRITICAL: Never auto-commit on timer in prod!
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        # Wraps JsonDeserializer so Poison Pill JSON goes to ErrorHandler/DLQ instead of infinite loop!
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.app.events"
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        max.poll.records: 100          # Bound batch size so processing finishes well before poll timeout
        max.poll.interval.ms: 300000   # 5 mins max between poll() calls

    listener:
      ack-mode: record                 # Commit offset right after each record listener succeeds
      concurrency: 3                   # Spawns 3 consumer threads (match partition count if 1 pod)
```

---

## 2. Producer Implementation (`KafkaTemplate`)

Always send messages **with a domain key** (for ordering) and handle the asynchronous `CompletableFuture` result:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentEventProducer {

    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    private static final String TOPIC = "payment-events";

    public void publishPayment(PaymentEvent event) {
        // Key = accountId guarantees strict ordering per account!
        CompletableFuture<SendResult<String, PaymentEvent>> future =
                kafkaTemplate.send(TOPIC, event.accountId(), event);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to publish event {} for account {}",
                        event.eventId(), event.accountId(), ex);
                // Trigger fallback / alert / keep in outbox for retry
            } else {
                RecordMetadata meta = result.getRecordMetadata();
                log.info("Published event {} to topic={} partition={} offset={}",
                        event.eventId(), meta.topic(), meta.partition(), meta.offset());
            }
        });
    }
}
```

---

## 3. Resilient Consumer with Non-Blocking Retries, Idempotency & DLQ

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentEventConsumer {

    private final PaymentProcessorService paymentService;

    @RetryableTopic(
            attempts = "4", // 1 initial attempt + 3 retries
            backoff = @Backoff(delay = 2000, multiplier = 2.0, maxDelay = 10000),
            exclude = {IllegalArgumentException.class, DeserializationException.class}, // Non-transient -> straight to DLT!
            dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    @KafkaListener(topics = "payment-events", groupId = "payment-processor-group")
    public void consume(
            PaymentEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset
    ) {
        log.info("Processing eventId={} from partition={} offset={}",
                event.eventId(), partition, offset);

        // Delegates to @Transactional idempotent service method
        paymentService.processIdempotently(event);
    }

    @DltHandler
    public void handleDeadLetter(
            PaymentEvent failedEvent,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String dltTopic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage
    ) {
        log.error("ALERT: Event {} landed in DLT {} due to: {}",
                failedEvent.eventId(), dltTopic, errorMessage);
        // Persist to failed_events DB table for ops dashboard / manual replay
    }
}
```

> **When Strict Ordering Is Mandatory (e.g., Account Ledger):**  
> Instead of `@RetryableTopic`, configure a `@Bean DefaultErrorHandler` with `DeadLetterPublishingRecoverer` and `ExponentialBackOff`. That performs **in-partition blocking retries** so Event #2 for `account_101` never overtakes a retrying Event #1!

---

# 10. Practical CLI & Consumer Lag Cheat Sheet

```bash
# 1. Create a production-style topic (3 partitions, replication factor 3)
./kafka-topics.sh --create \
  --topic payment-events \
  --partitions 3 \
  --replication-factor 3 \
  --bootstrap-server localhost:9092

# 2. Describe topic metadata (Leader, Replicas, ISR)
./kafka-topics.sh --describe \
  --topic payment-events \
  --bootstrap-server localhost:9092

# 3. Monitor Consumer Group Lag (CRITICAL Production Debugging Command!)
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe \
  --group payment-processor-group

# 4. Reset Consumer Group Offsets (e.g., replay last 2 hours after fixing a bug; group must be inactive)
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-processor-group \
  --topic payment-events \
  --reset-offsets --by-duration PT2H --execute
```

- **Consumer Lag** $= \text{Log-End-Offset (Latest Produced)} - \text{Current-Offset (Last Committed)}$. Monitoring consumer lag is the **#1 golden metric** for Kafka health in production!

---

# 11. Real-World System Design Use Cases

| Use Case | Producers | Topics | Consumers / Actions |
|---|---|---|---|
| **Ride-Sharing App (Uber/Ola)** | Rider App, Driver App, Telemetry GPS sensors | `ride-requests`, `driver-locations`, `ride-completed` | Matching engine, live map tracker, wallet billing & receipt service |
| **Financial Transactions** | Payment Gateways, Core Banking APIs (via Outbox) | `payment-initiated`, `account-debited`, `payment-dlt` | Fraud engine, compliance audit log, ledger updater, push notifications |
| **EV Charging Network** | Charger IoT hardware | `charger-status`, `charging-session`, `energy-metrics` | Mobile availability map, remote diagnostics monitor, invoicing service |

---

# 12. Pattern Recognition & Interview Cheat Sheet

| If the interviewer asks… | Think… |
|---|---|
| Why not use synchronous REST calls between microservices? | Tight coupling, cascading failures, and lost events when downstream services are down or slow. |
| How do we guarantee events for a specific `accountId` are processed strictly in order? | Use `accountId` as the **Partition Key** AND use **in-partition blocking retries** (`DefaultErrorHandler`), avoiding async/non-blocking retry topics for stateful mutations. |
| Consumer lag is growing rapidly; how do we scale up? | Increase topic **Partitions** and scale out consumer pods/threads up to that partition count; optimize slow DB/HTTP calls inside the listener. |
| We scaled from 4 consumer pods to 10 on a 4-partition topic, why didn't speed improve? | Within one consumer group, 1 partition maps to at most 1 consumer. 6 pods sit idle. |
| How do we ensure zero data loss if a broker crashes immediately after a write? | Producer `acks=all` + Topic `replication.factor=3` + `min.insync.replicas=2` + Consumer manual offset commit. |
| How do we prevent charging a customer twice if a consumer crashes before committing its offset? | **At-Least-Once delivery + Idempotent Consumer** storing `eventId` in a `processed_events` PostgreSQL table inside the **same `@Transactional` block**. |
| What happens if a producer sends corrupted JSON ("Poison Pill")? | Wrap `JsonDeserializer` inside **`ErrorHandlingDeserializer`** and route `DeserializationException` directly to a **Dead-Letter Topic (`.DLT`)** with zero retries. |
| Why do our consumers keep entering constant rebalance storms under heavy load? | Processing a batch takes longer than `max.poll.interval.ms`! Lower `max.poll.records`, speed up downstream calls, and enable `CooperativeStickyAssignor`. |
| How do we update PostgreSQL and publish to Kafka atomically without losing the event if the app crashes in between? | **Transactional Outbox Pattern** (write business entity + `outbox_events` row in one DB transaction, relay asynchronously to Kafka). |

---

# 13. Common Kafka Mistakes

- **Leaving `enable.auto.commit=true` for Critical Workflows:** Auto-committing on a background timer acknowledges messages before your DB write finishes, causing silent data loss on crashes.
- **Unwrapped `JsonDeserializer` in Spring Boot:** Without `ErrorHandlingDeserializer`, a single bad JSON payload fails *before* reaching your `@KafkaListener`, triggering an infinite consumer poll-crash loop that halts the partition!
- **Blocking the Consumer Thread Beyond `max.poll.interval.ms`:** Making slow synchronous external API calls across 500 polled records causes the broker to kick the consumer out of the group, triggering endless rebalance storms and duplicate processing.
- **Confusing Idempotent Producer with End-to-End Exactly-Once:** `enable.idempotence=true` only prevents duplicate writes from producer-to-broker network retries—your **consumer must still be idempotent**!
- **Dual-Writing to DB and Kafka Directly in Request Threads:** Causes state drift between PostgreSQL and Kafka whenever one succeeds and the other fails. Use the **Outbox Pattern** for financial/order systems.
- **Using `@RetryableTopic` When Strict Ordering Is Required:** Non-blocking retry topics move failed messages out-of-line while newer messages for the same key get processed immediately.

---

# 14. Revision Checklist

## Core Architecture
- [x] Explain Brokers, Topics, Partitions, and Immutable Append-Only Logs.
- [x] Explain Partition Keys ($\text{hash}(\text{key}) \bmod N$) and per-partition ordering guarantees.
- [x] Explain Leader/Follower replication, ISR, `acks=all`, and `min.insync.replicas=2`.

## Consumers, Reliability & Spring Boot
- [x] Explain how Consumer Groups achieve both load balancing (intra-group) and broadcast fan-out (inter-group).
- [x] Contrast Automatic vs. Manual Offset Commits and explain Rebalancing (`CooperativeStickyAssignor`, `max.poll.interval.ms`).
- [x] Contrast At-Most-Once, At-Least-Once, and Exactly-Once semantics.
- [x] Implement an **Idempotent Consumer** in Spring Boot + PostgreSQL and explain the **Transactional Outbox Pattern**.
- [x] Handle **Poison Pills** using `ErrorHandlingDeserializer`, backoff retries, and **Dead-Letter Queues (DLQ)**.

---

# 15. Level 14 — Kafka Progress Tracker

| Topic | Status | Notes |
|---|---|---|
| **Producer → Topic → Partition → Broker** | Covered | Event hub, brokers, immutable log topics, partition routing |
| **Ordering & Partition Keys** | Covered | Strict ordering per partition via message keys; partition resizing caveat |
| **Replication & High Availability** | Covered | Leader/Follower ISR election, `acks=all` + `min.insync.replicas=2` |
| **Consumer Groups & Offsets** | Covered | `__consumer_offsets`, manual vs auto commit, `CooperativeStickyAssignor` rebalancing |
| **Retry & Dead-Letter Queues (DLQ)** | Covered | Poison pills, `ErrorHandlingDeserializer`, blocking vs `@RetryableTopic` retries, DLT |
| **Delivery Guarantees & Idempotency** | Covered | At-least-once, idempotent producer (`PID`+Seq), DB idempotent consumer, Outbox pattern |
| **Spring Boot Kafka Integration** | Covered | Production `application.yml`, `KafkaTemplate`, `@KafkaListener`, `@DltHandler` |