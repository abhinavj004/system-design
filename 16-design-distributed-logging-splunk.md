# System Design 16: Design Distributed Logging System (Splunk / Logstash / ELK)

> **Source**: [Interview with Bunny - Episode 16](https://www.youtube.com/watch?v=cqs5t7zqzCk)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/16](https://www.interviewwithbunny.com/systemdesign/16)  
> **Topic**: Designing an Enterprise Distributed Logging & Observability Platform (Splunk, ELK Stack, OpenObserve)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Problem Scope & Core Architecture
In modern microservice ecosystems, thousands of distributed containers across multi-cloud environments generate billions of log lines daily. Software engineers and SREs require a centralized, highly reliable platform to collect, parse, normalize, store, search, and alert on system events in near real time.

The system must handle heterogeneous sources, support both real-time streaming and offline batch uploads, protect downstream search clusters from write saturation, and implement a **cost-tiered retention lifecycle** (Hot, Warm, Cold).

---

### 1.2 Functional Requirements (FR)
1. **Multi-Source Log Ingestion**:
   - **Real-Time Stream Ingestion**: Collect logs from thousands of Kubernetes pods, containers, and bare-metal servers worldwide via lightweight local agents (e.g., Fluent Bit, Logstash, Vector).
   - **Offline / Batch File Ingestion**: Ingest legacy application log dumps and offline appliance archives via bulk file upload.
2. **Log Parsing, Normalization & Sanitization**:
   - Ingest unformatted raw text, log4j stack traces, and JSON payloads.
   - Parse unstructured text using Grok/Regex patterns into standardized schemas.
   - Enrich logs with contextual metadata (`cluster_id`, `environment`, `hostname`, `service_name`).
   - Mask sensitive PII data (e.g., credit card numbers, auth tokens, passwords).
3. **Full-Text Search & Log Analytics Dashboard**:
   - Search across billions of log lines using keyword search, Boolean operators (`AND`, `OR`), exact phrases, and key-value filters (`service:checkout AND status:500`).
   - Filter logs by timestamp range, logging level (`DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`), and distributed trace ID.
4. **Real-Time Log Tailing (Live Tail)**:
   - Provide developers with live streaming log outputs for specific services via full-duplex WebSocket connections.
5. **Real-Time Anomaly Alerting**:
   - Evaluate incoming logs against predefined alerting rules (e.g., trigger PagerDuty/Slack if `ERROR` count on `PaymentService` exceeds 20 within 5 minutes).
6. **Multi-Tiered Retention & Cost Optimization**:
   - Automatic lifecycle transitions across **Hot Storage** (Fast queryable search), **Warm Storage** (Structured lookups), and **Cold Storage** (Compressed archival).

---

### 1.3 Non-Functional Requirements (NFR)
1. **High Ingestion Throughput**: Ingest tens of millions of log events per hour ($\approx 50,000 \text{ to } 100,000 \text{ logs/sec}$ peak write rate).
2. **Low Query & Search Latency**:
   - Log availability latency (time from application emission to search visibility): **$< 2 - 5\text{ seconds}$** (near real time).
   - Hot tier search response time: **$< 200\text{ms}$** (p95).
3. **High Availability (AP under CAP Theorem)**:
   - The ingestion tier must **never block or crash the host application**. If the centralized logging platform experiences degraded performance, local agents buffer logs locally on disk.
   - System guarantees **Availability over Strong Consistency**; a few seconds of indexing delay is completely acceptable.
4. **Fault Tolerance & Zero Log Loss**:
   - Critical system logs must not be lost during network partitions or node failovers.
   - End-to-end durability ensured via local agent disk buffers, Apache Kafka clustering, and distributed replication.
5. **Cost-Efficient Scalability**: Prevent runaway cloud storage and indexing costs by strictly limiting the life of full-text inverted indexes.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Ingestion Throughput
- **Scale**: 5,000 microservices across 50 Kubernetes clusters globally.
- Each service generates an average of 10 log lines per second.
  $$\text{Average Ingestion QPS} = 5,000 \times 10 = 50,000 \text{ logs/sec}$$
  $$\text{Peak Ingestion QPS} = 2 \times \text{Average} = 100,000 \text{ logs/sec}$$

#### Storage Volume
- Average raw log line size (including JSON metadata) $\approx 500 \text{ bytes}$.
- **Daily Raw Log Volume**:
  $$\text{Daily Logs} = 50,000 \text{ logs/sec} \times 86,400 \text{ sec} = 4.32 \text{ Billion logs/day}$$
  $$\text{Daily Storage} = 4.32 \times 10^9 \times 500 \text{ bytes} \approx 2.16 \text{ TB / day}$$
- **Monthly Storage**: $2.16 \text{ TB} \times 30 \approx 64.8 \text{ TB / month}$.
- **Yearly Archival**: $\approx 780 \text{ TB / year}$ (uncompressed).

#### Tiered Storage Strategy (Cost Optimization)
| Tier | Storage Engine | Retention Window | Storage Footprint | Cost Profile |
| :--- | :--- | :--- | :--- | :--- |
| **Hot Tier** | Elasticsearch / OpenSearch | **14 Days** | $\approx 30 \text{ TB}$ (with replica indexes) | Expensive SSD RAM & IOPS |
| **Warm Tier** | Apache Cassandra / ScyllaDB | **45 Days** | $\approx 97 \text{ TB}$ (compressed SSTables) | Mid-tier persistent disk |
| **Cold Tier** | AWS S3 / GCS (Snappy Parquet) | **1+ Years** | $\approx 150 \text{ TB}$ (5:1 compression) | Ultra-cheap object storage |

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|    Organization    |        |      LogAgent      |        |      LogEvent      |
+--------------------+        +--------------------+        +--------------------+
| org_id (PK, UUID)  | 1    * | agent_id (PK, UUID)| 1    * | event_id (PK, UUID)|
| org_name (VARCHAR) |<------>| org_id (FK)        |<------>| org_id (FK)        |
| client_token (VAR) |        | cluster_id (VAR)   |        | service_name (VAR) |
| token_ttl (TIME)   |        | host_ip (VARCHAR)  |        | trace_id (VARCHAR) |
| tier (ENTERPRISE)  |        | version (VARCHAR)  |        | level (ENUM:ERROR) |
+--------------------+        | status (HEALTHY)   |        | timestamp (TIME)   |
                              | last_heartbeat(TIM)|        | message (TEXT)     |
                              +--------------------+        | metadata (JSONB)   |
                                                            +--------------------+
                                                                      │ 1
                                                                      │
                                                                      ▼ *
+--------------------+        +--------------------+        +--------------------+
|     AlertRule      |        |   AlertIncident    |        |   BatchUploadJob   |
+--------------------+        +--------------------+        +--------------------+
| rule_id (PK, UUID) | 1    * | incident_id (PK)   |        | job_id (PK, UUID)  |
| org_id (FK)        |<------>| rule_id (FK)       |        | org_id (FK)        |
| service_name (VAR) |        | trigger_value (INT)|        | file_url (TEXT)    |
| condition (JSONB)  |        | status (OPEN/ACK)  |        | status (PROCESSED) |
| notification_target|        | created_at (TIME)  |        | total_lines (BIG)  |
+--------------------+        +--------------------+        +--------------------+
```

---

## 3. Step 3: API Design (Public REST & Streaming Interfaces)

### 3.1 Register Agent & Exchange Credentials
Lightweight DaemonSet agents authenticate during container startup:
```http
POST /api/v1/agents/register
Authorization: Bearer <org_provisioning_token>
Content-Type: application/json

Request Body:
{
  "cluster_id": "k8s-prod-us-east-1",
  "hostname": "worker-node-104",
  "agent_type": "FLUENT_BIT",
  "agent_version": "2.2.0",
  "environment": "PRODUCTION"
}

Response: 201 Created
{
  "agent_id": "agnt_8812-44af-9912",
  "access_token": "jwt_eyJhGciOi...",
  "token_ttl_seconds": 86400,
  "heartbeat_interval_seconds": 30,
  "config_signature": "sha256_b4129a..."
}
```

---

### 3.2 Ingest Log Stream (High-Throughput Batch Payload)
Agents aggregate local container logs and flush compressed micro-batches every 500ms:
```http
POST /api/v1/logs/ingest
Authorization: Bearer <agent_access_token>
Content-Encoding: gzip
Content-Type: application/json

Request Body (Compressed):
{
  "agent_id": "agnt_8812-44af-9912",
  "sent_at": "2026-10-18T14:00:00.120Z",
  "events": [
    {
      "timestamp": "2026-10-18T14:00:00.010Z",
      "service_name": "checkout-service",
      "level": "ERROR",
      "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
      "span_id": "00f067aa0ba902b7",
      "raw_message": "NullPointerException: PaymentGatewayClient.process() returned null",
      "attributes": {
        "container_id": "docker://9912384a",
        "pod_name": "checkout-service-7f6d8-99zxc"
      }
    },
    ...
  ]
}

Response: 202 Accepted
{
  "status": "QUEUED",
  "ingested_count": 250,
  "rejected_count": 0
}
```

---

### 3.3 Full-Text Search Logs (Dashboard Query Interface)
```http
POST /api/v1/logs/search
Authorization: Bearer <user_token>
Content-Type: application/json

Request Body:
{
  "query_string": "service_name:checkout-service AND level:ERROR AND \"NullPointerException\"",
  "time_range": {
    "start_time": "2026-10-18T12:00:00Z",
    "end_time": "2026-10-18T14:00:00Z"
  },
  "filters": {
    "environment": "PRODUCTION"
  },
  "pagination": {
    "limit": 50,
    "offset": 0
  },
  "sort_by": "timestamp",
  "sort_order": "DESC"
}

Response: 200 OK
{
  "total_hits": 1420,
  "took_ms": 42,
  "storage_tier_hit": "HOT_ELASTICSEARCH",
  "results": [
    {
      "event_id": "evt_991823-112",
      "timestamp": "2026-10-18T13:58:12.441Z",
      "service_name": "checkout-service",
      "level": "ERROR",
      "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
      "message": "NullPointerException: PaymentGatewayClient.process() returned null",
      "hostname": "worker-node-104"
    },
    ...
  ]
}
```

---

### 3.4 Live Tail Streaming (WebSocket Endpoint)
```http
GET /ws/v1/logs/tail?service=checkout-service&level=ERROR
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer <user_token>
```
Server streams matching log JSON payloads directly to the browser console with sub-second latency.

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/591859853_25724875320429770_5245894370665601546_n.jpg?updatedAt=1764409028874)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 16](https://www.interviewwithbunny.com/systemdesign/16))*

### Component Interaction Topology
```
[ Application Pods / Nodes ]
     │ (log4j stdout/stderr)
     ▼
[ Host Container Node File: /var/log/pods/*.log ]
     │
     ▼
[ Local Agent: Fluent Bit / Vector DaemonSet ]
     │
     ▼ (Compressed HTTPS POST)
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
│         (SSL Termination, Token Auth, Rate Limiting)        │
└─────────────────────────────────────────────────────────────┘
          │                                           │
  (Stream Ingestion)                         (Offline Batch Files)
          │                                           │
          ▼                                           ▼
┌────────────────────────┐                 ┌────────────────────────┐
│  Stream Ingest Service │                 │  File Ingest Service   │
│  (Stateless Receivers) │                 │  (Multipart Upload)    │
└────────────────────────┘                 └────────────────────────┘
          │                                           │
          └──────────────────┬────────────────────────┘
                             ▼
               ┌───────────────────────────┐
               │    Kafka Message Broker   │
               │   Topic: 'logs.raw.stream'│
               └───────────────────────────┘
                             │
                             ▼
               ┌───────────────────────────┐
               │   Stream Processor Engine │
               │      (Apache Flink)       │
               │  - Validation & Parsing   │
               │  - Enrichment & Masking   │
               │  - Anomaly Detection      │
               └───────────────────────────┘
                │            │            │            │
   (Error Alert)│            │            │            │ (Cold Archival)
                ▼            ▼            ▼            ▼
     ┌─────────────┐  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
     │ Kafka Topic │  │ Hot Storage │ │Warm Storage │ │Cold Storage │
     │ 'logs.alert'│  │(Elastic-    │ │ (Cassandra  │ │  (AWS S3    │
     └─────────────┘  │  Search)    │ │  Cluster)   │ │  Parquet)   │
            │         │  14 Days    │ │  45 Days    │ │  1+ Years   │
            ▼         └─────────────┘ └─────────────┘ └─────────────┘
     ┌─────────────┐         │               │               │
     │ Alerting &  │         └───────────────┼───────────────┘
     │ Notif Svc   │                         ▼
     └─────────────┘               ┌───────────────────┐
            │                      │ Search & Analytics│
            ▼                      │   Service API     │
     [ PagerDuty /                 └───────────────────┘
       Slack / SMS ]                         │
                                             ▼
                                   [ Web UI Dashboard ]
```

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Beneath the Microservice — How Logs Flow from Application to Agent

Bunny highlights a common interview misconception: developers often assume microservices invoke a REST endpoint directly to submit logs.
In reality, **direct HTTP logging from application threads is a severe anti-pattern** because any network hiccup or logging server downtime blocks business transactions.

```
+---------------------------------------------------------------------------------+
|                               Kubernetes Worker Node                            |
|                                                                                 |
|  [ User App Container ]                                                         |
|         │                                                                       |
|         │ log.info() / log.error()                                              |
|         ▼                                                                       |
|   (stdout / stderr)                                                             |
|         │                                                                       |
|         ▼                                                                       |
|  [ Container Engine: Containerd / Docker ]                                      |
|         │                                                                       |
|         │ Writes structured JSON line                                           |
|         ▼                                                                       |
|  [ Node Disk Storage: /var/log/pods/checkout-service_uuid/app.log ]              |
|         │                                                                       |
|         │ Local non-blocking disk tail                                          |
|         ▼                                                                       |
|  [ Fluent Bit DaemonSet Agent ]                                                 |
|     ├─ In-Memory & Local SQLite Buffer Ring (survives network disconnects)      |
|     ├─ Micro-Batch Aggregator (500ms or 256KB chunks)                           |
|     └─ Gzip Compression Engine                                                  |
|         │                                                                       |
+---------┼-----------------------------------------------------------------------+
          │ HTTPS POST (Non-blocking asynchronous flush)
          ▼
   [ Central Ingestion Gateway ]
```

1. **Non-Blocking Execution**: Applications write exclusively to `stdout`/`stderr`.
2. **Container Engine Capture**: Kubernetes routes streams to local node log files on SSDs.
3. **DaemonSet Agent (Fluent Bit)**: Exactly one Fluent Bit pod runs per physical node. It tails log files, applies local buffer rings on disk if downstream networks fail, and dispatches batched chunks upstream.

---

### 5.2 Deep Dive 2: Stream Normalization & Processing Pipeline (Apache Flink)

Raw logs arriving from thousands of independent teams exhibit vast discrepancies in formatting.
Instead of building fragile, independently maintained microservices, Bunny advises deploying **Apache Flink** as the centralized stream transformation engine:

```
[ Ingested Raw Log ]
  --> "2026-10-18 14:00:00,120 ERROR [checkout-svc] [tx_991823] User 4412 payment card 4111-2222-3333-4444 failed"
                                      │
                                      ▼
                        ┌───────────────────────────┐
                        │ Apache Flink Stream Job   │
                        └───────────────────────────┘
                                      │
    ┌─────────────────────────────────┼─────────────────────────────────┐
    ▼                                 ▼                                 ▼
[ 1. Parsing & Extraction ]    [ 2. PII Sanitization ]        [ 3. Metadata Enrichment ]
Grok: Timestamp, Level,        Mask regex pattern:            Add host_ip, region,
Service, Message               "4111-****-****-4444"          cluster_id, tenant_id
                                      │
                                      ▼
[ Standardized Normalized JSON Schema ]
{
  "timestamp": 1792341600120,
  "level": "ERROR",
  "service": "checkout-svc",
  "trace_id": "tx_991823",
  "message": "User 4412 payment card 4111-****-****-4444 failed",
  "cluster": "us-east-1",
  "env": "prod"
}
```

#### Flink Processing Steps:
1. **Schema Validation & Grok Parsing**: Unpacks unstructured log lines into structured key-value maps.
2. **PII Masking**: Regular expressions automatically detect and redact credit card numbers, passwords, and Social Security Numbers.
3. **Deduplication**: Identifies identical rapid-fire stack traces within a 1-second tumbling window and merges them with a `repeat_count` attribute to protect storage tiers.
4. **Multiplexed Routing**: Evaluates log severity and simultaneously dispatches copies to Hot Storage, Warm Storage, Cold Storage, and the Alerting Kafka Topic.

---

### 5.3 Deep Dive 3: The Multi-Tier Storage & Query Lifecycle

A major architectural insight from Bunny is that indexing every single log line in Elasticsearch forever is a financial and operational catastrophe: Elasticsearch inverted indexes consume massive RAM and disk overhead ($\approx 1.3\times$ raw size).

#### The 3-Tier Lifecycle Breakdown

```
       [ Stream Processor (Flink) ]
         │             │             │
(Day 0)  │             │             │
         ▼             ▼             ▼
   ┌───────────┐ ┌───────────┐ ┌───────────┐
   │ Hot Tier  │ │ Warm Tier │ │ Cold Tier │
   │ Elastic-  │ │ Cassandra │ │ Amazon S3 │
   │ search    │ │ Cluster   │ │ (Snappy   │
   │ (14 Days) │ │ (45 Days) │ │  Parquet) │
   └───────────┘ └───────────┘ └───────────┘
         │             │             │
         ▼             │             │
   (TTL Deletion       │             │
    at Day 14)         ▼             │
                 (TTL Deletion       │
                  at Day 45)         ▼
                               (Immutable 
                                1-5 Year Archival)
```

#### 1. Hot Tier: Elasticsearch (0 to 14 Days)
- **Role**: Powers developer dashboards and exploratory ad-hoc full-text searches.
- **Index Management**: Daily rotating indices (`logs-prod-2026-10-18`).
- **Lifecycle Action**: At Day 14, an automated Index Lifecycle Management (ILM) job deletes indices, freeing RAM and disk.

#### 2. Warm Tier: Cassandra (0 to 45 Days)
- **Role**: Serves point-lookup queries (e.g., "Fetch all logs for `trace_id = tx_991823`" or "Fetch ERROR logs for `checkout-service` between 10:00 and 10:30").
- **LSM-Tree Storage Engine**: Handles heavy write loads with minimal IOPS contention.
- **Cassandra Schema Design**:
  ```sql
  CREATE KEYSPACE distributed_logging 
  WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east': 3, 'ap-south': 3};

  -- Partitioned by service and date bucket to prevent oversized wide rows
  CREATE TABLE distributed_logging.logs_by_service (
      org_id uuid,
      service_name text,
      date_bucket text,             -- e.g., '2026-10-18'
      level text,                   -- 'ERROR', 'WARN', 'INFO'
      event_timestamp timestamp,
      trace_id text,
      event_id uuid,
      message text,
      metadata map<text, text>,
      PRIMARY KEY ((org_id, service_name, date_bucket), level, event_timestamp, event_id)
  ) WITH CLUSTERING ORDER BY (level ASC, event_timestamp DESC, event_id ASC)
    AND default_time_to_live = 3888000; -- 45 Days Native TTL
  ```

#### 3. Cold Tier: Amazon S3 / GCS (45 Days to 5 Years)
- **Role**: Long-term regulatory compliance, disaster recovery, and offline analytics.
- **Format**: Compressed **Apache Parquet (Snappy)** with columnar compression ($80\%$ space reduction).
- **Cost**: Less than $\$0.023$ per GB/month (over $90\%$ cheaper than Cassandra/ES). Can be queried on demand using **AWS Athena / Trino** serverless engines.

---

### 5.4 Deep Dive 4: Search Tier Query Routing Logic

When an engineer issues a query via the Dashboard, the Search Service intelligently routes between tiers:

```python
def execute_log_search(query_request):
    current_time = now()
    query_start = query_request.time_range.start_time
    age_in_days = (current_time - query_start).days

    # Route 1: Queries within the last 14 days hit Hot Tier
    if age_in_days <= 14:
        return query_elasticsearch(query_request)

    # Route 2: Structured queries between 14 and 45 days hit Cassandra
    elif age_in_days <= 45 and query_request.has_service_and_trace_id():
        return query_cassandra_warm_store(query_request)

    # Route 3: Older historical or non-indexed searches hit Athena/Parquet
    else:
        return submit_athena_cold_query(query_request)
```

---

### 5.5 Deep Dive 5: Real-Time Anomaly Alerting Engine

When critical errors spike, waiting for scheduled batch jobs risks prolonged outages.
1. **Detection in Flink**: During stream processing, Flink filters for `level IN ('ERROR', 'FATAL')` and evaluates sliding window rules.
2. **Rule Evaluation**:
   - `Rule 1`: Count of `ERROR` on `checkout-service` $>$ 50 in 1 minute.
   - `Rule 2`: Occurrence of string `OutOfMemoryError` or `DatabaseConnectionTimeout`.
3. **Dispatch**: Violations push an event to the Kafka `logs.alert` topic. An **Alerting Consumer** claims the event, deduplicates alerts (preventing alert fatigue), and triggers PagerDuty / Slack notifications via an external webhook within 3 seconds of error emission.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: Why use Kafka as a buffer instead of writing directly to the database?

Kafka provides critical decoupling and resilience:

- **Point 1**: Back-pressure handling - if DB is slow/down, Kafka buffers millions of logs without data loss,

- **Point 2**: Replay capability - can reprocess logs if processing logic changes or fails,

- **Point 3**: Multiple consumers - same log stream feeds Elasticsearch, Cassandra, and Alert service independently,

- **Peak load smoothing - ingestion spikes (10x normal) absorbed by Kafka, downstream processes at steady rate. Example**:
  Black Friday traffic spike - Kafka handles 1M logs/sec burst, Flink processes at 100K/sec, no data loss.

---

### Q2: How do you handle out-of-order logs with different timestamps?

Multi-layer approach:

- **Point 1**: Flink Watermarks - uses event time processing with 5-minute watermark delay, allows late-arriving logs to be included in correct time window,

- **Point 2**: Cassandra Clustering - sorts by timestamp within partition, maintains chronological order for queries,

- **Point 3**: Elasticsearch @timestamp - indexed for range queries, UI shows logs in order even if processed out-of-order,

- **Grace Period - Flink accepts logs up to 10 minutes late, drops extremely late logs to DLQ. Trade-off**:
  5-min watermark delays alerting but ensures 99.9% logs in correct window.

---

### Q3: What happens if Elasticsearch goes down? Can users still search logs?

Graceful degradation strategy:

- **Point 1**: Fallback to Cassandra - Search service automatically routes queries to Cassandra, slower (1-2s vs 200ms) but functional,

- **Point 2**: Elasticsearch recovery - Flink queues bulk index requests in Kafka topic 'es_retry', replays when ES recovers,

- **Point 3**: Cache layer - Redis caches recent common queries for 60s, serves during ES downtime,

- **Status notification - UI shows 'Limited search mode' banner, sets user expectations. Cassandra can handle point queries (by service+time) but lacks full-text search capability. Typical ES downtime**:
  <5 minutes with auto-restart.

---

### Q4: How do you ensure no log data is lost during ingestion?

Multiple durability guarantees:

- **Point 1**: Kafka persistence - replication factor=3, min.insync.replicas=2, acks=all ensures log written to 2+ brokers before acknowledging,

- **Point 2**: Agent buffering - FluentBit buffers 10MB locally on disk, retries failed sends with exponential backoff (max 24 hours),

- **Point 3**: Flink checkpointing - every 60s, saves processing state to durable storage (S3), can recover exactly-once on failure,

- **Point 4**: Cassandra replication - RF=3 with QUORUM writes, tolerates 1 node failure,

- **S3 archival - immutable cold storage with 11 9's durability. End-to-end guarantee**:
  At-least-once delivery (may have duplicates, handled by deduplication), zero data loss.

---

### Q5: How does the deduplication mechanism work in Flink?

Window-based deduplication:

- **Point 1**: Tumbling Window - 5-minute windows, groups logs by time slice,

- **Hash Generation - creates hash from (timestamp_minute, service_name, message_substring[0**:
  100]),

- **Point 3**: State Backend - Flink maintains RocksDB state with hash set of seen hashes within window,

- **Point 4**: Duplicate Detection - if hash exists in state, drop log, else add to state and emit downstream,

- **State Expiration - window closes after 5 min, state cleared to prevent unbounded growth. Handles**:
  duplicate sends from agent retries, load balancer routing same log to multiple endpoints. Trade-off: only deduplicates within 5-min window, cross-window duplicates not detected.

---

### Q6: What's your data retention and archival strategy?

Three-tier lifecycle:

- **Point 1**: Hot (0-14 days) - Cassandra + Elasticsearch, SSD storage, <200ms queries, costs $0.10/GB/month,

- **Point 2**: Warm (14-45 days) - Cassandra only (ES deleted), HDD storage, 1-2s queries, costs $0.05/GB/month,

- **Cold (>45 days) - S3 Parquet files, Glacier storage, 10-30s queries via Athena, costs $0.01/GB/month. Cron job runs daily**:
  moves ES indices to warm tier (ILM policy), exports Cassandra to S3, deletes old Cassandra data. Example: 1TB logs/day = $3000/month hot storage vs $300/month blended with tiering. Compression: Snappy compression 3:1, Parquet columnar format enables efficient queries on cold data.

---

### Q7: How do you handle log search across multiple time ranges efficiently?

Query routing strategy:

- **Point 1**: Time-based routing - if query range within last 14 days, query Elasticsearch (fast), if 14-45 days, query Cassandra (medium), if >45 days, query S3 via Athena (slow),

- **Point 2**: Parallel execution - for cross-tier queries (e.g., last 60 days), fan out to ES + Cassandra + S3 in parallel, merge results,

- **Point 3**: Elasticsearch query optimization - use date histogram aggregations for time-series, filter context instead of query context for non-scoring filters,

- **Point 4**: Cassandra partition pruning - WHERE clause includes partition key (service_name, log_date) to read minimal partitions,

- **S3 partitioning - Parquet files partitioned by year/month/day, Athena only scans relevant partitions. Example**:
  Query 'last 30 days' = ES (14 days) + Cassandra (16 days) in <500ms vs single tier would be 2-3s.

---

### Q8: How do you implement real-time log tailing efficiently?

WebSocket + Kafka Consumer approach:

- **Point 1**: Connection - client opens WebSocket to /logs/tail with filters (service, level),

- **Point 2**: Consumer Group - backend creates dedicated Kafka consumer in unique group (per WebSocket connection), subscribes to 'alert_topic',

- **Point 3**: Filtering - Kafka consumer uses Consumer Interceptor to filter messages matching client's criteria before sending over WebSocket,

- **Point 4**: Backpressure - if client slow to consume, buffer last 100 messages in-memory, drop oldest if buffer full,

- **Point 5**: Heartbeat - ping/pong every 30s to detect disconnects,

- **Reconnect - on disconnect, client stores last received log timestamp, reconnects with 'since' parameter to resume. Scalability**:
  Kafka supports 10K+ consumer groups, each WebSocket is independent, no shared state. Alternative: SSE (Server-Sent Events) for one-way streaming, simpler but no bidirectional communication.

---

### Q9: What's your alerting strategy to avoid alert fatigue?

Multi-level alert suppression:

- **Point 1**: Aggregation - alert on 'ERROR count >100 in 5 min' not 'every ERROR', reduces noise by 100x,

- **Point 2**: Cooldown Period - once alert fires, don't re-alert for same condition for 10 minutes (configurable),

- **Point 3**: Severity Levels - P0 (critical, page immediately), P1 (high, email+Slack), P2 (medium, Slack only), P3 (low, daily digest),

- **Point 4**: Deduplication - alerts with same fingerprint (service+error_type) grouped, send summary not individual alerts,

- **Point 5**: Escalation - if P1 alert not acknowledged in 15 min, escalate to P0 and page on-call,

- **Rate Limiting - max 10 alerts per service per hour, prevent alert storms. Example**:
  Payment service 1000 errors → single alert 'Payment ERROR spike: 1000 in 5 min' with link to dashboard, not 1000 individual alerts.

---

### Q10: How do you handle schema evolution when log formats change?

Backward-compatible schema design:

- **Point 1**: Schema Registry - maintain version history in registry, agents send schema_version with logs,

- **Point 2**: Multi-version Support - Flink has parsers for v1, v2, v3 schemas, routes to appropriate parser based on version,

- **Point 3**: Elasticsearch Mapping - dynamic mapping enabled, new fields auto-detected and indexed with default types,

- **Point 4**: Default Values - missing fields populated with defaults during processing (null for optional, 'UNKNOWN' for required),

- **Point 5**: Migration Path - during schema change, run both old and new parsers in parallel for 7 days, validate outputs match before deprecating old version,

- **Deprecated Field Handling - old fields kept for 90 days with 'deprecated' flag, gives clients time to upgrade agents. Example**:
  Add 'request_id' field → Flink auto-maps to ES, old logs without field show null, queries work across old+new logs.

---

## 7. Architectural Trade-Off Analysis

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Log Transmission** | Local DaemonSet Agent (Fluent Bit) | App Direct HTTP API Calls | Direct calls block application worker threads during network delays; DaemonSets isolate logging from core business logic. |
| **Stream Transformation** | Apache Flink Stateful Processor | Microservice Consumer Pods | Flink provides native distributed state, out-of-order event watermarking, and sliding window aggregations out of the box. |
| **Hot Storage Limit** | 14-Day Strict TTL in Elasticsearch | Infinite Retention in ES | Drastically curtails RAM/SSD infrastructure costs; older logs are safely retained in Cassandra and low-cost S3 Parquet. |
| **Warm Storage Engine** | Apache Cassandra (SSTables) | Relational DB (PostgreSQL) | Cassandra's LSM-Tree log-structured storage handles massive sequential write bursts without row-level locking bottlenecks. |
| **Query Routing** | Tiered Query Router (ES $\rightarrow$ Cassandra $\rightarrow$ Athena) | Single Unified Storage | Sacrifices uniform sub-second search across multi-year data to achieve orders-of-magnitude lower infrastructure cost. |

---

## 8. Failure Modes & Resiliency Strategies

1. **Host Agent Outage & Network Partition**:
   - Fluent Bit agents buffer up to 500 MB locally in SQLite ring files on host SSDs.
   - When the connection to the Central Ingestion Gateway is restored, the agent drains the backlog with rate limiting to avoid thundering-herd saturation.
2. **Kafka Ingestion Lag Spike**:
   - If log volume quadruples during a site-wide outage, Kafka partition lag increases.
   - Flink workers autoscale dynamically via Kubernetes HPA based on Kafka Consumer Group Lag metrics.
3. **Elasticsearch Cluster Red Status (Shard Unavailability)**:
   - Flink catches write exceptions to Elasticsearch without terminating the pipeline.
   - It redirects writes to Cassandra and S3, ensuring zero data loss while Elasticsearch index recovery takes place.
