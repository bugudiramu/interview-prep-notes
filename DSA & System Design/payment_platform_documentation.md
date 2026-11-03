# Payment Platform (Ledger System)

A production-ready, highly concurrent payment ledger and transfer system built with Spring Boot 4. This system is designed to handle financial transactions safely, preventing double-spends and race conditions while maintaining high throughput using an asynchronous event-driven architecture.

## 🚀 Core Technology Stack

* **Language & Framework:** Java 21, Spring Boot 4.1.1 (Web, Data JPA, Security, Kafka, Actuator)
* **Primary Database:** PostgreSQL 16 (Relational Ledger, ACID compliance)
* **In-Memory Datastore:** Redis 7 (Caching, Idempotency, Rate Limiting)
* **Message Broker:** Apache Kafka KRaft (Asynchronous pipelines, Dead-Letter Queues)
* **Migrations:** Flyway (Version-controlled database schema)
* **Testing:** JUnit 5, Testcontainers 2.0 (Real Docker containers for integration testing)

---

## 🏗️ Architecture & Features Built

### 1. Security & Error Handling
* **Stateless JWT Authentication:** BCrypt password hashing and custom JWT token generation/validation using Spring Security filters.
* **RFC 7807 Problem Details:** Standardized JSON error responses via a `@RestControllerAdvice` Global Exception Handler (e.g., 400 Bad Request, 429 Too Many Requests, 403 Forbidden).

### 2. Core Transfer Logic & Concurrency Control
* **ACID Transactions:** Strict `@Transactional` boundaries for all balance deductions and additions.
* **Pessimistic Row-Level Locking:** Uses PostgreSQL `SELECT FOR UPDATE` (`@Lock(PESSIMISTIC_WRITE)`) to lock sender and receiver rows during a transfer, mathematically eliminating double-spend race conditions.
* **Deadlock Prevention:** Lexicographical sorting of account UUIDs before locking ensures threads always acquire locks in the exact same order, preventing circular database deadlocks.

### 3. Idempotency & Rate Limiting
* **Distributed Idempotency:** Prevents duplicate network charges by utilizing Redis `SETNX` (Set if Not Exists) atomic operations on the transfer `referenceId`.
* **Fixed-Window Rate Limiting:** Protects the `/transfers` endpoint from abuse by limiting users to 5 requests per minute using Redis atomic `INCR` and `EXPIRE` commands.
* **Cache-Aside Pattern:** Fetches account details from Redis when available, and uses `@CacheEvict` to invalidate stale data the moment a database transaction commits.

### 4. Async Event-Driven Pipeline (Kafka)
* **Decoupled Notifications:** Once a transfer commits, a `TransactionEvent` is published to Kafka, allowing the main HTTP thread to return `201 Created` instantly.
* **Exponential Backoff & Retries:** Background consumers utilize `@RetryableTopic` to automatically retry failed third-party notifications with delays of 1s, then 2s.
* **Dead-Letter Queue (DLQ):** Poison-pill messages that exhaust all retries are safely routed to a `-dlt` topic and caught by a `@DltHandler`.
* **Persistent Audit Logging:** Both successful events and DLQ failures are permanently recorded in an `audit_logs` PostgreSQL table.

### 5. Observability & Query Optimization
* **Pagination & Filtering:** History endpoint uses `Pageable` to return chunked transaction data.
* **Composite B-Tree Indexes:** Database queries avoid expensive in-memory sorts by leveraging compound indexes (`source_account_id, created_at DESC`).
* **Spring Boot Actuator:** Exposes `/actuator/health` and `/actuator/metrics` for Kubernetes liveness/readiness probes and Prometheus scraping.

### 6. Integration Testing
* **Testcontainers:** Bypasses database mocking by spinning up real, disposable PostgreSQL Docker containers during `./gradlew test`.
* **High-Concurrency Testing:** Verifies pessimistic locking by firing 10 simultaneous threads at the exact same microsecond using Java `CountDownLatch`.

---

## 🗺️ How to Read the Codebase

Start reading the codebase from the edges inward. 

1. **`src/main/resources/db/migration/`**: Always look here first. The `V1` to `V4` SQL files show the exact database schema, constraints, and indexes. The database is the source of truth.
2. **`domain/entity/` & `repository/`**: The JPA data layer. Look at `Account.java` to see the `@Version` field (Optimistic Locking) and `AccountRepository.java` to see the `@Lock(PESSIMISTIC_WRITE)` annotation preventing race conditions.
3. **`dto/` & `controller/`**: The API boundary. The `TransferRequest.java` record shows the strict validation rules. Controllers handle HTTP routing and rate limiting.
4. **`exception/`**: `GlobalExceptionHandler.java` catches internal errors and formats them into standard RFC 7807 JSON responses.
5. **`security/`**: Contains `JwtAuthenticationFilter.java` (intercepts requests to validate Bearer tokens) and `JwtService.java` (handles token generation).
6. **`service/`**: The business logic orchestration. 
7. **`event/` & `worker/`**: The asynchronous Kafka layer. `EventPublisher.java` sends messages, and `NotificationWorker.java` consumes them in the background.

---

## 🔄 The Lifecycle of a Transfer Request

If you want to trace the execution of the system's most critical endpoint (`POST /api/v1/transfers`), follow this sequential flow:

1. **Authentication (`JwtAuthenticationFilter`):** The request arrives. The filter intercepts it, extracts the JWT from the `Authorization` header, validates the signature, and loads the user into the Spring Security Context.
2. **Rate Limiting (`TransferController`):** The controller delegates to `RateLimiterService`. Redis increments a counter for the user. If the user exceeds 5 requests per minute, it throws a `429 Too Many Requests`.
3. **Idempotency (`TransferService`):** Before touching PostgreSQL, `IdempotencyService` attempts an atomic `SETNX` in Redis using the `referenceId`. If the key already exists, the system halts and returns a `400 Duplicate`.
4. **Database Execution (`TransferDatabaseService`):** The request enters a strict `@Transactional` boundary. The service sorts the account UUIDs to prevent deadlocks, acquires a `SELECT FOR UPDATE` pessimistic lock on both rows in PostgreSQL, verifies sufficient funds, deducts/adds balances, and saves a `Transaction` ledger entry.
5. **Cache Invalidation (`TransferService`):** Once the database transaction successfully commits, the service explicitly evicts the stale balances for the source and destination accounts from the Redis cache.
6. **Async Event Publishing (`EventPublisher`):** A `TransactionEvent` is serialized to JSON and pushed to the `transaction-events` Kafka topic. The HTTP request immediately returns `201 Created` to the client.
7. **Background Consumption (`NotificationWorker`):** A separate thread picks up the Kafka event. If the notification fails (e.g., simulated gateway timeout), Spring Kafka automatically routes it to `retry-0` (1s delay), then `retry-1` (2s delay), and finally to a Dead-Letter Queue (`transaction-events-dlt`), writing the failure to the `audit_logs` PostgreSQL table.

---

## 🛠️ How to Run

1. **Start Infrastructure (PostgreSQL, Redis, Kafka):**
   ```bash
   docker compose up -d
   ```
2. **Run the Application:**
   ```bash
   ./gradlew bootRun
   ```
3. **Run the Test Suite (Requires Docker for Testcontainers):**
   ```bash
   ./gradlew test
   ```