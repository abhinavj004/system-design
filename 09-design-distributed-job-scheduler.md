# System Design 09: Design Distributed Job Scheduler (Airflow / Temporal / Celery)

> **Source**: [Interview with Bunny - Episode 9](https://www.youtube.com/watch?v=hLvB2haod5w)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/09](https://www.interviewwithbunny.com/systemdesign/09)  
> **Topic**: Designing a Highly Scalable, Fault-Tolerant Distributed Job Scheduler (Temporal / Airflow / Celery / Quartz)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Context & Problem Scope
A distributed job scheduler is responsible for accepting, sequencing, dispatching, and monitoring millions of deferred, recurring, or instant computational tasks (e.g., automated billing crons, report generation, video transcoding, DB cleanup, email newsletters) across an elastic fleet of heterogeneous worker nodes.

```
[ User / Service ] ──▶ [ Job Service ] ──▶ [ PostgreSQL DB ]
                                                │
                                                ▼ (Sliding Window Scan)
                                       [ Watcher Service ]
                                                │
                                                ▼
                                    [ Kafka / Redis ZSet ]
                                                │
                                                ▼
                                    [ Job Consumer Engine ]
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │ Worker Executor Fleet │
                                    │ (Docker/Celery/Pods)  │
                                    └───────────────────────┘
```

---

### 1.2 Functional Requirements (FR)
1. **Multi-Modal Job Submission**:
   - **Immediate Execution**: Run right now (`run-now`).
   - **Deferred One-Time Execution**: Run at a specific future UTC timestamp (e.g., tomorrow at 09:00 UTC).
   - **Recurring / Periodic Jobs**: Standard 6-field Unix Cron expressions (e.g., `0 30 5 * * ?` -> every day at 05:30 AM).
2. **Real-Time Job Monitoring & Status Dashboard**: Users can inspect live execution states (`SCHEDULED`, `QUEUED`, `RUNNING`, `SUCCESS`, `FAILED`, `CANCELLED`).
3. **Job Mutation & Cancellation**:
   - Users can update job parameters or reschedule execution times.
   - Users can cancel future jobs as well as **abort currently executing jobs in real time**.
4. **Execution Retries & Dead Letter Queue (DLQ)**: Configurable retry policies with maximum retry limits and dead-lettering for permanently failed jobs.
5. **Execution Logs & Failure Diagnostics**: Capture stdout, stderr, and failure stack traces accessible via a log URL.

*Explicit Out-of-Scope*: Complex Directed Acyclic Graph (DAG) dependency workflows (e.g., Task B runs only after Task A passes) are intentionally excluded to focus on the core distributed scheduling engine.

---

### 1.3 Non-Functional Requirements (NFR)
1. **Peak Throughput**: Support up to **$10,000\text{ jobs executed per second}$** concurrently.
2. **Strict Scheduling Latency**: The system must pick up and trigger jobs within **$\le 2\text{ seconds}$** of their designated scheduled execution time.
3. **At-Least-Once Execution Guarantee**: No scheduled job may ever be silently dropped or forgotten, even across node crashes.
4. **Availability over Consistency (AP in CAP Theorem)**: Prioritize system availability so users and internal services can schedule tasks continuously without rejection. Eventual consistency ($1\text{ to }2\text{ seconds}$ dashboard sync delay) is fully acceptable.
5. **Fault Tolerance & Self-Healing**: Automated detection of crashed worker pods with automatic task reassignment and idempotency safeguards.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput & Concurrency
- **Peak Execution Concurrency**: $10,000\text{ jobs/sec}$.
- **Daily Execution Volume**:
  $$\text{Jobs/Day} \approx 10,000 \times 3600 \times 8 \text{ (active hours)} \approx 288\text{ Million executions/day}$$
- **Job Creation QPS**: Average $\approx 1,000\text{ creations/sec}$; Peak $\approx 5,000\text{ creations/sec}$.

#### Storage Estimations (PostgreSQL Relational DB)
- **Job Metadata Record (`job`)**:
  - `job_id`: 16 bytes (UUID)
  - `name`: 64 bytes
  - `cron_expression`: 32 bytes
  - `payload_json`: 512 bytes
  - `status`, timestamps, retries: 64 bytes
  - Total per Job $\approx 700\text{ bytes}$
  - For $50\text{ Million}$ active job definitions: $50\text{M} \times 700\text{ bytes} \approx 35\text{ GB}$.
- **Job Execution History Record (`job_run`)**:
  - Each run entry $\approx 300\text{ bytes}$.
  - Daily storage: $288\text{M runs/day} \times 300\text{ bytes} \approx 86.4\text{ GB / day}$.
  - 90-day retention window: $86.4\text{ GB} \times 90 \approx 7.7\text{ TB}$ (fits comfortably on partitioned relational storage).

---

## 2. Step 2: Identification of Core Entities

```
+-----------------------------------+           +-----------------------------------+
|                Job                |           |              JobRun               |
+-----------------------------------+           +-----------------------------------+
| job_id (UUID, PK)                 | 1       * | id (BIGSERIAL, PK)                |
| name (VARCHAR)                    |<--------->| job_id (UUID, FK)                 |
| schedule_type (IMMEDIATE/CRON)    |           | status (QUEUED/RUNNING/SUCC/FAIL) |
| status (ACTIVE/PAUSED/CANCELLED)  |           | start_time (TIMESTAMP)            |
| scheduled_time (TIMESTAMP, PART)  |           | end_time (TIMESTAMP)              |
| cron_expression (VARCHAR(64))     |           | modified_time (TIMESTAMP)         |
| payload (JSONB)                   |           | executor_id (VARCHAR)             |
| max_retries (INT)                 |           | attempt_number (INT)              |
| created_at (TIMESTAMP)            |           | error_message (TEXT)              |
| updated_at (TIMESTAMP)            |           | log_url (TEXT)                    |
+-----------------------------------+           +-----------------------------------+
                                                              ▲
                                                              │ Heartbeat & State
                                                +─────────────┴─────────────────────+
                                                |     Worker / Executor Node        |
                                                +-----------------------------------+
                                                | executor_id (VARCHAR)             |
                                                | host_ip (VARCHAR)                 |
                                                | capacity_concurrency (INT)        |
                                                | last_heartbeat (TIMESTAMP)        |
                                                +-----------------------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Create / Schedule a Job
```http
POST /api/v1/jobs
Authorization: Bearer <api_token>
Content-Type: application/json

Request Body:
{
  "name": "weekly_billing_report",
  "schedule_type": "CRON",
  "cron_expression": "0 30 5 ? * FRI",   // Every Friday at 05:30 AM
  "payload": {
    "report_type": "FINANCIAL_LEDGER",
    "region": "APAC",
    "destination_s3": "s3://company-reports/apac/ledger/"
  },
  "max_retries": 3,
  "timeout_seconds": 900
}

Response: 201 Created
{
  "job_id": "job_daily_report_01",
  "name": "weekly_billing_report",
  "status": "SCHEDULED",
  "next_run_time": "2026-10-23T05:30:00Z",
  "created_at": "2026-10-18T10:00:00Z"
}
```

---

### 3.2 Trigger Immediate Execution (`run-now`)
```http
POST /api/v1/jobs/job_daily_report_01/run-now
Authorization: Bearer <api_token>

Response: 202 Accepted
{
  "job_id": "job_daily_report_01",
  "run_id": 994812,
  "status": "QUEUED",
  "queued_at": "2026-10-18T10:05:02Z"
}
```

---

### 3.3 Inspect Live Execution Status
```http
GET /api/v1/jobs/job_daily_report_01/status
Authorization: Bearer <api_token>

Response: 200 OK
{
  "job_id": "job_daily_report_01",
  "current_run": {
    "run_id": 994812,
    "status": "RUNNING",
    "attempt_number": 1,
    "start_time": "2026-10-18T10:05:04Z",
    "last_heartbeat": "2026-10-18T10:07:14Z",
    "executor_id": "worker-pod-az2-881a"
  },
  "previous_runs_summary": {
    "total_success": 48,
    "total_failed": 1
  }
}
```

---

### 3.4 Cancel a Scheduled or Currently Running Job
```http
POST /api/v1/jobs/job_daily_report_01/cancel
Authorization: Bearer <api_token>

Response: 200 OK
{
  "job_id": "job_daily_report_01",
  "status": "CANCELLED",
  "message": "Active execution interrupted; upcoming schedules revoked."
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Clients / Admin Dashboard / Microservices ]
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
│         (Auth, Rate Limiting, Route Discrimination)         │
└─────────────────────────────────────────────────────────────┘
          │                                           │
          │ (POST / PUT / DELETE Mutations)           │ (GET Status / Detail)
          ▼                                           ▼
┌───────────────────────┐                  ┌───────────────────────┐
│     Job Service       │                  │  Job Search Service   │
│ (Mutations & Run-Now) │                  │ (Stateless Read Pods) │
└───────────────────────┘                  └───────────────────────┘
          │           │                               │
          │           │ (Fast-Path Immediate Job)     │
          │           └──────────────┐                │ (Read / Status)
          ▼                          │                ▼
┌───────────────────────┐            │     ┌───────────────────────┐
│ Kafka Ingestion Queue │            │     │ PostgreSQL Primary DB │
└───────────┬───────────┘            │     │  (job & job_run tabs) │
            │                        │     └───────────────────────┘
            ▼                        │                ▲
┌───────────────────────┐            │                │ (Batch Read Every 20s)
│ Ingest Consumer Pods  │            │                │
└───────────┬───────────┘            │     ┌───────────────────────┐
            │                        │     │    Watcher Service    │
            ▼                        │     │(Sliding Window Puller)│
┌───────────────────────┐            │     └──────────┬────────────┘
│ PostgreSQL DB Write   │            │                │
└───────────────────────┘            │                │ (Push Next 5m Jobs)
                                     ▼                ▼
                       ┌────────────────────────────────────────┐
                       │   Apache Kafka Execution Pipeline      │
                       │   Topics: [run-queue, retry, dlq]      │
                       └──────────────────┬─────────────────────┘
                                          │
                                          ▼
                               ┌───────────────────────┐
                               │ Job Consumer Service  │
                               └──────────┬────────────┘
                                          │
                       ┌──────────────────┴──────────────────┐
                       │                                     │
                       ▼                                     ▼
           ┌───────────────────────┐             ┌───────────────────────┐
           │ Worker Executor 1     │             │ Worker Executor N     │
           │ (Runs Python/Docker)  │             │ (Runs Python/Docker)  │
           └───────────┬───────────┘             └───────────┬───────────┘
                       │                                     │
                       ├─────────────────┬───────────────────┘
                       │                 │ (Heartbeat Every 10s via Kafka)
                       ▼                 ▼
              ┌────────────────┐  ┌──────────────┐
              │  Redis Cache   │  │ S3 Log Store │
              │(Cancel Signals)│  │ (Stdout/Err) │
              └────────────────┘  └──────────────┘
```

### Architectural Layer Separation
1. **CQRS Boundary (Job Service vs. Job Search Service)**:
   - Bunny explicitly separates write mutations (`create`, `update`, `cancel`) from high-frequency dashboard reads (`status`, `metrics`), preventing dashboard query spikes from stalling ingestion pipelines.
2. **Ingestion Queue**: Offloads incoming bursts of user creations to an asynchronous Kafka topic, protecting PostgreSQL from write starvation during traffic spikes.
3. **Watcher Service**: The brain of the temporal scheduler; scans PostgreSQL using a sliding time window and loads impending tasks into execution queues.
4. **Execution Pipeline**: Decoupled Kafka topics managing pristine jobs, retry delays, and poison-pill DLQ messages.
5. **Worker Executor Fleet**: Isolated execution workers running Docker containers or Celery threads, checking Redis for cancellation interrupts and reporting heartbeats.

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: The Watcher Service & The Sliding Window Scan

A naive scheduler queries the database every second:
```sql
-- The Naive Disaster Query (Runs 1 time per second across 50M rows):
SELECT * FROM job WHERE scheduled_time <= NOW() AND status = 'SCHEDULED';
```
**Why this destroys production systems**:
- Executing table scans every second locks database pages, exhausts connection pools, and spikes CPU to 100%.

#### Bunny's Solution: 20-Second Sliding Window Batching
Instead of scanning every second, the **Watcher Service** executes a range query every **20 seconds** to fetch all jobs scheduled to run in the next **5-minute sliding window**:

```sql
SELECT job_id, scheduled_time, payload, max_retries 
FROM job 
WHERE scheduled_time BETWEEN NOW() AND NOW() + INTERVAL '5 minutes'
  AND status = 'SCHEDULED';
```

```
Timeline:
[NOW] ────────────────────── 5-Minute Window ──────────────────────▶ [NOW + 5m]
  │                             │                                       │
  ├── Job A (due at +10s) ──────┤                                       │
  ├── Job B (due at +45s) ──────┤                                       │
  └── Job C (due at +3m) ───────┴───────────────────────────────────────┘
  (Pushed to Kafka execution queue with timestamp metadata)
```

1. The Watcher Service reads the batch into memory.
2. It pushes the batch into the **Kafka `run-queue`** topic.
3. It immediately updates the state in the `job_run` table to **`QUEUED`**.
4. **State Checkpointing in Redis**:
   To prevent duplicate pulls or gaps if the Watcher Service crashes, it records its progress in Redis:
   ```bash
   SET scheduler:watcher:last_scanned_timestamp "2026-10-18T10:05:00Z"
   ```
   When a replacement Watcher pod boots up, it reads `last_scanned_timestamp` and seamlessly resumes without missing a beat.

---

### 5.2 Deep Dive 2: Worker Heartbeating & Dead-Worker Detection

Long-running jobs (e.g., 10 minutes) can fail midway if the underlying Kubernetes worker pod runs Out-Of-Memory (OOM) or encounters a kernel panic. If the worker abruptly dies, it cannot send a failure message.

```
[ Worker Executor ] ──(Every 10s Heartbeat)──▶ [ Kafka Status Topic ] ──▶ [ Consumer ] ──▶ [ job_run DB ]
                                                                                               │
                                                                                    (Updates modified_time)
                                                                                               ▲
                                                                                               │
                                               [ Watcher Service ] ────────────────────────────┘
                                               Detects: status = 'RUNNING' AND 
                                                        modified_time < NOW() - 15 seconds!
```

#### The 15-Second Heartbeat Protocol:
1. When a worker begins executing a task, it sets `job_run.status = 'RUNNING'` and `job_run.start_time = NOW()`.
2. A lightweight background thread inside the worker issues a heartbeat ping every **10 seconds** via Kafka:
   ```json
   {
     "event": "HEARTBEAT",
     "job_id": "8a72b831",
     "run_id": 994812,
     "executor_id": "worker-pod-az2-881a",
     "timestamp": 1729253410
   }
   ```
3. The consumer updates `job_run.modified_time = NOW()`.
4. **The Watcher Sweep (Dead-Worker Reaper)**:
   Every 20 seconds, along with scanning for new jobs, the Watcher Service sweeps for abandoned tasks:
   ```sql
   SELECT id, job_id, attempt_number FROM job_run 
   WHERE status = 'RUNNING' 
     AND modified_time < NOW() - INTERVAL '15 seconds';
   ```
   Any matching job is officially declared **orphaned**. The Watcher immediately increments `attempt_number`, updates status to `QUEUED`, and republishes the task to Kafka for reassignment to a healthy worker node.

---

### 5.3 Deep Dive 3: Retries, Exponential Backoff & Dead Letter Queue (DLQ)

When a task fails with an unhandled exception:

```
[ Worker ] ──(Catches Exception)──▶ [ Push to Kafka retry-topic ]
                                            │
                                            ▼
                             [ Exponential Backoff Delay ]
                             Delay = Base (2s) * 2^(attempt)
                                            │
               ┌────────────────────────────┴────────────────────────────┐
               ▼ (attempt <= max_retries)                                ▼ (attempt > max_retries)
     [ Re-queue into run-queue ]                              [ Push to Dead Letter Queue (DLQ) ]
                                                                             │
                                                                             ▼
                                                                [ job_run.status = 'FAILED' ]
                                                                [ Alert Ops via PagerDuty ]
```

1. **Attempt Tracking**:
   - `Attempt 1`: Fails -> Worker publishes to `retry-topic` with delay $= 2^1 = 2\text{ seconds}$.
   - `Attempt 2`: Fails -> Delay $= 2^2 = 4\text{ seconds}$.
   - `Attempt 3`: Fails -> Delay $= 2^3 = 8\text{ seconds}$.
2. **Exhaustion & Poison Pill Handling**:
   - Once `attempt_number >= max_retries`, the task is routed to the **Dead Letter Queue (DLQ)**.
   - Status updates permanently to `FAILED`.
   - The error stack trace and logs are uploaded to Amazon S3, and the URL is saved in `job_run.log_url`.

---

### 5.4 Deep Dive 4: Aborting a Currently Running Job (Redis Cancellation Signal)

Canceling a job scheduled for tomorrow is easy (just set `status = 'CANCELLED'` in the database).  
**The Hard Problem**: How do you safely stop a task that is **actively running** inside a worker process?

```
User triggers Cancel ──▶ [ Job Service ]
                              │
                              ▼
                 [ Redis Atomic Set with TTL ]
                 SET job:cancel:8a72b831 "true" EX 60
                              │
                              ▲ (Checked every 2s)
                     [ Worker Executor Thread ]
                     if (redis.get("job:cancel:" + id) == "true") {
                         thread.interrupt();
                         rollbackTransaction();
                         cleanupTempFiles();
                         updateStatus("CANCELLED");
                     }
```

1. The user calls `POST /api/v1/jobs/{id}/cancel`.
2. `JobService` writes an atomic key into **Redis Cluster** with a $60\text{-second TTL}$:
   ```bash
   SETNX job:cancel:8a72b831-29e1 "true"
   EXPIRE job:cancel:8a72b831-29e1 60
   ```
3. Workers executing tasks run a lightweight cancellation watcher thread that inspects Redis every 2 seconds.
4. On detection of `"true"`, the worker raises an `InterruptedException`, closes active DB sockets, cleans up scratch disks, and emits a `CANCELLED` status event.
5. **Why TTL is mandatory**: If the job finishes naturally before the cancel is processed, the ephemeral Redis key expires automatically after 60 seconds, preventing memory leaks in Redis.

---

### 5.5 Deep Dive 5: The Kafka FIFO Trap & The Redis Sorted Set (`ZSET`) Alternative

Bunny highlights a subtle interview trap:

#### The Kafka Head-of-Line Blocking Problem
- Kafka partitions are strictly **FIFO (First-In, First-Out)** append-only logs.
- If the Watcher Service pre-loads 1,000 tasks due across the next 5 minutes into Kafka, and a user suddenly calls **`run-now`** on a critical job, the immediate job is appended to the **tail** of the Kafka partition!
- Result: The "run-now" job sits stuck waiting behind 1,000 future jobs, violating the $\le 2\text{s}$ latency SLA.

```
Kafka Partition: [ Job 1 (due 10:01) ] [ Job 2 (due 10:02) ] ... [ Run-Now Job (STUCK AT END!) ]
```

#### Bunny's Solution: Redis Sorted Sets (`ZSET`) for Temporal Priority Scheduling

```
Redis Sorted Set: ZADD scheduler:timeline <timestamp_epoch> <job_run_id>

Score: 1729253410  -> Job 1 (due 10:01:50)
Score: 1729253400  -> Run-Now Job (due 10:00:00) <- AUTOMATICALLY INSERTED AT HEAD!
Score: 1729253460  -> Job 2 (due 10:02:40)
```

1. Jobs are inserted using `ZADD` with their intended execution epoch millisecond as the **score**:
   ```bash
   ZADD scheduler:pending_jobs 1729253400000 "job_run_994812"
   ```
2. When a `run-now` request arrives, its score is set to `CURRENT_EPOCH_MS`. Redis automatically positions it at the **very front** of the sorted set ($O(\log N)$ insertion).
3. The consumer service pulls ready tasks using atomic range retrieval:
   ```bash
   ZRANGEBYSCORE scheduler:pending_jobs -inf <CURRENT_TIMESTAMP> LIMIT 0 100
   ZREMRANGEBYSCORE scheduler:pending_jobs -inf <CURRENT_TIMESTAMP>
   ```
4. This completely eliminates head-of-line blocking and guarantees sub-second dispatch for immediate tasks.

---

### 5.6 Deep Dive 6: PostgreSQL Database Indexing & Table Partitioning

Because the database handles millions of state updates daily, table indexing must strictly match access patterns.

```sql
-- 1. Range Partitioning on 'job' table by scheduled_time (Monthly Partitions)
CREATE TABLE job (
    job_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    schedule_type VARCHAR(32) NOT NULL,
    status VARCHAR(32) NOT NULL,
    scheduled_time TIMESTAMP WITH TIME ZONE NOT NULL,
    cron_expression VARCHAR(64),
    payload JSONB,
    max_retries INT DEFAULT 3,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (job_id, scheduled_time)
) PARTITION BY RANGE (scheduled_time);

-- 2. Primary Index on job_id for single-point lookups
CREATE INDEX idx_job_lookup ON job (job_id);

-- 3. Composite Index on job_run for the Watcher's Dead-Worker Sweep
CREATE INDEX idx_job_run_dead_worker_sweep 
ON job_run (status, modified_time) 
WHERE status = 'RUNNING';

-- 4. Single-point index for run tracking
CREATE INDEX idx_job_run_job_id ON job_run (job_id);
```

> **Why Partial Index `WHERE status = 'RUNNING'` is Critical**:  
> Out of 100 million historical records in `job_run`, only $\approx 10,000$ are in the `'RUNNING'` state at any single moment. The partial index contains only those 10,000 entries, allowing the Watcher's 15-second dead-worker sweep query to execute in **under 2 milliseconds**!

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you prevent the same job from being scheduled twice by multiple watcher instances?

Distributed locking with Redis:

- **Lock acquisition - before scheduling job, watcher attempts SET lock**:
  job:{job_id}:schedule {watcher_id} NX EX 60 (set if not exists, 60s expiry),

- **Point 2**: Lock success - if returns 1, watcher proceeds to publish job to Kafka, updates last_polled_time in DB, releases lock,

- **Point 3**: Lock failure - if returns 0, another watcher already processing this job, skip to next job,

- **Point 4**: Database-level protection - UPDATE jobs SET last_polled_time = NOW() WHERE job_id = {id} AND last_polled_time < (NOW - 30s), only updates if not recently polled,

- **Auto-expiry - lock expires in 60s if watcher crashes, prevents deadlock. Alternative**:
  Use Kafka as single-source scheduling - only 1 watcher publishes to Kafka (leader election via Zookeeper), consumers handle rest. Trade-off: Redis lock is faster, simpler vs Kafka leader election is more robust for multi-datacenter. Example: Job J1 next_run_time=10:00 AM → Watcher-1 and Watcher-2 poll at same time → Watcher-1 acquires lock → publishes J1 to Kafka → releases lock → Watcher-2's lock fails → skips J1.

---

### Q2: How do you handle jobs that are scheduled while the scheduler is down?

Backfill mechanism:

- **Point 1**: Detection - on watcher restart, query jobs WHERE next_run_time < NOW() AND status IN ('scheduled', 'running'), identifies missed jobs,

- **Point 2**: Categorization - jobs with next_run_time in last 2 hours = recent misses (high priority), >2 hours = stale (may skip or backfill based on policy),

- **Point 3**: Backfill execution - for each missed job, create JobRun with scheduled_at = original_next_run_time, status='backfill', publish to Kafka 'run' topic,

- **Point 4**: Rate limiting - publish backfill jobs at 100 jobs/sec to prevent overwhelming executors (if 1000 missed jobs, takes 10s to queue all),

- **Point 5**: SLA check - if job has SLA (must run within 1 hour of schedule), skip if SLA breached, mark as 'missed_sla', alert user,

- **Recurring jobs - for cron jobs, calculate all missed runs**:
  if daily job down for 3 days, create 3 JobRuns for Day 1, 2, 3, or skip to most recent (configurable). Example: Scheduler down from 10:00-12:00, Job A scheduled hourly (10:00, 11:00, 12:00) → on restart at 12:05, detect 2 missed runs → backfill creates runs for 10:00, 11:00 → publish both to Kafka → executors process → update next_run_time to 13:00.

---

### Q3: How do you implement retry mechanism with exponential backoff?

Multi-stage retry pipeline:

- **Point 1**: Initial failure - executor catches exception, checks job.retry_count (e.g., 3) vs current attempt

- **Point 1**: , if retries remaining, proceed,

- **Delay calculation - retry_delay = base_delay * (2 ^ (attempt - 1)) + random(0, base_delay/2), example**:
  attempt 1 → 60s, attempt 2 → 120s + jitter (0-30s), attempt 3 → 240s + jitter,

- **Publish to retry topic - executor publishes to Kafka 'retry' with {job_id, run_id, payload, attempt**:
  2, scheduled_retry_at: NOW() + retry_delay},

- **Retry consumer - dedicated consumer group reads 'retry' topic, for each message**:
  calculate wait_time = scheduled_retry_at - NOW(), if wait_time > 0: sleep(wait_time), then republish to 'run' topic,

- **Point 5**: Final failure - if attempt > retry_count, publish to 'dead' topic, update DB status='failed_permanently', send alert via SNS/email,

- **Jitter rationale - prevents thundering herd (100 jobs fail at same time, all retry at exact same moment without jitter). Alternative**:
  Use Kafka delayed message feature (Kafka 3.0+) or SQS delay queue for retry scheduling. Example: Job fails at 10:00 → retry_count=3 → attempt 1 fails → retry at 10:01 (60s) → attempt 2 fails → retry at 10:03 (120s) → attempt 3 fails → retry at 10:07 (240s) → attempt 4 (exceeds retry_count) → dead letter queue.

---

### Q4: How do you handle job dependencies in a DAG (Directed Acyclic Graph)?

Dependency resolution with topological ordering:

- **Storage - Job table has dependencies column**:
  job_b.dependencies = [job_a_id, job_c_id] means B depends on A and C,

- **Cycle detection - on job creation/update, run DFS (Depth-First Search) starting from new job, if visit same node twice, cycle exists, reject with 'Circular dependency detected**:
  A→B→C→A',

- **Scheduling check - watcher before scheduling Job B, queries**:
  SELECT COUNT(*) FROM job_runs WHERE job_id IN (job_a_id, job_c_id) AND status='success' AND DATE(scheduled_at) = TODAY, if count != 2 (missing dependencies), skip Job B this cycle,

- **Trigger on completion - when Job A completes successfully, publish 'job.completed' event to Kafka with {job_id**:
  A, run_date},

- **Point 5**: Dependency watcher - separate service consumes 'job.completed', queries jobs WHERE dependencies CONTAINS job_a_id, checks if all dependencies now met, if yes, publishes dependent jobs to 'run' topic,

- **Parallel execution - jobs at same level (C and D both depend only on B) published simultaneously for parallel execution. Alternative**:
  Airflow's approach - precompute DAG at deployment, store as graph in memory, traverse on each run. Example: DAG: A → B → C, D (B depends on A, C and D depend on B) → A runs at 10:00, succeeds → B scheduled for 10:05, succeeds → C and D both scheduled for 10:10 in parallel.

---

### Q5: What happens if an executor crashes while running a job?

Executor failure detection and recovery:

- **Heartbeat monitoring - executors publish heartbeat to Redis executor**:
  {id}:heartbeat every 10s with timestamp,

- **Point 2**: Watcher health check - separate health check service polls Redis every 30s, checks if any executor's heartbeat > 60s old (3 missed heartbeats), marks as 'unhealthy',

- **Point 3**: Job identification - query JobRuns WHERE executor_id = {unhealthy_id} AND status='running', finds orphaned jobs,

- **Rescheduling - for each orphaned job**:
  update status='executor_died', create new JobRun with attempt = old_attempt + 1 (counts as retry), publish to Kafka 'run' topic,

- **Cleanup - remove executor**:
  {id} from Redis executors:available pool, alert ops team,

- **Idempotency requirement - jobs must be idempotent (safe to run multiple times) because crashed job may have partially completed before executor died. Example**:
  Executor E1 running Job J1 (writing to database) → E1 crashes at 50% completion → heartbeat stops → health check detects after 60s → Job J1 rescheduled to Executor E2 → E2 re-runs full job → job's code must handle 'partial completion' scenario (check if data already written, skip or upsert). Prevention: Use distributed locks for critical sections (job locks specific resource before modifying).

---

### Q6: How do you implement priority-based job scheduling?

Multi-tier priority queue system:

- **Point 1**: Priority definition - jobs have priority field (1-10, 10=highest), stored in Job table,

- **Topic separation - Kafka topics**:
  high_priority_run (priority 8-10), medium_priority_run (5-7), low_priority_run (1-4),

- **Point 3**: Watcher routing - when scheduling job, publish to appropriate topic based on job.priority,

- **Point 4**: Consumer allocation - high priority topic has 50 consumer instances, medium has 30, low has 20 (2.5x more resources for high priority),

- **Point 5**: Within-topic ordering - jobs in same priority topic processed FIFO (First In First Out) using Kafka partitioning,

- **Starvation prevention - if low priority jobs waiting >1 hour, temporarily boost to medium priority (aging algorithm). Alternative**:
  Single topic with priority header, consumers poll high-priority partitions more frequently. Example: Job A (priority 9) and Job B (priority 3) scheduled at same time → A published to high_priority_run, B to low_priority_run → high topic has 50 consumers, low has 20 → A assigned to executor in <1s, B waits 5s for executor. Trade-off: More topics = more complexity but better isolation, single topic = simpler but consumers need priority-aware logic.

---

### Q7: How do you handle time zone conversions for scheduled jobs?

Server-side time zone normalization:

- **Storage - all timestamps in DB stored as UTC (jobs.next_run_time = '2025-01-21T00**:
  00:00Z'),

- **User input - job creation accepts schedule with time zone**:
  {schedule: '0 0 * * *', timezone: 'America/New_York'},

- **Conversion on save - backend converts to UTC using timezone library (pytz in Python)**:
  cron '0 0 * * *' in EST (UTC-5) → UTC '0 5 * * *', stores UTC version,

- **Display - when user views job, convert back to their timezone for display**:
  '2025-01-21T00:00:00Z' UTC → '2025-01-20T19:00:00' EST,

- **Point 5**: DST handling - recalculate next_run_time when daylight saving time changes (March/November), e.g., job '9 AM EST' shifts 1 hour in UTC,

- **Watcher logic - always works in UTC, no timezone awareness needed for scheduling. Complexity**:
  Recurring jobs across DST boundary - job scheduled for '2 AM EST daily' on DST start day (2 AM doesn't exist), skip to 3 AM or run at 1 AM (configurable). Example: User in India (IST, UTC+5:30) schedules job for daily at midnight local time → backend stores as UTC 18:30 previous day → watcher at 18:30 UTC triggers job → user sees '00:00 IST' in UI. Alternative: Store timezone with each job, watcher converts on-the-fly (more flexible but complex).

---

### Q8: How do you implement job cancellation for already running jobs?

Graceful cancellation protocol:

- **Point 1**: User request - POST /jobs/{job_id}/cancel, service checks status, if 'running', publish to Kafka 'cancel' topic with {job_id, run_id},

- **Point 2**: Executor subscription - all executors subscribe to 'cancel' topic with consumer group per executor (ensures all get message),

- **Point 3**: Executor matching - executor checks if run_id matches currently running job, if yes, initiates cancellation,

- **Point 4**: Signal sending - executor sends SIGTERM (signal 15) to job process, allows graceful shutdown (close connections, save state),

- **Point 5**: Timeout - if process doesn't exit in 30s, send SIGKILL (signal 9) to force terminate,

- **Point 6**: Status update - executor updates DB status='cancelled', end_time=NOW(), publishes 'job.cancelled' event,

- **Cleanup - release any distributed locks held by job, rollback transactions if applicable. Edge case**:
  Job already completed before cancellation received → executor ignores cancel message, DB status remains 'success'. Alternative: Use shared memory flag (Redis cancel:{run_id} = true), job periodically checks flag and exits if set (requires job code cooperation). Example: Long-running ETL job (2 hours) → user cancels after 30 min → cancel event published → executor sends SIGTERM → job's cleanup handler runs (commits partial data) → exits gracefully → status='cancelled', user can retry or analyze partial results.

---

### Q9: How do you prevent job duplication on executor restart?

Idempotent job execution with state tracking:

- **Point 1**: Job uniqueness - each JobRun has unique run_id, executor stores currently_running_job = run_id in Redis on start,

- **Point 2**: Restart detection - on executor restart, check Redis for currently_running_job, if exists, it was killed mid-execution,

- **Point 3**: Status reconciliation - query DB for run_id status, if still 'running', mark as 'executor_died' (executor crashed), don't re-execute (prevents duplication),

- **Point 4**: Kafka offset management - executor commits Kafka offset AFTER job completes, on restart, uncommitted messages re-delivered (at-least-once),

- **Point 5**: Idempotency enforcement - job_id + scheduled_at combination ensures uniqueness, if executor tries to start job with same (job_id, scheduled_at), DB unique constraint fails, skip execution,

- **Exactly-once attempt - use Kafka transactions (Kafka 0.11+) with transactional.id per executor, ensures message processed and offset committed atomically. Trade-off**:
  At-least-once (simpler, requires idempotent jobs) vs exactly-once (complex, guarantees no duplication but slower). Example: Executor E1 running Job J1 (run_id=R1) → E1 crashes → E1 restarts → checks Redis, finds currently_running_job=R1 → queries DB, R1 status='running' → updates R1 to 'executor_died', doesn't re-run → health check reschedules R1 with new run_id=R2 → counted as retry.

---

### Q10: What's your strategy for monitoring and alerting on job failures?

Multi-level monitoring and alerting:

- **Metrics collection - executors publish metrics to Prometheus/CloudWatch**:
  job_success_count, job_failure_count, job_duration_ms, jobs_in_queue (by priority),

- **Failure thresholds - alert rules**:
  single job failure rate >50% (6 of last 10 runs failed) = critical alert, overall system failure rate >10% in 1 hour = warning,

- **Point 3**: SLA monitoring - if job has SLA (must complete within 2 hours of schedule), alert if end_time - scheduled_at > 2 hours,

- **Point 4**: Dead letter queue size - alert if dead topic has >100 messages (indicates systemic issue, not isolated failures),

- **Point 5**: Executor health - alert if <30% executors healthy (capacity issue),

- **Dashboard - Grafana dashboard showing**:
  jobs by status (pie chart), execution time trends (line graph), failure reasons (top 10 errors). Alert channels:

- **Point 7**: PagerDuty for critical (job X failed 5 times),

- **Point 8**: Slack for warnings (10% system failure rate),

- **Email daily digest of all failures. Example**:
  ETL job fails 3 times in row → alert 'Job ETL_Pipeline failing (3/3 attempts), error: DB connection timeout' sent to Slack → on-call engineer investigates, finds DB overloaded → scales DB → job succeeds on retry 4.

---

## 7. Interview Summary & Architectural Decision Matrix

| Architectural Dimension | Bunny's Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Scheduler Polling Mechanism** | 20-Second Sliding Window Scan | 1-Second DB Polling | Prevents continuous DB connection saturation and page-lock thrashing; batches upcoming tasks safely. |
| **Dead-Worker Detection** | 10s Heartbeat + 15s Sweep (`job_run.modified_time`) | OS Ping / Keep-Alive | Detects frozen process loops and application-level deadlocks that OS network pings miss entirely. |
| **Live Job Cancellation** | Redis In-Memory Flag with TTL | DB Polling inside Worker | In-memory lookup ($< 1\text{ms}$) avoids hammering SQL databases while threads poll for interrupts. |
| **Temporal Queue Engine** | Redis Sorted Set (`ZSET`) | Native Kafka Log | Eliminates head-of-line blocking; enables instant $O(\log N)$ insertion of `run-now` jobs ahead of deferred jobs. |
| **Relational Indexing** | Partial Composite Index on `job_run` | Full Table B-Tree Index | Reduces index tree size by $99.9\%$, keeping heartbeat checks in RAM without disk I/O bottlenecks. |
| **Job Mutation Routing** | CQRS (Job Service vs Search Service) | Monolithic Shared API Pod | Shields read-only status dashboards from write-heavy job creation and batch scheduling spikes. |
