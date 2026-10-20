# System Design: Email Delivery System (Gmail / Outlook)

> **Source**: [Interview with Bunny - Episode 20](https://youtu.be/87GhPP3jMp0)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/20](https://www.interviewwithbunny.com/systemdesign/20)  
> **Topic**: Designing a Global Distributed Email Platform (Gmail / Outlook)

---

## 1. Step 1: Requirements Gathering & Estimations

Email delivery systems are planet-scale, highly federated communication backbones. They interact not only with internal users within the same provider domain, but also with hundreds of thousands of independent external mail transfer agents (MTAs) across the public internet using standard protocol suites.

### 1.1 Functional Requirements (FRs)

1. **User Onboarding & Email Identity Reservation**:
   - Users can register unique email addresses (e.g., `alice@gmail.com`).
   - The system must strictly enforce global email uniqueness with zero duplicate allocations.
2. **Recipient Auto-Suggestion & Contact Resolution**:
   - Provide sub-millisecond recipient auto-suggestions in the "To / CC / BCC" composer based on user history, frequent recipients, and address book contacts across 5 distinct interaction scenarios.
3. **Draft Composition & Auto-Save**:
   - As users compose messages, the client periodically and automatically persists the draft content and attachment references without requiring explicit user saves.
4. **Attachment Uploading**:
   - Support large file attachments (up to 25 MB directly, or cloud links for larger files) using a decoupled two-step upload protocol.
5. **Send & Deliver Emails (Internal & External Domain Routing)**:
   - **Internal Delivery**: Directly and efficiently deliver emails addressed to users within the same provider domain (`@gmail.com` $\to$ `@gmail.com`).
   - **External Relay**: Deliver emails addressed to third-party domains (`@gmail.com` $\to$ `@outlook.com` / `@yahoo.com`) via standard Internet SMTP protocols and MX record resolution.
6. **Inbound Email Reception**:
   - Accept, authenticate, scan, and route incoming external emails arriving from third-party SMTP servers over the public internet.
7. **Mailbox Organization & Threading**:
   - Organize messages into folders (`Inbox`, `Sent`, `Drafts`, `Outbox`, `Trash`, `Spam`) and group replies into conversational threads.
8. **Full-Text Email Search**:
   - Allow users to search their personal email history by sender, recipient, subject, date range, and message body keywords.

---

### 1.2 Non-Functional Requirements (NFRs)

1. **Availability vs. Consistency (CAP Strategy)**:
   - **User Onboarding / Email Reservation (CP - Strict Consistency)**: Email registration must guarantee absolute consistency. Two users attempting to claim `bunny@gmail.com` concurrently must result in exactly one winner and one conflict rejection.
   - **Email Ingestion & Browsing (AP - High Availability)**: Reading inboxes, composing drafts, and initiating sends must be 99.999% available globally. Eventual consistency is standard for email delivery (transit times of a few seconds to a minute across the internet are normal).
2. **Reliability & Zero Message Loss (11 9's Durability)**:
   - Once a user clicks "Send", the message must never vanish. If downstream networks, spam filters, or receiving SMTP servers fail, messages must persist in an **Outbox buffer** with automated exponential backoff retries.
3. **Security & Sanitization**:
   - All inbound and outbound traffic must undergo multi-stage virus, malware, phishing, and enterprise Data Loss Prevention (DLP) inspection before final inbox placement or external relay.
4. **Low Latency & High Responsiveness**:
   - Recipient auto-complete suggestions: $<10\text{ ms}$.
   - Inbox view loading: $<100\text{ ms}$.

---

### 1.3 Capacity Estimations & Back-of-the-Envelope Math

#### Scale Assumptions:
- **Total Registered Accounts**: $1.5\text{ Billion}$ users
- **Daily Active Users (DAU)**: $500\text{ Million}$
- **Average Emails Sent per DAU**: $10\text{ emails/day}$
- **Average Emails Received per DAU** (personal + marketing/notifications): $40\text{ emails/day}$

#### Throughput Math:
- **Outbound Email Send QPS**:
  $$\text{Daily Outbound Volume} = 500,000,000 \times 10 = 5\text{ Billion emails/day}$$
  $$\text{Average Outbound QPS} = \frac{5,000,000,000}{86,400} \approx 57,870\text{ emails/sec}$$
  $$\text{Peak Outbound QPS (2.5}\times\text{)} \approx 145,000\text{ emails/sec}$$
- **Inbound Email Reception QPS**:
  $$\text{Daily Inbound Volume} = 500,000,000 \times 40 = 20\text{ Billion emails/day}$$
  $$\text{Average Inbound QPS} = \frac{20,000,000,000}{86,400} \approx 231,500\text{ emails/sec}$$
  $$\text{Peak Inbound QPS} \approx 500,000\text{ emails/sec}$$

#### Storage Estimations (5-Year Horizon):
- Average email payload:
  - Text metadata + body: $\approx 50\text{ KB}$
  - Attachments: $\approx 20\%$ of emails carry an attachment averaging $1\text{ MB} \implies \text{Weighted average} = 250\text{ KB}$ per email.
- **Daily Storage Growth**:
  $$\text{Daily Volume} = 25\text{ Billion total emails} \times 250\text{ KB} = 6.25\text{ Petabytes (PB) / day}$$
  $$\text{Annual Storage Growth} \approx 2.28\text{ Exabytes (EB) / year}$$
- Storage Architecture Mandate:
  - Highly optimized separation between **Email Metadata & Headers** (stored in distributed sharded NoSQL / Google Spanner / Bigtable) and **Raw Heavy Attachments & Blobs** (stored in distributed Object Storage like Google Cloud Storage / AWS S3).

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+       +--------------------+       +--------------------+
|        User        |       |    UserContact     |       |    Conversation    |
+--------------------+       +--------------------+       +--------------------+
| user_id (UUID)     | 1   * | id (UUID)          | 1   * | thread_id (UUID)   |
| email_address (PK) |<----->| owner_user_id(UUID)|<----->| subject (VARCHAR)  |
| password_hash      |       | contact_email(TEXT)|       | participant_hashes |
| status (ACTIVE)    |       | interaction_count  |       +--------------------+
+--------------------+       +--------------------+                 | 1
                                                                    | *
+--------------------+       +--------------------+       +--------------------+
|    MailboxItem     | *   1 |    EmailMessage    |       | AttachmentMetadata |
+--------------------+<----->+--------------------+       +--------------------+
| item_id (UUID)     |       | message_id (UUID)  | 1   * | id (UUID)          |
| user_id (UUID)     |       | thread_id (UUID)   |<----->| message_id (UUID)  |
| folder (INBOX/SENT)|       | sender_email(TEXT) |       | filename (VARCHAR) |
| message_id (UUID)  |       | to_list (JSONB)    |       | size_bytes (BIGINT)|
| is_read (BOOLEAN)  |       | subject (TEXT)     |       | s3_blob_key (TEXT) |
| received_at (TS)   |       | body_content (TEXT)|       | scan_status(CLEAN) |
+--------------------+       +--------------------+       +--------------------+
                                       | 1
                                       | 1
                             +--------------------+
                             |  ValidationRecord  |
                             +--------------------+
                             | message_id (UUID)  |
                             | spam_status (PASS) |
                             | malware_status(PASS|
                             | policy_check (PASS)|
                             | overall_verdict    |
                             +--------------------+
```

### Entity Descriptions:
1. **User**: Global unique identity record. `email_address` is the immutable primary key.
2. **UserContact & FrequentRecipient**: Local address book entries and weighted historical communication cache used for auto-complete.
3. **Conversation / Thread**: Clusters related messages sharing original headers (`In-Reply-To`, `References`).
4. **EmailMessage**: Immutable message payload including sender, recipient lists, subject, body text/HTML, and attachment references.
5. **MailboxItem**: Represents a user's pointer to an email within a specific folder (`INBOX`, `SENT`, `DRAFTS`, `OUTBOX`, `SPAM`, `TRASH`). Decouples message storage from user folder state.
6. **AttachmentMetadata**: Pointers to physical blobs stored in distributed object storage with checksums and virus scan verdicts.
7. **ValidationRecord**: State machine tracking asynchronous security, spam, and DLP validation gates.

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Recipient Auto-Complete & Contacts Query
```http
GET /api/v1/users/suggest?query=al&limit=5
Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "suggestions": [
    { "name": "Alice Smith", "email": "alice.smith@gmail.com", "source": "FREQUENT_CACHE" },
    { "name": "Alex Jones", "email": "alex@company.org", "source": "GOOGLE_CONTACTS" }
  ]
}
```

---

### 3.2 Decoupled Attachment Upload Protocol
#### Step 1: Pre-flight Upload Request
```http
POST /api/v1/attachments/upload-initiate
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "file_name": "quarterly_financials.pdf",
  "file_size": 14205000, // 14.2 MB
  "content_type": "application/pdf"
}

Response: 200 OK
{
  "attachment_id": "att_88129-bc7",
  "signed_upload_url": "https://storage.googleapis.com/mail-attachments/att_88129-bc7?signature=...",
  "expires_in_sec": 900
}
```
*(Client uploads the binary directly to Object Storage via HTTP PUT, bypassing backend microservice servers).*

---

### 3.3 Draft Auto-Save API
```http
PUT /api/v1/emails/drafts/{draft_id}
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "to": ["bob@outlook.com"],
  "cc": ["manager@gmail.com"],
  "subject": "System Design Deliverables",
  "body_html": "<p>Hi Bob, attached are the design blueprints...</p>",
  "attachment_ids": ["att_88129-bc7"],
  "thread_id": "thrd_4421-9988"
}

Response: 200 OK
{
  "draft_id": "drft_00129",
  "last_saved_at": "2026-10-18T10:14:22Z"
}
```

---

### 3.4 Send Email Request
```http
POST /api/v1/emails/send
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "draft_id": "drft_00129",
  "client_send_timestamp": 1729254862
}

Response: 202 Accepted
{
  "message_id": "msg_9941-8812",
  "status": "QUEUED_IN_OUTBOX",
  "queued_at": "2026-10-18T10:14:25Z"
}
```

---

### 3.5 Full-Text Email Search
```http
GET /api/v1/emails/search?q=financials+from:alice&folder=INBOX&page=1&limit=25
Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "total_hits": 4,
  "messages": [
    {
      "message_id": "msg_9941-8812",
      "thread_id": "thrd_4421-9988",
      "from": "alice@gmail.com",
      "subject": "System Design Deliverables",
      "snippet": "...attached are the design blueprints including quarterly financials...",
      "has_attachments": true,
      "received_at": "2026-10-18T10:14:30Z"
    }
  ]
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Web / Mobile Mail Client ]
            │
            ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           API Gateway & Load Balancer                           │
│        (SSL Termination, Rate Limiting, JWT Auth, Path-based Routing)           │
└─────────────────────────────────────────────────────────────────────────────────┘
      │                           │                          │
      ▼                           ▼                          ▼
┌──────────────┐          ┌──────────────┐           ┌──────────────┐
│ User Service │          │ Compose Mail │           │Mail Send Svc │
│ & AutoSug Svc│          │   Service    │           │ (Pre-flight) │
└──────────────┘          └──────────────┘           └──────────────┘
      │      │                    │                          │
      │      ▼                    ▼                          ▼
      │ ┌──────────┐      ┌──────────────┐           ┌──────────────┐
      │ │User Cache│      │   Draft DB   │           │Mailbox Outbox│
      │ │ (Redis)  │      │(Document DB) │           │(Spanner/PG)  │
      │ └──────────┘      └──────────────┘           └──────────────┘
      ▼                          │                          │ (Outbox CDC)
┌──────────────┐                 │                          ▼
│ Sharded User │                 │                  ┌───────────────┐
│ DB (Spanner) │                 │                  │Kafka: outbox- │
└──────────────┘                 │                  │ send-requests │
                                 │                  └───────────────┘
                                 │                          │
                                 │                          ▼
                                 │                  ┌────────────────────────┐
                                 │                  │  Outbox Delivery       │
                                 │                  │  Orchestration Engine  │
                                 │                  └────────────────────────┘
                                 │                   │          │          │
                                 │       ┌───────────┘          │          └───────────┐
                                 │       ▼                      ▼                      ▼
                                 │ ┌──────────┐           ┌──────────┐           ┌──────────┐
                                 │ │Spam/Phish│           │Policy/DLP│           │Attachment│
                                 │ │ Scanner  │           │  Engine  │           │Pre-check │
                                 │ └──────────┘           └──────────┘           └──────────┘
                                 │       │                      │                      │
                                 │       └───────────┬──────────┴──────────────────────┘
                                 │                   ▼
                                 │         ┌──────────────────┐
                                 │         │ Validation State │
                                 │         │  DB (Audit Log)  │
                                 │         └──────────────────┘
                                 │                   │ (All Clear / Sanitized)
                                 │                   ▼
                                 │         ┌──────────────────┐
                                 │         │  Domain Router   │
                                 │         └──────────────────┘
                                 │            │            │
            ┌────────────────────┘            │            └────────────────────┐
            ▼                                 ▼                                 ▼
┌───────────────────────┐         ┌───────────────────────┐         ┌───────────────────────┐
│Object Storage (S3/GCS)│         │Kafka: inbound-internal│         │Kafka: outbound-relay  │
│  [Attachment Blobs]   │         └───────────────────────┘         └───────────────────────┘
└───────────────────────┘                     │                                 │
                                              ▼                                 ▼
                                  ┌───────────────────────┐         ┌───────────────────────┐
                                  │ Delivery Consumer Svc │         │   SMTP Relay Worker   │
                                  └───────────────────────┘         └───────────────────────┘
                                              │                                 │
                                              ▼                                 ▼ (Port 25 TCP)
                                  ┌───────────────────────┐         ┌───────────────────────┐
                                  │  Recipient Mailbox DB │         │ External Internet     │
                                  │   (Inbox Placement)   │         │ (Outlook / Yahoo MTAs)│
                                  └───────────────────────┘         └───────────────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Global User DB Sharding & Concurrent Registration Safety

Bunny explains that at $1.5\text{ Billion}$ users, a naive monolithic relational database cannot scale:

#### 1. Sharding via Consistent Hashing:
- The system shards user records across hundreds of database partitions using a consistent hashing ring.
- **Partition Key**: The normalized, lower-cased email address:
  $$\text{Shard ID} = \text{ConsistentHash}(\text{normalize}(email\_address)) \pmod N$$
- *Why consistent hashing over simple modulo ($N$)?* Adding a new database shard only migrates $\frac{1}{N}$ of existing keys, eliminating the catastrophic re-sharding storms of fixed modulo arithmetic.

#### 2. Concurrency Control on Account Creation (Preventing Identity Stealing):
- When Alice and Bob attempt to register `innovator@gmail.com` at the exact same millisecond:
  - Both requests map to the exact same shard.
  - The table schema enforces a strict unique constraint:
    ```sql
    CREATE TABLE users (
      user_id UUID NOT NULL,
      email_address VARCHAR(255) PRIMARY KEY,
      password_hash VARCHAR(255) NOT NULL,
      status VARCHAR(32) NOT NULL,
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
    );
    ```
  - The database's primary key index engine serializes the write on the shard. The first transaction commits; the second fails with a `409 Conflict / UniqueConstraintViolation`. Zero duplicate email IDs can ever exist.

---

### 5.2 Deep Dive 2: The 5 Recipient Auto-Suggestion Scenarios & User Cache

Bunny breaks down a nuanced interview question: *How does the email client auto-suggest addresses without scanning a 1.5-billion-row user database on every keystroke?*

```
User types in recipient field:
         │
         ▼
[ Is prefix in Local Client Cache / LRU? ]
   ├── YES ──► Render Auto-Suggestion instantly (< 1 ms)
   └── NO
         │
[ Does sender type full email + press ENTER? ]
   ├── NO  ──► Do NOT search global DB (Protects system from keystroke DDoS)
   └── YES
         │
         ▼
[ Does recipient end in @gmail.com (Internal Domain)? ]
   ├── YES ──► Query Sharded User DB via Consistent Hash -> Render Info Card Popup
   └── NO  ──► External Domain -> No lookup possible; accept as raw recipient string
```

#### The 5 Scenarios Detailed:
1. **Scenario 1 (Unknown recipient, first time)**: Sender types `random_person@gmail.com`. Because this person is not in the sender's history, **zero auto-suggestions appear**. The system does not leak user existence on partial keystrokes.
2. **Scenario 2 (External domain, first time)**: Sender types `recruiter@bloomberg.com`. External identities are unknown to Gmail's user database $\implies$ **no auto-suggestion**.
3. **Scenario 3 (Previously emailed recipient)**: Sender previously exchanged emails with `recruiter@bloomberg.com`. The email is cached in the sender's dedicated **Frequent Recipient Cache** (Redis LRU storing top 100 historical contacts) $\implies$ **Auto-suggests on 2 characters**.
4. **Scenario 4 (Address Book / Google Contacts)**: Sender has `uncle_bob@yahoo.com` in Google Contacts. Synced to the user cache $\implies$ **Auto-suggests instantly**.
5. **Scenario 5 (Full Email Typed + ENTER on Internal Domain)**: Sender types `bunny.design@gmail.com` and hits Enter. The client fires a single targeted point-lookup to the sharded User DB. If found, it fetches the user's avatar, display name, and public profile card.

---

### 5.3 Deep Dive 3: The Transactional Outbox Pattern for Send Resilience

Bunny highlights a fatal system design flaw: *Calling the SMTP relay or external queues directly from the API handler*.
- If the application server crashes after receiving the send request but before the message enters the network queue, the email disappears without a trace.
- **The Outbox Pattern Guarantee**:
  1. The user clicks Send.
  2. The `Mail Send Service` writes the complete message payload into the user's `outbox_emails` table inside a single transactional boundary:
     ```sql
     BEGIN TRANSACTION;
     INSERT INTO outbox_emails (message_id, sender_id, recipient_list, payload_json, status, created_at)
     VALUES ('msg_9941', 'usr_001', '["bob@outlook.com"]', '...', 'PENDING', NOW());
     COMMIT;
     ```
  3. A dedicated **Change Data Capture (CDC)** engine (Debezium reading WAL / Binlog) streams new outbox records into the `outbox-send-requests` Kafka topic.
  4. If Kafka is temporarily down, the email safely sits in the persistent `outbox_emails` table until Kafka recovers.

---

### 5.4 Deep Dive 4: Asynchronous Validation & Scanning Pipeline

Emails must be inspected before dispatching to the internet to prevent Gmail from being blacklisted as a spam source:

```
┌────────────────────────────────────────────────────────┐
│              Outbox Delivery Orchestrator              │
└────────────────────────────────────────────────────────┘
          │ (Fires Async Parallel Inspection Jobs)
          ├────────────────────────┬────────────────────────┐
          ▼                        ▼                        ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Spam & Phishing │     │ Enterprise Policy│     │ Attachment Scan  │
│ Classifier (ML)  │     │   & DLP Engine   │     │ (Precomputed DB) │
└──────────────────┘     └──────────────────┘     └──────────────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   ▼
                      ┌──────────────────────────┐
                      │   Validation State DB    │
                      │ (Atomic Bitmask Updates) │
                      └──────────────────────────┘
```

#### Why Attachment Scanning Must Be Precomputed:
- Scanning a 25 MB `.zip` or `.pdf` file with anti-virus engines takes 3 to 10 seconds.
- Doing this synchronously at the moment of clicking "Send" degrades user experience and creates thread pool starvation.
- **Bunny's Precomputed Attachment Scan**:
  1. As soon as the user attaches a file during draft composition, S3 triggers an `ObjectCreated` event.
  2. The `Attachment Scanner Service` immediately scans the blob in the background and writes `status = 'CLEAN'` into `S3_Validation_DB`.
  3. When the user eventually clicks "Send", the Delivery Orchestrator performs an instant index lookup: `SELECT scan_status FROM attachment_scans WHERE id = :att_id`. Latency: $<2\text{ ms}$!

#### Preventing Pipeline Deadlocks via the Validation State DB:
- If the DLP/Policy Check service experiences a 10-minute outage:
  - Do *not* hold open thread pools in memory.
  - The status remains `PENDING_POLICY_CHECK` in the `Validation State DB`.
  - A scheduled sweeper worker re-evaluates lagging checks, isolating degraded checkers without stalling healthy outbound traffic.

---

### 5.5 Deep Dive 5: Internet SMTP Relay Protocol & MX Resolution Flow

When delivering an email from `alice@gmail.com` to `bob@outlook.com`, the **SMTP Relay Worker** executes the RFC 5321 Internet standard sequence:

```
[ Gmail SMTP Relay Worker ]                         [ Outlook MX Mail Server ]
            │                                                   │
            │ 1. DNS MX Query: "outlook.com"                    │
            │ ─────────────────────────────────► [ DNS Server ] │
            │ ◄───────────────────────────────── [ MX Records ] │
            │    (outlook-com.olc.protection.outlook.com: P10)  │
            │                                                   │
            │ 2. Open TCP Socket on Port 25                     │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 220 outlook.com ESMTP Server Ready
            │                                                   │
            │ 3. EHLO mail-relay.google.com                     │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 250-outlook.com Hello, 250-STARTTLS, 250 SIZE 36700160
            │                                                   │
            │ 4. STARTTLS (Upgrade to Encrypted TLS Connection) │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 220 2.0.0 SMTP server ready for TLS
            │                                                   │
            │ 5. MAIL FROM:<alice@gmail.com>                    │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 250 2.1.0 Sender OK
            │                                                   │
            │ 6. RCPT TO:<bob@outlook.com>                      │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 250 2.1.5 Recipient OK (Verified Bob Exists!)
            │                                                   │
            │ 7. DATA                                           │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 354 Start mail input; end with <CR><LF>.<CR><LF>
            │                                                   │
            │ 8. [Transmit MIME Headers, Body, S3 File Links]   │
            │    <CR><LF>.<CR><LF>                              │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 250 2.0.0 OK: Message queued for delivery
            │                                                   │
            │ 9. QUIT                                           │
            │──────────────────────────────────────────────────►│
            │ ◄─────────────────────────────────────────────────│ 221 2.0.0 Service closing transmission
```

#### Critical Protocol Nuance: Recipient Verification (Step 6)
- The sender transmits `RCPT TO:<bob@outlook.com>`.
- The Outlook server checks its own internal directory:
  - If `bob` exists $\implies$ Returns `250 Recipient OK`.
  - If `bob` does *not* exist $\implies$ Returns `550 5.1.1 User unknown`.
  - The Gmail relay worker catches the `550` error, aborts data transfer, and generates a **System Bounce Notification (Mailer-Daemon)** back into Alice's inbox: *"Address not found: Your message wasn't delivered to bob@outlook.com because the address couldn't be found."*

---

### 5.6 Deep Dive 6: Search Architecture (Aggregated Elasticsearch Pipeline)

- In the Mailbox Database, user emails are separated into normalized tables: `messages` (subject, body), `mailbox_items` (user_id, folder, read_state), and `attachments` (file types).
- Running runtime SQL `LIKE '%financials%'` or complex multi-table joins across billions of rows causes catastrophic database lock contention.
- **The Search Ingestion Engine**:
  1. Delivery consumers emit a `MessageDeliveredEvent` to Kafka.
  2. The **Search Aggregator Worker** joins message headers, body text, folder state, and attachment metadata into a single denormalized document.
  3. The document is indexed into **Elasticsearch** (or Google internal search index) partitioned by `user_id`.
  4. Search queries hit dedicated Elasticsearch clusters, isolating the transactional Mailbox DB from read search spikes.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you ensure reliable email delivery with retry mechanisms and handle failures?

Reliable email delivery with Outbox pattern and retry logic:

- **Outbox pattern (durable queue)**:
  When user sends email, INSERT INTO outbox table (durable storage), Publish to Kafka 'email.outbound', SMTP worker processes, If successful: DELETE FROM outbox, If failed: Keep in outbox for retry, Guarantees: Email survives server crash (persisted in DB), At-least-once delivery (retry until success or max attempts).

- **Retry strategy**:
  Exponential backoff: 1st retry after 1 min, 2nd after 5 min, 3rd after 15 min, max 10 attempts over 24 hours, Retry conditions: Temporary failures (4xx SMTP codes like 421 Service unavailable, 450 Mailbox busy), Network timeouts, Receiver server down.

- **Failure handling**:
  Permanent failures (5xx codes like 550 User not found, 552 Mailbox full): Stop retrying, Mark status='FAILED', Send bounce notification to sender: 'Delivery failed: User alice@yahoo.com does not exist', INSERT into sender's inbox as system message.

- **Dead Letter Queue (DLQ)**:
  After max retries (10 attempts, 24 hours), Move to DLQ Kafka topic: 'email.failed', Manual investigation by ops team, Possible issues: Misconfigured recipient server, Persistent network partition, Invalid recipient that passed initial validation.

- **Delivery confirmation**:
  SMTP 250 OK response → email accepted by receiver, UPDATE outbox SET status='SENT', sent_at=now(), DELETE FROM outbox, Receiver may still reject later (after accepting via SMTP) due to spam filters, Track final delivery via DSN (Delivery Status Notification) if supported.

- **Idempotency**:
  Same email_id sent multiple times (due to retries) → receiver deduplicates by message_id header: Message-ID: <{email_id}@gmail.com>, Prevents duplicate emails in user's inbox. Result: Guaranteed delivery with at-least-once semantics, graceful failure handling with user notification, durable queue survives failures, exponential backoff prevents overwhelming receiver.

---

### Q2: How does the Outbox Delivery Orchestrator implement SLA-based priority routing?

SLA-based priority routing with 6 scenarios:

- **Priority determination**:
  HIGH priority (target <1 min delivery): Same domain (gmail.com → gmail.com, internal routing), Known contacts (recipient in sender's contacts, auto-suggestion enabled), VIP senders (emails from boss@company.com), NORMAL priority (target 5 min): External first-time contact (gmail.com → yahoo.com, no prior interaction), Subject doesn't contain urgent keywords, LOW priority (target 30 min): Bulk sends (>100 recipients), Newsletter emails (detected by unsubscribe link, bulk headers), Marketing campaigns (tagged by sender).

- **Implementation**:
  Outbox Delivery Orchestrator consumes 'email.outbound', Determines priority: IF (recipient_domain == sender_domain): priority = 'HIGH' // Same domain, ELSE IF (recipient IN sender_contacts): priority = 'HIGH' // Known contact, ELSE IF (subject CONTAINS ['urgent', 'asap', 'immediate']): priority = 'ELEVATED', ELSE IF (recipient_count > 100 OR 'List-Unsubscribe' header present): priority = 'LOW', ELSE: priority = 'NORMAL', Routes to Kafka topic: email.smtp.{priority}, Kafka.send('email.smtp.high', emailPayload) or 'email.smtp.normal' or 'email.smtp.low'.

- **Worker allocation**:
  HIGH: 50 SMTP workers (dedicated, always available for fast delivery), NORMAL: 100 workers (majority of traffic, balanced capacity), LOW: 20 workers (rate limited, processes during off-peak to avoid impacting high/normal).

- **Auto-suggestion correlation**:
  Scenarios 1-6 from image all reference auto-suggestion, Auto-suggestion = recipient appears in sender's contact list or has prior email interaction, Enables HIGH priority routing (faster delivery for known contacts improves user experience).

- **Rate limiting per priority**:
  HIGH: No rate limit (immediate processing), NORMAL: 1000 emails/min per domain (gradual ramp-up for new domains), LOW: 100 emails/min per domain (prevents spam, spreads load).

- **Monitoring**:
  Track SLA compliance: % of HIGH priority emails delivered <1 min (target 99%), % of NORMAL delivered <5 min (target 95%), % of LOW delivered <30 min (target 90%), Alert if SLA violated (queue backlog, worker shortage). Result: Fast delivery for important emails (same domain, known contacts), efficient resource allocation (most workers on normal priority), abuse prevention (bulk emails rate limited), SLA-driven user experience.

---

### Q3: Walk through the complete SMTP delivery process with SPF, DKIM, and DMARC validation.

Complete SMTP delivery with authentication:

- **DNS MX lookup**:
  SMTP worker needs to send to alice@outlook.com, Query DNS: dig MX outlook.com, Response: outlook-com.olc.protection.outlook.com (priority 5) with IP addresses, Select lowest priority MX server (priority 5 = highest priority in MX records, lower number = higher priority).

- **TCP connection**:
  Connect to receiver SMTP server: TCP connection to outlook-com.olc.protection.outlook.com:25 (SMTP), Or port 587 (submission), port 465 (SMTPS with TLS), TLS handshake: STARTTLS command → upgrade to encrypted connection, Prevents email interception (MITM attacks).

- **SMTP handshake**:
  Client (Gmail SMTP worker): EHLO gmail.com, Server (Outlook): 250-outlook.com Hello [74.125.0.1], Server lists supported extensions: 250-STARTTLS, 250-AUTH PLAIN LOGIN, 250-SIZE 35000000 (max 35MB email).

- **Sender verification**:
  Client: MAIL FROM:<sender@gmail.com>, Server: 250 OK (accepts sender, may check later via SPF).

- **Recipient verification**:
  Client: RCPT TO:<alice@outlook.com>, Server checks local delivery table: SELECT user_id FROM users WHERE email='alice@outlook.com', If exists: 250 OK, If not exists: 550 User not found (reject, sender gets bounce).

- **Email transmission**:
  Client: DATA, Server: 354 Start mail input; end with <CRLF>.<CRLF>, Client sends headers + body: From: sender@gmail.com, To: alice@outlook.com, Subject: Meeting Tomorrow, Date: Mon, 26 Jan 2026 10:30:00 -0800, Message-ID: <E123@gmail.com>, DKIM-Signature: v=1; a=rsa-sha256; d=gmail.com; s=default; h=from:to:subject; bh=base64hash; b=signaturebase64, Content-Type: multipart/mixed (if attachments), Blank line, Email body (plain text or HTML), Attachments (Base64 encoded), Client: . (period on line by itself = end of message).

- **SPF validation (Sender Policy Framework)**:
  Server queries DNS: dig TXT gmail.com → spf.gmail.com, Response: v=spf1 ip4:74.125.0.0/16 ip4:74.125.224.0/19 ~all, Server checks: Is sending IP (74.125.0.1) in allowed ranges?, Result: PASS (IP authorized) or FAIL (unauthorized IP, likely spoofed), If FAIL: May reject email or mark as spam.

- **DKIM validation (DomainKeys Identified Mail)**:
  Server extracts DKIM-Signature header from email: d=gmail.com (signing domain), s=default (selector, identifies which key to use), h=from:to:subject (signed headers), bh=hash of body, b=signature of headers, Server fetches public key: dig TXT default._domainkey.gmail.com, Response: Public key (RSA or Ed25519), Server verifies: Hash email body → compare with bh, Sign headers using public key → compare with b, Result: PASS (signature valid, email not tampered) or FAIL (signature invalid, email modified).

- **DMARC validation (Domain-based Message Authentication)**:
  Server queries: dig TXT _dmarc.gmail.com, Response: v=DMARC1; p=reject; rua=mailto:dmarc@gmail.com (policy=reject strict, report aggregate to dmarc@gmail.com), Server checks alignment: Does 'From' header domain (gmail.com) match SPF/DKIM domain?, SPF: MAIL FROM domain = gmail.com? (aligned), DKIM: d= domain = gmail.com? (aligned), If SPF PASS AND aligned: DMARC PASS, If DKIM PASS AND aligned: DMARC PASS, If both FAIL + policy=reject: Reject email (550 DMARC policy violation), If both FAIL + policy=quarantine: Accept but mark as spam, If both FAIL + policy=none: Accept, log only.

- **Final acceptance**:
  If all checks pass (SPF, DKIM, DMARC, spam filters), Server: 250 OK Message accepted for delivery, Client: QUIT, Server: 221 Goodbye (close connection).

- **Post-delivery**:
  Server inserts email into alice@outlook.com's mailbox, Sends DMARC aggregate report to gmail.com (daily summary of auth results), Gmail tracks delivery confirmation, updates outbox status='SENT'. Result: Authenticated email delivery with anti-spoofing protection, SPF prevents IP spoofing, DKIM prevents email tampering, DMARC enforces policy, ensures legitimate emails delivered, spam/phishing blocked.

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **User Identity Database** | Consistent Hashing Ring + Relational Primary Key | Monolithic Database or Key-Value Store | Relational PK on sharded instance enforces strict ACID uniqueness for email handles; consistent hashing avoids re-sharding storms. |
| **Recipient Auto-Suggestion** | Tiered Local Cache (Redis LRU / Google Contacts) | Keystroke DB queries against global user directory | Prevents catastrophic keystroke DDoS on DB; protects user privacy by never revealing un-contacted accounts. |
| **Attachment Transfer** | Direct Object Storage (S3/GCS) via Pre-signed URLs | Streaming attachments through backend application servers | Keeps multi-megabyte binary payloads off application server memory and socket queues. |
| **Outbound Reliability** | Transactional Outbox + Debezium CDC | Direct async HTTP / Queue calls from API handler | Guarantees zero message loss even under catastrophic node crashes or network partitions. |
| **Security Validation** | Async Parallel Orchestrator + Precomputed Attachment Scan | Synchronous serial scanning at click of "Send" | Precomputing attachment scans during draft authoring reduces send latency from 5+ seconds to $<100\text{ ms}$. |
| **External Mail Transit** | Standard SMTP (Port 25) with DNS MX Resolver | Proprietary RPC / HTTP APIs | Mandatory compliance with global RFC 5321 Internet standards for federated inter-domain communication. |
