# ADR 004: Asynchronous Job Execution Architecture (PostgreSQL SKIP LOCKED vs Celery / Redis)

| Metadata | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-11-17 |
| **Deciders** | Lead Software Architect & AI Infrastructure Engineer |

---

## 1. Context & Problem Statement

Evaluation matrices can involve dozens to hundreds of combinatorial test cases across multiple LLM endpoints. Executing these synchronously within an HTTP request thread would result in gateway timeouts (HTTP 504) and client connection drops.

We need an asynchronous background job processing system that supports:
1. Durable task persistence across server restarts.
2. Bounded concurrency to respect provider rate limits (e.g., 15 RPM for Gemini, 30 RPM for Groq).
3. Zero-cost deployment feasibility (no additional paid infrastructure).
4. Task retries and progress tracking.

---

## 2. Decision Drivers

1. **Zero Financial Cost**: Must run within free-tier resource allocations without requiring paid managed Redis or RabbitMQ.
2. **ACID Transactional Consistency**: Enqueueing evaluation tasks must commit atomically with the evaluation run record.
3. **Simplicity & Operational Footprint**: Minimize moving parts in the initial deployment.
4. **Reliability & Heartbeats**: Ability to detect stalled or crashed tasks and recover gracefully.

---

## 3. Considered Options

- **Option 1: PostgreSQL-backed Queue using `FOR UPDATE SKIP LOCKED` with Async Worker Coroutine**
- **Option 2: Celery with Redis / RabbitMQ**
- **Option 3: In-Memory Python `asyncio.Queue` / BackgroundTasks**
- **Option 4: Cloud Task Services (AWS SQS / Google Cloud Tasks)**

---

## 4. Decision & Justification

**Chosen Option: Option 1 — PostgreSQL-backed Queue using `FOR UPDATE SKIP LOCKED` with Async Worker Coroutine.**

### Key Justifications:
1. **Elimination of Redis Cost & Infrastructure**: Managed cloud Redis instances cost money or require running a dedicated container process that exceeds the 512 MB Render free-tier RAM budget. PostgreSQL is already provisioned for data storage.
2. **ACID Atomicity**: When an evaluation run is created, inserting the run record and its 100 test execution tasks into `job_queue` happens in a single database transaction. If the transaction fails, no orphan jobs are created. Dual-writing to PostgreSQL and Redis introduces split-brain edge cases.
3. **`SKIP LOCKED` High Concurrency**: PostgreSQL's `SELECT ... FOR UPDATE SKIP LOCKED` allows worker coroutines to claim pending jobs atomically with zero lock contention:
   ```sql
   WITH next_job AS (
       SELECT id FROM job_queue
       WHERE status = 'PENDING' AND scheduled_at <= NOW()
       ORDER BY priority DESC, created_at ASC
       LIMIT 1
       FOR UPDATE SKIP LOCKED
   )
   UPDATE job_queue
   SET status = 'RUNNING', locked_at = NOW(), locked_by = :worker_id
   FROM next_job
   WHERE job_queue.id = next_job.id
   RETURNING job_queue.*;
   ```
4. **Rejection of Pure In-Memory `asyncio.Queue`**: In-memory queues lose all in-flight evaluation tasks whenever the container restarts or Render spins down after 15 minutes of inactivity.

---

## 5. Consequences

### Positive:
- $0.00 additional hosting cost.
- Fully transactional, crash-resilient job processing.
- Simple observability: queue depth, throughput, and failures are inspected via standard SQL queries.

### Negative / Trade-offs:
- Polling overhead: Workers poll the database at configured intervals (e.g., every 500ms when tasks exist, back off to 2s when idle).
- Not designed for 100,000+ jobs/sec high-throughput streaming. However, for LLM evaluation matrices (typically hundreds to low thousands of tasks), PostgreSQL easily outperforms requirements. If throughput requirements grow, the queue interface can be swapped for Redis/Celery without changing evaluation domain logic.
