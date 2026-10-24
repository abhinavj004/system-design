# System Design 15: Design Scalable Notification System (SMS, Email, Push, OTP)

> **Source**: [Interview with Bunny - Episode 15](https://www.youtube.com/watch?v=Yv9q8d11Y7k)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/15](https://www.interviewwithbunny.com/systemdesign/15)  
> **Topic**: Designing a Highly Scalable, Multi-Tenant Distributed Notification System (Twilio, SendGrid, APNS, FCM)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Problem Scope & Tenancy Model
A modern notification platform serves as the centralized communications backbone for multiple enterprise organizations (**Clients** like Amazon, Uber, Flipkart, Netflix) to send communications to their downstream customers (**End Users**).

The system must handle heterogeneous delivery channels (SMS, Email, In-App Push, WebSockets), enforce user-level notification preferences, ensure strict delivery guarantees for critical messages (such as Banking One-Time Passwords - OTPs), and handle bulk promotional broadcasts without choking the real-time pipeline.

---

### 1.2 Functional Requirements (FR)
1. **Multi-Channel Dispatch**: Support dispatch across three distinct communication channels:
   - **SMS**: Transactional OTPs, order updates (via Twilio, MessageBird, MSG91).
   - **Email**: Rich transactional receipts, marketing campaigns (via SendGrid, AWS SES, Mailgun).
   - **In-App Mobile Push**: Device notifications for iOS (APNs) and Android (Firebase Cloud Messaging - FCM).
2. **Dual Scheduling Modes**:
   - **Real-Time / Immediate**: Time-critical alerts dispatched in sub-seconds (e.g., Auth OTPs, fraud alerts).
   - **Scheduled / Deferred**: Batch promotional notifications scheduled for a specific timestamp (e.g., festive sale at 08:00 AM).
3. **Dynamic Template Management**: Enterprise clients can configure and version message templates with dynamic placeholders (e.g., `Hello {{user_name}}, your order #{{order_id}} has shipped`). Channel-specific formats must be enforced (concise plain-text for SMS vs. rich HTML/CSS for Email).
4. **End-User Preference Engine**: End users can configure granular communication preferences (e.g., `Email: ENABLED`, `SMS: DISABLED`, `Push: ENABLED`) and quiet hour restrictions. The system must filter dispatches against these preferences prior to vendor invocation.
5. **End-to-End Lifecycle & Delivery Tracking**:
   - Track lifecycle states: `PENDING` $\rightarrow$ `QUEUED` $\rightarrow$ `SENT_TO_VENDOR` $\rightarrow$ `DELIVERED` (or `FAILED` / `BOUNCED` / `READ`).
   - Ingest asynchronous delivery receipts from external vendor webhooks.
6. **Client Reporting & Analytics**: Enterprise clients can query aggregate delivery rates, open rates, vendor latency metrics, and detailed audit trails.

---

### 1.3 Non-Functional Requirements (NFR)
1. **High Ingestion Throughput**: Handle an industry-scale load of **1,000,000 notifications per minute** ($\approx 16,667 \text{ notifications/sec}$, with peak bursts up to $50,000 \text{ notifications/sec}$).
2. **High Availability (AP under CAP Theorem)**:
   - Communication ingestion must never reject upstream API calls.
   - Eventual consistency is completely acceptable for user preference updates and template edits (a 2–5 second propagation delay is harmless).
3. **Tiered Latency SLAs**:
   - **Critical Tier (OTP / Financial Security Alerts)**: Delivery SLA $< 2\text{ seconds}$ end-to-end.
   - **Standard & Promotional Tier**: Delivery SLA within $5 - 10\text{ seconds}$ (or deferred batch processing).
4. **Zero Message Loss (Durability Guarantee)**:
   - Critical transactional messages must never be lost due to server crashes or transient message broker reboots.
   - Guaranteed using the **Transactional Outbox Pattern** combined with **Change Data Capture (CDC)**.
5. **Vendor Rate-Limit Protection (Egress Throttling)**: Prevent downstream vendor throttling (HTTP 429 Too Many Requests) by enforcing token bucket rate limiters per downstream telecom/email partner.
6. **Strict Idempotency**: Network retries from upstream clients must not result in duplicate SMS sends (e.g., charging a user twice or sending twin OTPs).

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput
- **Average Traffic**:
  $$\text{Average QPS} = \frac{1,000,000 \text{ notifications}}{60 \text{ seconds}} \approx 16,667 \text{ req/sec}$$
- **Peak Burst Traffic**: $3\times \text{Average} \approx 50,000 \text{ req/sec}$.
- **Channel Distribution**:
  - Push Notifications: $60\% \implies 10,000 \text{ QPS}$
  - SMS & OTP: $20\% \implies 3,334 \text{ QPS}$
  - Email: $20\% \implies 3,334 \text{ QPS}$

#### Storage Estimations (5-Year Retention)
- `NotificationRecord` size:
  - `notification_id`: 16 bytes (UUID)
  - `client_id`: 16 bytes
  - `user_id`: 16 bytes
  - `template_id`: 8 bytes
  - `channel`: 4 bytes
  - `status`: 4 bytes
  - `payload_json` (variables): 250 bytes
  - `timestamps` (created, updated, delivered): 24 bytes
  - Total per record $\approx 350 \text{ bytes}$.
- **Daily Storage**:
  $$\text{Daily Volume} = 1,000,000 \times 60 \times 24 \approx 1.44 \text{ Billion notifications/day}$$
  $$\text{Storage/day} = 1.44 \times 10^9 \times 350 \text{ bytes} \approx 504 \text{ GB / day}$$
- Hot relational operational data is retained for 30 days ($\approx 15 \text{ TB}$), while cold historical event logs are offloaded to **Google BigQuery / Snowflake / S3 Parquet Data Lake** for multi-year compliance and reporting.

#### Caching Memory (User Preferences)
- 100 Million active users $\times$ 64 bytes (preference bitmask + client ID) $\approx 6.4 \text{ GB RAM}$.
- Fits entirely within a standard 16 GB Redis cluster.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|       Client       |        |NotificationTemplate|        |   UserPreference   |
+--------------------+        +--------------------+        +--------------------+
| id (PK, UUID)      | 1    * | id (PK, VARCHAR)   |        | id (PK, UUID)      |
| name (VARCHAR)     |<------>| client_id (FK)     |        | client_id (FK)     |
| api_key (VARCHAR)  |        | name (VARCHAR)     |        | external_user_id   |
| tier (ENTERPRISE)  |        | channel (SMS/EMAIL)|        | email_enabled(BOOL)|
| created_at (TIME)  |        | content_body (TEXT)|        | sms_enabled (BOOL) |
+--------------------+        | variables (JSON)   |        | push_enabled (BOOL)|
         │ 1                  | version (INT)      |        | quiet_hours (JSON) |
         │                    | is_active (BOOL)   |        +--------------------+
         ▼ *                  +--------------------+
+--------------------+        +--------------------+        +--------------------+
| NotificationRecord |        |   OutboxRecord     |        | NotificationEvent  |
+--------------------+        +--------------------+        +--------------------+
| id (PK, UUID)      | 1    1 | id (PK, BIGSERIAL) | 1    * | event_id (PK, UUID)|
| client_id (FK)     |<------>| notification_id(FK)|<------>| notification_id(FK)|
| external_user_id   |        | event_type (VAR)   |        | status (ENUM)      |
| template_id (FK)   |        | payload (JSONB)    |        | vendor (TWILIO)    |
| channel (ENUM)     |        | is_published(BOOL) |        | vendor_msg_id (VAR)|
| priority (HIGH/LOW)|        | created_at (TIME)  |        | event_time (TIME)  |
| status (ENUM)      |        +--------------------+        | error_code (VAR)   |
| created_at (TIME)  |                                      +--------------------+
+--------------------+
```

### Distinction: Client vs. User
- **Client (Tenant)**: The business enterprise invoking our platform (e.g., `Uber`, `Amazon`). Clients own API keys, templates, and billing tiers.
- **User (Recipient)**: The end customer receiving the message. Users possess device tokens (FCM/APNs), phone numbers, email addresses, and channel opt-out preferences.

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Create / Update Message Template
```http
POST /api/v1/templates
Authorization: Bearer <client_api_key>
Content-Type: application/json

Request Body:
{
  "name": "ORDER_SHIPPED_ALERT",
  "channel": "SMS",                   // "SMS" | "EMAIL" | "PUSH"
  "type": "TRANSACTIONAL",            // "TRANSACTIONAL" | "PROMOTIONAL"
  "content_body": "Hi {{user_name}}, your order #{{order_id}} has shipped via {{carrier}}! Track: {{tracking_url}}",
  "variables": ["user_name", "order_id", "carrier", "tracking_url"],
  "version": 1
}

Response: 201 Created
{
  "template_id": "tpl_ord_ship_v1",
  "client_id": "cli_amazon_us",
  "version": 1,
  "created_at": "2026-10-18T10:00:00Z"
}
```

---

### 3.2 Trigger Notification Dispatch
```http
POST /api/v1/notifications
Authorization: Bearer <client_api_key>
X-Idempotency-Key: idemp_98234-abcd-9912
Content-Type: application/json

Request Body:
{
  "template_id": "tpl_ord_ship_v1",
  "recipient": {
    "external_user_id": "usr_9918237",
    "phone": "+14155552671",
    "email": "alex@example.com"
  },
  "variables": {
    "user_name": "Alex",
    "order_id": "AMZ-99120-X",
    "carrier": "FedEx",
    "tracking_url": "https://trk.amz.com/99120"
  },
  "channel": "SMS",
  "priority": "HIGH",                 // "HIGH" (OTP) | "STANDARD" | "LOW" (Marketing)
  "scheduled_at": null                // null for immediate dispatch
}

Response: 202 Accepted
{
  "notification_id": "notif_4491-a1b2-c3d4",
  "status": "QUEUED",
  "priority": "HIGH",
  "created_at": "2026-10-18T10:05:00Z"
}
```
> **Bunny's Key Insight (HTTP 202 Accepted)**: Under high concurrency ($50\text{K QPS}$), a notification request must **never block** on downstream telecommunication handshakes. The API validates credentials and schema, queues the message, and returns `202 Accepted` within $< 15\text{ms}$.

---

### 3.3 Set End-User Preferences
```http
PUT /api/v1/users/{external_user_id}/preferences
Authorization: Bearer <client_api_key>
Content-Type: application/json

Request Body:
{
  "channels": {
    "email": true,
    "sms": false,
    "push": true
  },
  "categories": {
    "transactional": true,           // Critical OTP/receipts cannot be disabled
    "promotional": false
  },
  "quiet_hours": {
    "enabled": true,
    "start_utc": "22:00",
    "end_utc": "07:00"
  }
}

Response: 200 OK
{
  "status": "UPDATED",
  "external_user_id": "usr_9918237",
  "updated_at": "2026-10-18T10:06:30Z"
}
```

---

### 3.4 Inbound Vendor Delivery Webhook
```http
POST /api/v1/webhooks/providers/twilio
Authorization: Signature <hmac_sha256_signature>
Content-Type: application/json

Request Body:
{
  "MessageSid": "SM_twilio_9981240192",
  "MessageStatus": "delivered",        // "delivered" | "failed" | "undelivered"
  "To": "+14155552671",
  "ErrorCode": null,
  "Timestamp": "2026-10-18T10:05:04Z"
}

Response: 200 OK
```

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/efbwwegwgrkgnelgnlgnelgreglg.png?updatedAt=1760949557207)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 15](https://www.interviewwithbunny.com/systemdesign/15))*

### Component Interaction Topology
```
[ Enterprise Clients: Amazon, Uber, Flipkart ]
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
│         (Auth, SSL Termination, Ingress Rate Limiting)      │
└─────────────────────────────────────────────────────────────┘
          │                                           │
  (Manage Templates & Preferences)           (Trigger Notifications)
          │                                           │
          ▼                                           ▼
┌────────────────────────┐                 ┌────────────────────────┐
│  Metadata Management   │                 │  Notification Ingest   │
│  (Template & Pref Svc) │                 │  Service (Stateless)   │
└────────────────────────┘                 └────────────────────────┘
          │          │                                │          │
          │          ▼                                │          │
          │   ┌─────────────┐       (Critical Path:   │          │ (Standard / Promo Path:
          │   │ Redis Cache │        OTP Direct Jump) │          │  Transactional Outbox)
          │   │ (User Prefs)│                         ▼          ▼
          │   └─────────────┘                 ┌──────────┐ ┌───────────────────┐
          │                                   │          │ │ PostgreSQL DB     │
          ▼                                   │          │ │ - notif_records   │
┌────────────────────────┐                    │          │ │ - outbox_table    │
│ PostgreSQL Relational  │                    │          │ └───────────────────┘
│ (Templates & Metadata) │                    │          │           │
└────────────────────────┘                    │  Kafka   │           ▼
                                              │  Broker  │ ┌───────────────────┐
                                              │  Cluster │ │ Debezium CDC Log  │
                                              │          │ │ Reader Engine     │
                                              │          │ └───────────────────┘
                                              │          │           │
                                              └──────────┘<──────────┘
                                                    │
                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Provider Consumer Dispatchers                        │
│    ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────────┐    │
│    │  SMS Dispatcher  │  │ Email Dispatcher │  │ In-App Push Dispatch  │    │
│    │ (Twilio / MSG91) │  │(SendGrid/AWS SES)│  │ (Apple APNs / FCM)    │    │
│    └──────────────────┘  └──────────────────┘  └───────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                      │                                     │
                      ▼                                     ▼
           [ External Telcos / APNs ]            [ Webhook Ingestion Engine ]
                      │                                     │
                      ▼                                     ▼
             [ End-User Devices ]               ┌───────────────────────┐
                                                │ Delivery Event Kafka  │
                                                └───────────────────────┘
                                                    │               │
                                                    ▼               ▼
                                         ┌─────────────────┐ ┌─────────────────┐
                                         │ PostgreSQL DB   │ │ BigQuery / OLAP │
                                         │ (Update Status) │ │ (Analytics Log) │
                                         └─────────────────┘ └─────────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: The Critical Dilemma — Direct Kafka vs. Transactional Outbox Pattern

Bunny explains a fundamental architectural flaw present in naive notification systems:
- **The Naive Flow**: The ingestion service accepts a message, publishes it directly to Kafka, returns `202 Accepted` to the client, and relies on background workers to insert the record into the database.
- **The Failure Point**: If the service crashes after acknowledging the client but before Kafka acknowledges the message, or if a Kafka broker experiences split-brain/leader failover, **the notification is permanently lost without trace**. For enterprise clients like Amazon, claiming an order confirmation SMS was sent when it never left the system violates regulatory SLAs.

#### The Dual-Path Architectural Solution
To reconcile **extreme durability** with **sub-second latency**, the system splits into two distinct operational pipelines based on priority:

```
                            [ Inbound Notification Request ]
                                           │
                                           ▼
                                 { Is Priority == HIGH? }
                                  /                    \
                            YES  /                      \  NO (Standard / Promotional)
                                ▼                        ▼
                  ┌──────────────────────┐     ┌──────────────────────┐
                  │ Direct Fast-Path     │     │ Transactional Outbox │
                  │ Push to Kafka Topic: │     │ Atomic DB Commit:    │
                  │ 'critical.sms.otp'   │     │ (notif + outbox)     │
                  └──────────────────────┘     └──────────────────────┘
                               │                         │
                               │                         ▼
                               │               ┌──────────────────────┐
                               │               │ Debezium CDC Engine  │
                               │               │ Stream to Kafka      │
                               │               └──────────────────────┘
                               │                         │
                               ▼                         ▼
                  ┌───────────────────────────────────────────────────┐
                  │              Kafka Partition Pipeline             │
                  └───────────────────────────────────────────────────┘
```

#### 1. The Standard / Promotional Path (Transactional Outbox + Debezium CDC)
- The Notification Service writes to both `notification_records` and `outbox_table` inside a **single ACID transaction**:
  ```sql
  BEGIN;
  INSERT INTO notification_records (id, client_id, external_user_id, template_id, channel, status)
  VALUES ('notif_9981', 'cli_amz', 'usr_4412', 'tpl_ship', 'SMS', 'PENDING');

  INSERT INTO outbox_table (notification_id, event_type, payload)
  VALUES ('notif_9981', 'NOTIFICATION_CREATED', '{"phone": "+14155552671", "msg": "Order Shipped"}');
  COMMIT;
  ```
- **Debezium CDC (Change Data Capture)** monitors PostgreSQL's Write-Ahead Log (WAL) and streams committed outbox rows directly into Kafka.
- **Guarantees**: 100% durability, zero dual-write anomalies, and guaranteed at-least-once publishing.

#### 2. The Critical Path (Direct Fast-Path for OTPs)
- For Banking OTPs, waiting for disk sync, DB commit, and WAL CDC introduces a $200\text{ms} - 500\text{ms}$ delay.
- The service pushes OTPs **directly to the dedicated `critical.sms.otp` Kafka topic**.
- **Rationale for Durability Trade-off**: If an OTP is lost in an unprecedented catastrophic Kafka crash, end users naturally hit "Resend OTP" within 20 seconds. Low latency ($< 1\text{s}$) overrides strict DB-first durability for ephemeral codes with 60-second TTLs.

---

### 5.2 Deep Dive 2: Kafka Multi-Topic Partitioning & Priority Matrix

Bunny outlines a 9-topic matrix partitioning scheme based on **Channel** $\times$ **Severity**:

```
                               ┌────────────────────────────────┐
                               │      Notification Queue        │
                               └────────────────────────────────┘
                                  │           │              │
        ┌─────────────────────────┘           │              └──────────────────────────┐
        ▼                                     ▼                                         ▼
┌──────────────────────┐            ┌──────────────────────┐                  ┌──────────────────────┐
│    Critical Topics   │            │   Standard Topics    │                  │  Promotional Topics  │
├──────────────────────┤            ├──────────────────────┤                  ├──────────────────────┤
│ - critical.sms       │            │ - standard.sms       │                  │ - promo.sms          │
│ - critical.email     │            │ - standard.email     │                  │ - promo.email        │
│ - critical.push      │            │ - standard.push      │                  │ - promo.push         │
└──────────────────────┘            └──────────────────────┘                  └──────────────────────┘
        │                                     │                                         │
        ▼                                     ▼                                         ▼
┌──────────────────────┐            ┌──────────────────────┐                  ┌──────────────────────┐
│ Dedicated OTP Pods   │            │ Standard Dispatchers │                  │ Bulk Campaign Pods   │
│ (Max replica scaling,│            │ (Autoscaled on lag)  │                  │ (Throttled, low-tier │
│ zero queue lag)      │            │                      │                  │  vendor bandwidth)   │
└──────────────────────┘            └──────────────────────┘                  └──────────────────────┘
```

#### Resilience Topics
1. **Retry Topic (`notification.retry`)**: Notifications encountering transient 5xx errors from vendors are requeued with exponential backoff (e.g., retry after 2s, 8s, 32s).
2. **Dead Letter Queue (`notification.dlq`)**: After 3 failed retries, messages enter the DLQ for manual operator triage, preventing bad payloads from blocking consumer partitions.

---

### 5.3 Deep Dive 3: Dual-Tier Rate Limiting Architecture

A critical production insight from Bunny is that rate limiting must be enforced at **two distinct boundaries**:

```
[ Inbound Clients ] ──> [ Tier 1: Ingress Rate Limiter ] ──> [ Notification Platform ]
                                                                      │
                                                                      ▼
                        [ Tier 2: Egress Rate Limiter  ] ──> [ External Vendors ]
                        (Twilio / SendGrid Concurrency)       (Twilio, APNs, FCM)
```

#### Tier 1: Ingress Rate Limiting (API Gateway)
- **Goal**: Protect our infrastructure from malicious clients, DDoS attacks, or buggy client loops.
- **Algorithm**: Redis-backed **Token Bucket** per `client_id`:
  - Tier Enterprise (Amazon): 20,000 requests/sec.
  - Tier Starter: 500 requests/sec.
- **Action on Breach**: Returns `429 Too Many Requests` with `Retry-After: 5` header.

#### Tier 2: Egress Rate Limiting (Provider Dispatchers)
- **Goal**: Protect relationships with external vendors and avoid hitting vendor-enforced account throttling.
  - *Example*: Twilio allocates 100 SMS/sec per short-code. Pushing 5,000 SMS/sec results in Twilio dropping messages or levying heavy penalty fees.
- **Implementation**: Distributed Redis Rate Limiter utilizing Lua scripts to acquire leaky-bucket tokens prior to dispatching HTTP calls to vendors:
  ```lua
  -- Atomic token bucket acquisition in Redis
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local current = tonumber(redis.call('get', key) or "0")

  if current + 1 > limit then
      return 0 -- Throttled
  else
      redis.call("INCRBY", key, 1)
      if current == 0 then
          redis.call("EXPIRE", key, 1) -- 1-second window
      end
      return 1 -- Permitted
  end
  ```

---

### 5.4 Deep Dive 4: User Preference Caching & Fast Invalidation

Before dispatching an SMS or promotional email, the dispatcher must evaluate the recipient's preference settings. 
Scanning the relational database on every single one of the $16,667\text{ QPS}$ would immediately saturate the database connection pool.

#### Cache-Aside Pattern with Redis
- **Cache Key**: `user:pref:{client_id}:{external_user_id}`
- **Data Structure**: Redis Hash or Bitfield storing boolean flags:
  ```redis
  HSET user:pref:cli_amz:usr_4412 email 1 sms 0 push 1 quiet_start 22 quiet_end 07
  ```
- **Read Path**: Dispatcher executes `HGETALL` in $< 1\text{ms}$. If `sms == 0`, the dispatcher instantly marks the record as `SKIPPED_USER_PREFERENCE` and avoids wasting vendor money.
- **Write Path & Invalidation**:
  - When the user updates preferences via `PUT /api/v1/users/{id}/preferences`, the Preference Service writes to the database and invalidates the Redis key (`DEL`).
  - Next dispatch repopulates the cache (Cache-Aside).

---

### 5.5 Deep Dive 5: Decoupled Delivery Tracking & Reporting Engine

A major anti-pattern highlighted by Bunny is having worker nodes directly write status changes (`SENT`, `DELIVERED`) into the operational database. Under 1M notifications/min, multiple concurrent writes lock database rows.

#### Decoupled Status Flow
```
[ Vendor Webhook / Callback ] 
             │
             ▼
[ Webhook Ingestion Service ]
             │
             ▼
┌───────────────────────────────┐
│ Kafka Topic: 'delivery-status'│
└───────────────────────────────┘
     │                     │
     ▼                     ▼
┌──────────────────┐  ┌───────────────────────────────┐
│ Status Consumer  │  │ BigQuery Analytics Consumer   │
└──────────────────┘  └───────────────────────────────┘
     │                     │
     ▼                     ▼
┌──────────────────┐  ┌───────────────────────────────┐
│ PostgreSQL DB    │  │ Google BigQuery / ClickHouse  │
│ (Single UPDATE:  │  │ (Immutable append of all      │
│  status='DELIV') │  │  lifecycle state transitions) │
└──────────────────┘  └───────────────────────────────┘
```
1. **Operational DB**: Retains only the **latest state** (`status = DELIVERED`, `delivered_at = NOW()`).
2. **Analytics DB (BigQuery / ClickHouse)**: Retains **full event history** with millisecond timestamps (`QUEUED` at 10:00:00.100, `DISPATCHED` at 10:00:00.350, `DELIVERED` at 10:00:01.200) for deep client analytics and funnel conversion reporting.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you ensure no notification is lost and implement at-least-once delivery guarantee?

At-least-once delivery with Transactional Outbox pattern:

- **Problem with dual writes**:
  Naive approach: INSERT notification to DB, then publish to Kafka, Issues: If Kafka publish succeeds but DB write fails → notification lost (no record in DB), If DB write succeeds but Kafka publish fails → notification stuck (never sent), Network failures, service crashes can cause inconsistency.

- **Outbox pattern solution**:
  Single transaction: BEGIN TRANSACTION, INSERT INTO notifications (notification_id, user_id, template_id, channel, payload, status: 'PENDING', priority, created_at), INSERT INTO notifications_outbox (outbox_id, notification_id, event_type: 'notification.created', payload, published: false, created_at), COMMIT (atomic - both succeed or both fail).

- **CDC/Publisher service**:
  Polls outbox table: SELECT * FROM notifications_outbox WHERE published=false ORDER BY created_at LIMIT 1000 (batch processing), For each unpublished record: Determine Kafka topic by priority + channel: high + email → 'notifications.critical.email', normal + sms → 'notifications.standard.sms', Publish: Kafka.send(topic, key: notification_id, value: payload), If successful: UPDATE notifications_outbox SET published=true, published_at=now() WHERE outbox_id={id}, If Kafka fails: published stays false, retry on next poll (every 100ms), Guarantees: If DB COMMIT succeeded → message in outbox → will be published eventually (retries until success).

- **At-least-once semantics**:
  Outbox → Kafka: at-least-once (may publish same message twice if CDC crashes after Kafka success but before marking published), Kafka → Consumer: at-least-once (Kafka may redeliver messages on consumer restart), Idempotency: Consumer checks delivery_status: SELECT COUNT(*) WHERE notification_id={id} AND channel={channel}, if exists → skip (already sent), prevents duplicate emails/SMS to users.

- **Exactly-once = at-least-once + idempotency**:
  Outbox ensures message never lost, Idempotent consumers ensure message processed only once, Result: Notification sent exactly once to user, even with failures/retries.

- **Failure handling**:
  CDC Publisher crashes: Restarts, polls outbox again, republishes unsent messages (published=false), Kafka partition leader fails: Kafka auto-elects new leader, CDC Publisher retries automatically, Consumer crashes: Kafka redelivers message to another consumer (consumer group), idempotency check prevents duplicate send.

- **Production example**:
  User places order → Order Service calls Notification API, Outbox ensures order confirmation email created (DB persisted), CDC Publisher sends to Kafka within 100ms, Email consumer delivers via SendGrid within 5 sec, Even if any component crashes: notification eventually sent (retries until success). Result: Zero notification loss, at-least-once delivery guaranteed, idempotent consumers prevent duplicates, resilient to failures (services, network, Kafka, providers).

---

### Q2: How do you handle rate limiting when sending millions of notifications to prevent overwhelming external providers?

Rate limiting with Token Bucket algorithm and Redis:

- **Provider limits**:
  Twilio SMS: 100 messages/sec per account, SendGrid Email: 50 emails/sec (free tier), 10K/sec (paid), FCM Push: 10K messages/sec, APNS: No documented limit but best practice <5K/sec.

- **Token Bucket algorithm**:
  Concept: Bucket holds tokens (e.g., 100 tokens), each send consumes 1 token, tokens refill at fixed rate (100 tokens/sec), if bucket empty → wait until refill.

- **Redis implementation**:
  Bucket key: rate_limit:email:sendgrid, Capacity: 100 tokens (max burst), Refill rate: 100 tokens/sec (matches SendGrid limit), Atomic Lua script (ensures thread-safety): local key = KEYS[1], local capacity = tonumber(ARGV[1]), local refill_rate = tonumber(ARGV[2]), local tokens = redis.call('GET', key) or capacity, local now = redis.call('TIME'), if tokens >= 1 then redis.call('DECR', key), return 1 (proceed), else return 0 (rate limited, wait), end, redis.call('EXPIRE', key, 10) (reset bucket after 10 sec idle).

- **Consumer flow**:
  Consumer receives notification from Kafka, Before sending: Check rate limit: result = redis.eval(lua_script, 'rate_limit:email:sendgrid', 100, 100), if result = 1 (tokens available): Send email via SendGrid, else (rate limited): Sleep 100ms, retry (busy wait), OR push back to Kafka (better - allows other consumers to process other messages).

- **Backpressure to Kafka**:
  If all consumers rate limited (provider at max capacity), Consumers slow down (sleep/retry), Kafka buffers messages (millions of messages capacity), Message lag increases (visible in monitoring), Auto-scaling: If lag > 1000 → add more consumers (but still subject to provider rate limit), If provider limit reached: Can't send faster (bottleneck), queue grows, eventual delivery (when capacity available).

- **Multi-provider strategy**:
  Primary: SendGrid (50 emails/sec), Fallback: AmazonSES (50 emails/sec), Total capacity: 100 emails/sec (both providers), Load balancing: Round-robin or weighted (SendGrid 70%, SES 30%), If SendGrid rate limited: Route to SES (circuit breaker pattern).

- **Priority handling**:
  Critical notifications (OTP): Bypass rate limiter (can't wait, time-sensitive), OR dedicated rate bucket (critical_rate_limit:sms:twilio with higher limit), Standard/Promotional: Subject to rate limit (acceptable delay).

- **Batching optimization**:
  SendGrid bulk API: Send 1000 emails in single API call (vs 1 email per call), Effective rate: 50 API calls/sec × 1000 emails/call = 50K emails/sec, Reduces API overhead, increases throughput 1000×.

- **Monitoring**:
  Track: Tokens available (should not hit 0 frequently), Rate limit events (how often consumers wait), Provider response times (detect slowdowns), Alert: If rate limit hit > 10% of time → increase capacity (upgrade provider tier, add fallback). Result: Providers never overwhelmed (respects rate limits), backpressure prevents system overload, multi-provider increases capacity, batching optimizes throughput, critical notifications prioritized.

---

### Q3: Walk through complete notification flow from creation to delivery with all state transitions and failure handling.

Complete notification flow (Order Confirmation Email):

- **Trigger**:
  User places order, Order Service calls: POST /api/v1/notifications {templateId: 'order_confirmation', recipientId: 'user123', variables: {name: 'Alice', order_id: 'ORD456', amount: '$49.99'}, channels: ['email', 'push'], priority: 'normal'}.

- **Notification Svc validation**:
  Authenticate request (API key/JWT), Fetch template: SELECT * FROM templates WHERE template_id='order_confirmation' AND is_active=true, template: {subject: 'Order {{order_id}} confirmed', body: 'Hi {{name}}, your order {{order_id}} for {{amount}} has been confirmed...'}, Validate variables: Check {name, order_id, amount} present, if missing → 400 error, Fetch user preferences: Redis GET user_pref:user123 → {channels: {email: true, push: true}, types: {transactional: true}}, Check: email enabled ✓, transactional enabled ✓ (order confirmation is transactional).

- **Render template**:
  Replace: subject: 'Order ORD456 confirmed', body: 'Hi Alice, your order ORD456 for $49.99 has been confirmed...', Generate notification_id: 'NOTIF789'.

- **Transactional write (Outbox pattern)**:
  BEGIN TRANSACTION, INSERT notifications: (notification_id: 'NOTIF789', user_id: 'user123', template_id, channel: 'EMAIL', payload: {subject, body, to: 'alice@example.com'}, status: 'PENDING', priority: 'normal', created_at: now()), INSERT notifications_outbox: (outbox_id, notification_id: 'NOTIF789', event_type: 'notification.created', payload, published: false), COMMIT, Response: 200 OK {notificationId: 'NOTIF789', status: 'PENDING'}.

- **CDC/Publisher polling**:
  SELECT * FROM notifications_outbox WHERE published=false (polls every 100ms), Finds: {outbox_id, notification_id: 'NOTIF789', payload, published: false}, Publish: Kafka.send('notifications.standard.email', key: 'NOTIF789', value: payload), Update: notifications_outbox SET published=true, published_at=now().

- **Email Consumer**:
  Subscribes to 'notifications.standard.email', Receives: {notification_id: 'NOTIF789', channel: 'EMAIL', payload: {subject, body, to}}, Idempotency: SELECT COUNT(*) FROM delivery_status WHERE notification_id='NOTIF789' AND channel='EMAIL', if 0 → proceed (first time), Rate limit: Check Redis token bucket: rate_limit:email:sendgrid, if tokens >= 1 → DECR, proceed, Send: POST SendGrid API {to: 'alice@example.com', subject, content: body}, Response: 202 Accepted {id: 'msg_sg123'}.

- **Update status**:
  UPDATE notifications SET status='SENT', sent_at=now(), INSERT delivery_status: (delivery_id, notification_id: 'NOTIF789', channel: 'EMAIL', provider: 'SendGrid', provider_message_id: 'msg_sg123', status: 'SENT', sent_at: now()).

- **SendGrid webhook (async)**:
  Minutes later: POST /webhooks/sendgrid {event: 'delivered', sg_message_id: 'msg_sg123', email: 'alice@example.com', timestamp}, Webhook handler: UPDATE delivery_status SET status='DELIVERED', delivered_at=now() WHERE provider_message_id='msg_sg123', UPDATE notifications SET status='DELIVERED'.

- **Push notification (parallel)**:
  Same flow for channel='PUSH', Consumer: Push consumer group, Send: FCM.send({token: user_device_token, notification: {title: 'Order Confirmed', body: 'Your order ORD456...'}}), Multi-channel: Email + Push sent independently (both succeed or fail independently).

- **Failure scenario (temporary)**:
  SendGrid timeout (10 sec no response), Consumer: Catch exception, Publish to 'notifications.retry': {notification_id: 'NOTIF789', channel: 'EMAIL', retry_count: 1, scheduled_at: now() + 1 min}, Retry consumer: Waits 1 min, reattempts send, If fails again: retry_count: 2, scheduled_at: now() + 5 min (exponential backoff), Max 3 retries: If all fail → move to DLQ (notifications.dlq), alert ops team.

- **Failure scenario (permanent)**:
  Email bounced (invalid address): SendGrid webhook: {event: 'bounce', reason: 'Invalid email'}, UPDATE delivery_status SET status='FAILED', error_message='Email bounced: invalid address', UPDATE notifications SET status='FAILED', Don't retry (permanent failure), Optionally: Notify sender (Order Service) that email failed.

- **State transitions**:
  PENDING (created, in outbox) → published to Kafka, SENT (delivered to provider, awaiting confirmation), DELIVERED (provider confirmed delivery to recipient), FAILED (permanent failure, don't retry), CANCELLED (user cancelled notification before sending). Result: Reliable delivery with state tracking, retry for temporary failures, idempotency prevents duplicates, webhooks confirm final delivery, multi-channel support, complete audit trail.

---

## 7. Architectural Trade-Off Analysis

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Durability Pattern** | Transactional Outbox + Debezium CDC | Direct Kafka Ingestion | Prevents lost messages during network blips; accepts $\sim 100\text{ms}$ CDC pipeline overhead for non-OTP messages. |
| **OTP Delivery Path** | Direct-to-Kafka Fast-Path | DB Outbox Flow | Bypasses database disk latency to achieve sub-second delivery; accepts minor durability risk since OTPs can be resent. |
| **User Preference Check** | Redis In-Memory Hash Cache | Relational DB Query on Dispatch | Prevents database connection exhaustion during 50,000 QPS marketing blasts; introduces minor eventual consistency lag. |
| **Delivery Status Tracking** | Decoupled Kafka Status Pipeline | Direct DB Updates from Workers | Eliminates row lock contention and database write saturation across hundreds of concurrent dispatch pods. |
| **Historical Reporting** | OLAP Append Store (BigQuery) | PostgreSQL Complex Aggregations | Isolates expensive analytical queries (e.g. 90-day deliverability funnels) from the operational transactional database. |

---

## 8. Failure Modes & Resiliency Strategies

1. **Vendor Outage (e.g., Twilio US-East down)**:
   - **Multi-Vendor Failover**: The SMS Dispatcher maintains dynamic health scores per vendor. If Twilio returns 5xx error spikes or latency exceeds 3 seconds, the dispatcher automatically reroutes traffic to **MessageBird** or **AWS SNS** using a Circuit Breaker pattern.
2. **Push Token Invalidation (App Uninstalled)**:
   - APNs and FCM return feedback codes (e.g., `DeviceTokenNotRegistered`).
   - The webhook listener intercepts this code and marks the device token as `DEACTIVATED` in the user registry to prevent future wasted requests.
3. **Consumer Lag During Festive Sales (Black Friday)**:
   - Consumer auto-scalers track **Kafka Consumer Lag**. If consumer lag exceeds 50,000 unread records on `standard.sms`, Kubernetes automatically scales dispatcher pods from 10 to 50 replicas within 60 seconds.
