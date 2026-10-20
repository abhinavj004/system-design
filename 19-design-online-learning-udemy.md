# System Design: Online Learning Platform (Udemy / Coursera)

> **Source**: [Interview with Bunny - Episode 19](https://youtu.be/PUf-Q7lNtv8)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/19](https://www.interviewwithbunny.com/systemdesign/19)  
> **Topic**: Designing a Scalable Online Learning & Teaching Platform (Udemy / Coursera)

---

## 1. Step 1: Requirements Gathering & Estimations

An online learning platform connects instructors who create and monetize educational material with millions of global students consuming video courses, taking assessments, and tracking curriculum progress.

### 1.1 Functional Requirements (FRs)

#### Student Capabilities:
1. **User Onboarding & Authentication**: Students can register, log in, manage profiles, and authenticate securely via JWT/SSO.
2. **Course Discovery & Search**: Students can search for courses via full-text keyword search and filter by category, difficulty level, language, price, and ratings.
3. **Course Enrollment & Payments**:
   - Free courses allow instant zero-friction enrollment.
   - Paid courses integrate with secure payment gateways (Stripe/PayPal/Razorpay). Course access must be granted immediately upon successful settlement.
4. **Adaptive Video Streaming**: Students can stream course lectures in real-time with nominal startup latency (< 2 seconds), adapting dynamically to varying network bandwidths (480p to 4K).
5. **Curriculum Progress Tracking**: System tracks individual lecture completion and timestamp playback progress, resuming playback across devices.
6. **Assessments & Quizzes**: Students can submit quizzes, coding exercises, or assignments and receive automated scores.
7. **Reviews & Discussion**: Students can rate, review, and leave comments on enrolled courses.

#### Instructor Capabilities:
8. **Instructor Onboarding & KYC Verification**: Instructors must undergo identity and credentials verification (e.g., government ID, passport, or national identity API like UIDAI) before publishing paid content.
9. **Curriculum & Content Management**: Instructors can structure courses into modules/chapters, upload video lectures, attachments, quizzes, and manage pricing.
10. **Course Moderation Workflow**: Uploaded courses remain in draft/pending status until reviewed and approved by moderators/automated safety scanners prior to global publishing.

#### Enterprise Extension (Google / Microsoft Interview Follow-Up):
11. **B2B Multi-Tenancy**: Support enterprise organizations (e.g., Walmart, Microsoft) with custom subdomains, corporate SSO, tenant-branded landing pages, and dedicated tenant data isolation.

---

### 1.2 Non-Functional Requirements (NFRs)

1. **High Availability vs. Consistency (CAP Strategy)**:
   - **Browsing, Search & Video Playback (AP - High Availability)**: Catalog browsing and video playback must remain 99.99% available globally. Eventual consistency is completely acceptable for catalog updates and newly published courses.
   - **Payment & Access Entitlement (CP - Strict Consistency)**: The payment processing and enrollment entitlement path must be strictly consistent. A user who pays must never experience access denial, nor must phantom free entitlements occur.
2. **Low Latency Playback**:
   - Video startup time must be under **$2\text{ seconds}$** anywhere in the world.
   - Course catalog search results must return in under **$100\text{ ms}$**.
3. **Durability & Fault Tolerance**:
   - Raw video master files and course assets stored in Cloud Object Storage must achieve **$99.999999999\%$ (11 9's)** durability.
4. **Resilience under Telemetry Spikes**:
   - Millions of concurrent video watch sessions send progress heartbeats every 10 seconds; the ingest tier must absorb this telemetry without causing database write exhaustion.

---

### 1.3 Capacity Estimations & Back-of-the-Envelope Math

#### Scale Assumptions:
- **Total Registered Users**: $50\text{ Million}$
- **Daily Active Users (DAU)**: $5\text{ Million}$
- **Total Published Courses**: $250,000$ courses
- **Average Lectures per Course**: $40$ lectures $\implies 10\text{ Million}$ total video assets.
- **Average Lecture Duration**: $15\text{ minutes}$.

#### Throughput Math:
- **Search & Catalog Read QPS**:
  - Each active user performs $\approx 10$ search/browse queries per day:
    $$\text{Daily Queries} = 5,000,000 \times 10 = 50,000,000\text{ requests/day}$$
    $$\text{Average Read QPS} = \frac{50,000,000}{86,400} \approx 580\text{ QPS}$$
    $$\text{Peak Read QPS (3}\times\text{)} \approx 1,750\text{ QPS}$$
- **Video Playback Sessions**:
  - Peak concurrent video viewers: $500,000$ concurrent streams.
  - Heartbeat telemetry sent every $10\text{ seconds}$ per active stream:
    $$\text{Telemetry Ingest QPS} = \frac{500,000\text{ streams}}{10\text{ seconds}} = 50,000\text{ writes/sec (Telemetry)}$$
    *(Bunny highlights this massive write volume: persisting 50,000 writes/sec directly into a transactional relational DB will crash it!)*

#### Storage Math:
- **Video Storage**:
  - Raw master video ($1080\text{p}$, $15\text{ min}$) $\approx 500\text{ MB}$.
  - Multi-bitrate transcoding ($1080\text{p}, 720\text{p}, 480\text{p}, 360\text{p}$) $\approx 1\text{ GB}$ per lecture.
  - $10\text{ Million lectures} \times 1\text{ GB} = 10\text{ Petabytes (PB)}$ of encoded video in object storage.
- **Metadata Storage**:
  - Course metadata record: $\approx 2\text{ KB} \times 250,000 \approx 500\text{ MB}$.
  - User records: $50\text{M} \times 1\text{ KB} = 50\text{ GB}$.
  - Progress tracking records: $50\text{M users} \times 5\text{ enrolled courses} \times 40\text{ lectures} \times 100\text{ bytes} \approx 1\text{ Terabyte (TB)}$.

#### Bandwidth Math (Egress):
- $500,000$ concurrent streams streaming at an average adaptive bitrate of $2\text{ Mbps}$:
  $$\text{Peak Egress Bandwidth} = 500,000 \times 2\text{ Mbps} = 1,000,000\text{ Mbps} = 1\text{ Tbps (Terabits per second)}$$
  *(This massive egress mandates multi-CDN edge delivery directly from S3/CloudFront/Akamai, never routing video payloads through microservice application servers).*

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+       +--------------------+       +--------------------+
|        User        |       |     Instructor     |       |       Course       |
+--------------------+       +--------------------+       +--------------------+
| id (UUID)          | 1   1 | id (UUID)          | 1   * | id (UUID)          |
| email (VARCHAR)    |<----->| user_id (UUID)     |<----->| instructor_id(UUID)|
| role (STUDENT/INST)|       | bio (TEXT)         |       | title (VARCHAR)    |
| password_hash      |       | verified_status    |       | status (DRAFT/PUB) |
+--------------------+       | gov_doc_id         |       +--------------------+
                             +--------------------+                 | 1
                                                                    | *
+--------------------+       +--------------------+       +--------------------+
|     Enrollment     | *   1 |   PaymentOrder     |       |   CourseModule     |
+--------------------+<----->+--------------------+       +--------------------+
| access_id (UUID)   |       | order_id (UUID)    |       | id (UUID)          |
| user_id (UUID)     |       | user_id (UUID)     |       | course_id (UUID)   |
| course_id (UUID)   |       | course_id (UUID)   |       | title (VARCHAR)    |
| access_type        |       | amount (DECIMAL)   |       | sequence_order     |
| status (ACTIVE)    |       | status (PAID/FAIL) |       +--------------------+
+--------------------+       +--------------------+                 | 1
          | 1                                                       | *
          | *                                             +--------------------+
+--------------------+       +--------------------+       |   LectureContent   |
|    UserProgress    | *   1 |   QuizAssessment   |       +--------------------+
+--------------------+<----->+--------------------+       | id (UUID)          |
| user_id (UUID)     |       | id (UUID)          |       | module_id (UUID)   |
| lecture_id (UUID)  |       | lecture_id (UUID)  |       | type (VIDEO/QUIZ)  |
| watched_seconds    |       | questions (JSON)   |       | video_s3_key       |
| is_completed (BOOL)|       | passing_score      |       | duration_sec       |
+--------------------+       +--------------------+       +--------------------+
```

### Entity Descriptions:
1. **User**: Common identity profile, authentication credentials, role flags (`STUDENT`, `INSTRUCTOR`, `ADMIN`).
2. **Instructor**: Verification profile, Aadhaar/Govt ID verification reference, payout banking details, cumulative ratings.
3. **Course**: Master course entity containing title, slug, description, category, difficulty level, thumbnail URL, and lifecycle status (`DRAFT`, `UNDER_REVIEW`, `PUBLISHED`, `ARCHIVED`).
4. **CoursePrice & CourseStats**: Decoupled pricing rules (currency, tier, discounts) and aggregated metrics (average rating, review count, student count).
5. **CourseModule & LectureContent**: Hierarchical curriculum structure. Modules represent sections; lectures represent granular assets (video S3 references, duration, preview allowance).
6. **CourseEnrollment (Entitlement)**: Authorization mapping linking a user to a purchased course with scope (`LIFETIME`, `SUBSCRIPTION`) and state (`ACTIVE`, `EXPIRED`, `REVOKED`).
7. **PaymentOrder & PaymentOutbox**: Financial audit log of transactions, gateway references, and transactional outbox entries.
8. **UserProgress**: State of video playback timestamps, completion booleans, and percentage of course completed.
9. **QuizAssessment & Question**: Associated evaluation modules, question payloads, scoring weights, and student submission histories.
10. **ReviewComment**: User ratings (1–5 stars) and qualitative feedback attached to courses.

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Course Discovery & Search
```http
GET /api/v1/courses/search?q=system+design&category=software&level=intermediate&rating=4.5&price=paid&page=1&limit=20
Accept: application/json

Response: 200 OK
{
  "total_hits": 142,
  "page": 1,
  "limit": 20,
  "courses": [
    {
      "course_id": "crs_8829-411a",
      "title": "Mastering Distributed System Design",
      "instructor": {
        "id": "inst_102",
        "name": "Bunny",
        "is_verified": true
      },
      "thumbnail_url": "https://cdn.udemy-cdn.com/thumbs/crs_8829.jpg",
      "price": { "amount": 499.00, "currency": "INR" },
      "rating": 4.9,
      "total_reviews": 1240,
      "total_students": 18500
    }
  ]
}
```

---

### 3.2 Instructor Course Creation & Chunk Upload Pre-Flight
```http
POST /api/v1/instructor/courses/{course_id}/lectures/{lecture_id}/media/upload-initiate
Authorization: Bearer <instructor_jwt>
Content-Type: application/json

Request Body:
{
  "file_name": "lecture_01_url_shortener.mp4",
  "file_size_bytes": 1073741824, // 1 GB
  "content_type": "video/mp4",
  "chunk_count": 10
}

Response: 200 OK
{
  "upload_session_id": "sess_upl_991823",
  "recommended_chunk_size": 104857600, // 100 MB
  "part_urls": [
    { "part_number": 1, "signed_s3_url": "https://s3.amazonaws.com/udemy-raw-videos/crs_8829/lec_01.mp4?partNumber=1&uploadId=xyz..." },
    { "part_number": 2, "signed_s3_url": "https://s3.amazonaws.com/udemy-raw-videos/crs_8829/lec_01.mp4?partNumber=2&uploadId=xyz..." }
  ]
}
```

---

### 3.3 Course Enrollment & Payment Checkout
```http
POST /api/v1/courses/{course_id}/enroll
Authorization: Bearer <student_jwt>
Content-Type: application/json

Request Body:
{
  "currency": "INR",
  "payment_method": "STRIPE",
  "idempotency_key": "idemp_5599-2231-ab88"
}

Response: 200 OK (For Free Course)
{
  "status": "ENROLLED",
  "access_id": "acc_7721-002",
  "course_id": "crs_8829-411a",
  "enrolled_at": "2026-10-18T10:30:00Z"
}

Response: 402 Payment Required / Redirect (For Paid Course)
{
  "status": "PAYMENT_PENDING",
  "order_id": "ord_9901-4412",
  "checkout_url": "https://checkout.stripe.com/c/pay/cs_live_xyz9812",
  "amount": 499.00,
  "currency": "INR"
}
```

---

### 3.4 Video Playback Manifest
```http
GET /api/v1/courses/{course_id}/lectures/{lecture_id}/playback-manifest
Authorization: Bearer <student_jwt>

Response: 200 OK
{
  "lecture_id": "lec_01_url",
  "drm_protected": false,
  "manifest_type": "HLS",
  "master_playlist_url": "https://edge-cdn.udemy.com/hls/crs_8829/lec_01/master.m3u8?jwt_token=exp_1730000000",
  "subtitles": [
    { "lang": "en", "url": "https://edge-cdn.udemy.com/subs/lec_01_en.vtt" }
  ],
  "last_watched_timestamp_sec": 412
}
```

---

### 3.5 High-Throughput Progress Telemetry Heartbeat
```http
POST /api/v1/progress/heartbeat
Authorization: Bearer <student_jwt>
Content-Type: application/json

Request Body:
{
  "course_id": "crs_8829-411a",
  "lecture_id": "lec_01_url",
  "current_position_sec": 420,
  "duration_sec": 900,
  "client_timestamp": 1729254000
}

Response: 202 Accepted
{
  "recorded": true
}
```

---

## 4. Step 4: High-Level Architecture

The architecture is partitioned into two distinct planes: the **Instructor Ingestion & Moderation Plane** and the **Student Learning & Playback Plane**.

```
[ Students (Web / Mobile / TV) ]             [ Instructors & Admin Moderators ]
               │                                              │
               ▼                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           API Gateway & Load Balancer                           │
│        (SSL Termination, Rate Limiting, JWT Validation, Tenant Subdomain)       │
└─────────────────────────────────────────────────────────────────────────────────┘
      │                    │                    │                     │
      ▼                    ▼                    ▼                     ▼
┌──────────────┐   ┌──────────────┐    ┌─────────────────┐   ┌─────────────────┐
│ User & Auth  │   │ Course Search│    │   Enrollment    │   │ Course Catalog  │
│   Service    │   │   Service    │    │   & Payments    │   │  & Moderation   │
└──────────────┘   └──────────────┘    └─────────────────┘   └─────────────────┘
      │                    │                    │                     │
      ▼                    ▼                    ▼                     ▼
┌──────────────┐   ┌──────────────┐    ┌─────────────────┐   ┌─────────────────┐
│   User DB    │   │Elasticsearch │    │Payment Outbox DB│   │ Course RDBMS DB │
│ (PostgreSQL) │   │(Precomputed) │    │  (PostgreSQL)   │   │(Modules/Lessons)│
└──────────────┘   └──────────────┘    └─────────────────┘   └─────────────────┘
                          ▲                     │                     │
                          │                     ▼                     ▼
                   ┌──────────────┐    ┌─────────────────┐     ┌──────────────┐
                   │Aggregator CDC│    │  Transactional  │     │Media Uploader│
                   │(Debezium/app)│    │   CDC Pipeline  │     │   Service    │
                   └──────────────┘    └─────────────────┘     └──────────────┘
                          ▲                     │                     │
                          │                     ▼                     │
┌─────────────────────────────────┐    ┌─────────────────┐            │
│  User Progress Telemetry Stream │    │  Kafka Broker   │            │
│  [Heartbeat API] ──► [Kafka]    │    │ (Payment Events)│            │
└─────────────────────────────────┘    └─────────────────┘            │
                 │                              │                     │
                 ▼                              ▼                     │
       ┌──────────────────┐            ┌─────────────────┐            │
       │ Progress Flusher │            │ Permission Sync │            │
       │ (1-Min Agg Batch)│            │    Consumer     │            │
       └──────────────────┘            └─────────────────┘            │
                 │                              │                     │
                 ▼                              ▼                     │
       ┌──────────────────┐            ┌─────────────────┐            ▼
       │   Progress DB    │            │ Course Access DB│    ┌──────────────┐
       │ (Cassandra/PG)   │            │  (Entitlements) │    │Pre-signed S3 │
       └──────────────────┘            └─────────────────┘    └──────────────┘
                                                ▲                     │
                                                │ (Verify Access)     ▼
                                       ┌─────────────────┐    ┌──────────────┐
                                       │ Playback Service│    │AWS S3 Storage│
                                       │(Signs CDN Token)│    │ (Raw Chunks) │
                                       └─────────────────┘    └──────────────┘
                                                │                     │
                                                ▼                     ▼
                                       ┌─────────────────┐    ┌──────────────┐
                                       │ Edge CDN Tier   │◄───│  Transcoder  │
                                       │(CloudFront/Akam)│    │Worker (FFmpeg│
                                       └─────────────────┘    └──────────────┘
                                                │
                                                ▼ (HLS Multi-Bitrate Streams)
                                     [ Video Player Client ]
```

### Component Interaction Flow:
1. **Catalog Ingestion**: Instructor uploads course structure to the `Course Catalog Service` (PostgreSQL). Raw videos are uploaded directly from the browser to Amazon S3 using pre-signed URLs provided by the `Media Uploader Service`.
2. **Moderation & Transcoding**: Admin moderator validates content via the `Course Moderation Service`. Upon approval, asynchronous FFmpeg transcoder workers encode the video into HLS streams ($1080\text{p}, 720\text{p}, 480\text{p}, 360\text{p}$) and generate `.m3u8` playlists.
3. **Search Ingestion**: Changes in the Course DB trigger a Change Data Capture (CDC) aggregator that flattens relational data (titles, instructors, stats, ratings, tags) into JSON documents indexed in **Elasticsearch**.
4. **Checkout & Entitlement**: On payment completion, the payment transaction and an Outbox event are committed atomically to PostgreSQL. Debezium pumps the event into Kafka. The `Permission Sync Service` consumes the event and inserts entitlement into `Course Access DB`.
5. **Video Playback**: The client requests a playback manifest. The `Playback Service` verifies the user's entitlement against `Course Access DB` and returns a time-limited signed CloudFront/Akamai CDN URL for the `.m3u8` master manifest. The client streams chunks directly from the nearest CDN edge node.
6. **Progress Tracking**: The client fires lightweight telemetry heartbeats every 10 seconds into Kafka. Background workers debounce and batch write the terminal timestamp into the `Progress DB` once per minute.

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Video Ingestion, Transcoding & Adaptive Bitrate Streaming (HLS/DASH)

Bunny emphasizes that video handling is the single most bandwidth-intensive component of Udemy:
- **Never Route Raw Video Through Backend Microservices**: Uploading a 2 GB lecture directly through an application pod locks up socket buffers and saturates backend network interfaces.
- **Pre-signed Multi-Part S3 Uploads**:
  1. Instructor client queries `/upload-initiate`.
  2. Media Uploader Service calls AWS S3 `CreateMultipartUpload` and returns pre-signed PUT URLs for each 50 MB chunk.
  3. Client uploads chunks in parallel directly to S3.
  4. Client calls `/upload-complete`; S3 stitches parts into the master video object.

```
[ Instructor Browser ]
         │ 1. Request Signed URLs
         ▼
┌───────────────────────┐
│ Media Uploader Service│
└───────────────────────┘
         │ 2. S3:CreateMultipartUpload
         ▼
┌───────────────────────┐ 3. Returns Signed URLs  ┌──────────────────────┐
│  Object Storage (S3)  │◄─────────────────────────│  Instructor Browser  │
│     [Raw Bucket]      │  Direct Multipart Upload │ (Parallel 50MB Chunks│
└───────────────────────┘                          └──────────────────────┘
         │ 4. S3 ObjectCreated Event
         ▼
┌───────────────────────┐
│ AWS SQS / Kafka Queue │
└───────────────────────┘
         │ 5. Pull Transcoding Job
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│               Transcoding Fleet (FFmpeg / AWS MediaConvert)             │
│  - Decode raw input video                                              │
│  - Encode profiles: 1080p (4.5M), 720p (2.5M), 480p (1M), 360p (400k)   │
│  - Segment into 6-second `.ts` or `.m4s` chunks                        │
│  - Generate Master `.m3u8` and variant playlists                       │
└────────────────────────────────────────────────────────────────────────┘
         │ 6. Store Transcoded Segments & Manifests
         ▼
┌───────────────────────┐ 7. Edge Invalidation & Warmup ┌─────────────────┐
│  Object Storage (S3)  │───────────────────────────────►│  CDN Edge Tier  │
│  [Transcoded Bucket]  │                                │ (CloudFront)   │
└───────────────────────┘                                └─────────────────┘
```

#### Adaptive Bitrate Streaming (ABR):
- The client video player reads the master manifest (`master.m3u8`):
  ```m3u8
  #EXTM3U
  #EXT-X-STREAM-INF:BANDWIDTH=4500000,RESOLUTION=1920x1080
  1080p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
  720p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
  480p/index.m3u8
  ```
- **Dynamic Quality Switching**: The client constantly calculates download throughput of the last 6-second chunk. If network latency spikes, the player requests the next segment from the 480p playlist seamlessly without buffering.

---

### 5.2 Deep Dive 2: Transactional Outbox Pattern for Payment & Entitlements

Bunny highlights a critical interview trap: **Dual Database Writes across Microservices**.
- *The Trap*: When a payment succeeds, the Payment Service writes to `payment_db` and then makes a synchronous HTTP/gRPC call to `enrollment_service` to write to `course_enrollment_db`. If network fails or the enrollment service is restarting, money is debited but the student receives zero access.
- *The Optimal Solution*: **Transactional Outbox with Debezium CDC**.

```sql
-- Executed inside a single ACID database transaction in Payment DB:
BEGIN TRANSACTION;

-- 1. Insert Payment Audit Record
INSERT INTO payment_orders (order_id, user_id, course_id, amount, currency, status, created_at)
VALUES ('ord_9901', 'usr_4412', 'crs_8829', 499.00, 'INR', 'COMPLETED', NOW());

-- 2. Insert Event into Outbox Table
INSERT INTO payment_outbox (event_id, aggregate_type, aggregate_id, event_type, payload, processed)
VALUES (
  'evt_00192',
  'COURSE_PURCHASE',
  'ord_9901',
  'PaymentSuccessfulEvent',
  '{"user_id": "usr_4412", "course_id": "crs_8829", "order_id": "ord_9901", "access_type": "LIFETIME"}',
  FALSE
);

COMMIT;
```

```
┌────────────────────────────────────────────────────────┐
│                   Payment PostgreSQL                   │
│   ┌─────────────────────┐    ┌─────────────────────┐   │
│   │   payment_orders    │    │   payment_outbox    │   │
│   └─────────────────────┘    └─────────────────────┘   │
└──────────────────────────────────────────┬─────────────┘
                                           │ (WAL / Binlog)
                                           ▼
                                ┌─────────────────────┐
                                │Debezium CDC Engine  │
                                └─────────────────────┘
                                           │
                                           ▼
                                ┌─────────────────────┐
                                │Kafka Topic: payments│
                                └─────────────────────┘
                                     │            │
            ┌────────────────────────┘            └────────────────────────┐
            ▼                                                              ▼
┌────────────────────────┐                                     ┌────────────────────────┐
│  Permission Sync Svc   │                                     │  Notification Service  │
└────────────────────────┘                                     └────────────────────────┘
            │                                                              │
            ▼                                                              ▼
┌────────────────────────┐                                     ┌────────────────────────┐
│  Course Enrollment DB  │                                     │  Sends Email / Receipt │
│ (usr_4412 -> crs_8829) │                                     │  to Student via SES    │
└────────────────────────┘                                     └────────────────────────┘
```

> **Resilience Guarantee**: If Kafka is temporarily down, the event remains safe in the relational `payment_outbox` table. As soon as Kafka recovers, Debezium resumes from the last confirmed log sequence number (LSN). Zero lost purchases.

---

### 5.3 Deep Dive 3: Search Pipeline & Precomputed Aggregations (Elasticsearch)

- **The Problem**: In PostgreSQL, course metadata is normalized across 5 tables: `courses`, `course_prices`, `instructors`, `course_stats`, and `course_tags`.
  ```sql
  -- NAIVE RUNTIME JOIN (Bottlenecks at 1,000 QPS)
  SELECT c.id, c.title, i.name, p.amount, s.avg_rating 
  FROM courses c
  JOIN instructors i ON c.instructor_id = i.id
  JOIN course_prices p ON c.id = p.course_id
  JOIN course_stats s ON c.id = s.course_id
  WHERE c.category = 'Tech' AND s.avg_rating >= 4.5
  ORDER BY s.enrolled_count DESC LIMIT 20;
  ```
- **Bunny's Aggregator CDC Pattern**:
  1. Any insert/update to `courses`, `instructors`, or `course_stats` emits a CDC event.
  2. The **Aggregator CDC Worker** fetches all joined data, constructs a denormalized JSON document, and indexes it into **Elasticsearch**.
  3. Elasticsearch provides sub-$30\text{ ms}$ multi-facet queries, typos tolerance (Lucene fuzzy queries), and auto-complete suggestions.

#### Denormalized Elasticsearch Document Schema:
```json
{
  "course_id": "crs_8829-411a",
  "title": "Mastering Distributed System Design",
  "slug": "mastering-distributed-system-design",
  "description": "Comprehensive guide to cracking L6/L7 system design interviews...",
  "instructor": {
    "id": "inst_102",
    "name": "Bunny",
    "is_verified": true,
    "rating": 4.9
  },
  "category": "Software Engineering",
  "sub_category": "System Design",
  "level": "INTERMEDIATE",
  "price": {
    "INR": 499.00,
    "USD": 19.99
  },
  "stats": {
    "enrolled_count": 18500,
    "average_rating": 4.92,
    "review_count": 1240
  },
  "tags": ["system design", "microservices", "kafka", "distributed systems"],
  "status": "PUBLISHED",
  "last_updated_at": 1729250000
}
```

---

### 5.4 Deep Dive 4: User Progress Telemetry Buffer (Mitigating Write Exhaustion)

- **The Problem**: 500,000 active concurrent viewers streaming video send a progress ping every 10 seconds.
  $$\text{Raw Telemetry Writes} = 50,000\text{ writes per second}$$
  Directly writing 50,000 updates/sec to PostgreSQL causes massive write contention, row-level locks, and WAL exhaustion.
- **Bunny's Kafka Buffering & Batching Strategy**:
  1. The client player posts progress to `/api/v1/progress/heartbeat`.
  2. The stateless API tier writes the event directly into Kafka partition keyed by `user_id:course_id`:
     `Key: "usr_4412:crs_8829" | Payload: {"lecture_id": "lec_01", "position_sec": 420, "ts": 1729254000}`.
  3. **Batch Consumer Worker**:
     - Consumes messages using an in-memory sliding window of 60 seconds.
     - Collapses intermediate updates: If user sends pings for second 10, 20, 30, 40, 50, 60, the consumer **discards the first 5** and keeps only the latest offset (second 60).
     - Writes batched upserts into the `UserProgress` table every 60 seconds:
       ```sql
       INSERT INTO user_progress (user_id, course_id, lecture_id, watched_seconds, is_completed, updated_at)
       VALUES (:user_id, :course_id, :lecture_id, :watched_seconds, :is_completed, NOW())
       ON CONFLICT (user_id, lecture_id) 
       DO UPDATE SET 
         watched_seconds = GREATEST(user_progress.watched_seconds, EXCLUDED.watched_seconds),
         is_completed = (EXCLUDED.watched_seconds >= user_progress.total_duration * 0.9),
         updated_at = NOW();
       ```
  4. Reduces relational DB write throughput from **$50,000\text{ writes/sec}$ down to under $850\text{ writes/sec}$** (a $98.3\%$ write reduction!).

---

### 5.5 Deep Dive 5: Enterprise Multi-Tenancy Architecture (Google/Microsoft Follow-Up)

During Bunny's interviews at Google and Microsoft, interviewers introduced the enterprise B2B pivot: *How do you support corporate clients like Walmart or Microsoft with isolated courses, custom branding, and corporate security?*

#### 1. Tenant Resolution via Dedicated Subdomains:
- **Enterprise URL**: `https://walmart.udemy.com` or `https://microsoft.udemy.com`.
- The Edge Load Balancer inspects the `Host` HTTP header:
  ```
  Host: walmart.udemy.com
  ```
- Extracts `tenant_id = "walmart"`.

#### 2. Enterprise SSO & SAML / OIDC:
- Instead of the public email/password database, requests for `*.udemy.com` are redirected to the company's Identity Provider (Okta, Azure AD / Microsoft Entra ID, PingFederate).
- Upon SAML 2.0 / OIDC assertion validation, an enterprise session token is issued containing:
  ```json
  {
    "sub": "emp_99182",
    "tenant_id": "walmart",
    "roles": ["EMPLOYEE"],
    "allowed_curriculums": ["tech_bootcamp_2026", "leadership"]
  }
  ```

#### 3. Dynamic Landing Page Templating:
- Tenant configurations are stored as structured JSON blobs in a globally replicated cache/database:
  ```json
  {
    "tenant_id": "walmart",
    "branding": {
      "logo_url": "https://cdn.udemy.com/tenants/walmart/logo.svg",
      "primary_color": "#0071DC",
      "custom_banner": "Walmart Tech Learning Academy"
    },
    "landing_template_id": "enterprise_v2",
    "curated_course_collections": ["crs_8829-411a", "crs_1120-77a"]
  }
  ```

#### 4. Data Isolation & Sharding Strategy:
Bunny outlined three multi-tenant isolation tiers:
1. **Pooled Multi-Tenancy (Row-Level Isolation with `tenant_id`)**:
   - Best for SMBs (Small/Medium Businesses). All tenants share database tables with `tenant_id` indexed on every query.
2. **Schema-Level Isolation (Separate DB Schemas per Tenant)**:
   - Shared DB cluster, but dedicated schema (`CREATE SCHEMA walmart;`). Prevents noisy neighbor joins.
3. **Database Sharding per Large Enterprise (Dedicated Shards)**:
   - For massive enterprise accounts (e.g., Walmart with 2M employees), all tenant tables reside on a dedicated partitioned database instance. Routing is handled deterministically at the API Gateway layer via the tenant configuration directory.

---

### 5.6 Deep Dive 6: Instructor Verification & KYC Pipeline

To prevent fraud and copyright infringement, instructors must be validated before receiving course monetization payouts:
1. **Document Submission**: Instructor uploads identity documents (Aadhaar, Passport, Tax ID) through the instructor portal.
2. **Verification State Machine**:
   ```
   [ UNVERIFIED ] ──► [ KYC_SUBMITTED ] ──► [ IN_PROGRESS ] ──► [ VERIFIED ]
                                                   │
                                                   ▼
                                             [ REJECTED ]
   ```
3. **Third-Party API Integration**:
   - For India: Invokes UIDAI / Karza / HyperVerge APIs for Aadhaar/PAN validation.
   - For Global: Invokes Persona / Trulioo / Stripe Identity.
4. **Asynchronous Webhook Callback**:
   - Identity providers process documents asynchronously. Once complete, an inbound webhook hits `POST /api/v1/kyc/webhook`, updating `instructors.verification_status = 'VERIFIED'`.
   - The instructor is now unlocked to set non-zero prices on their course catalog.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you achieve <2 second video startup time with adaptive bitrate streaming?

Video streaming architecture with <2sec startup:

- **Transcoding pipeline**:
  FFmpeg transcodes to 4 resolutions (1080p 5Mbps, 720p 2.5Mbps, 480p 1Mbps, 360p 500Kbps), HLS packaging creates master.m3u8 + variant playlists + 6-sec .ts segments,

- **CDN delivery**:
  CloudFront 200+ edge locations, user routed to nearest (<50ms latency), cache hit rate 95% (segments cached 30 days, manifests 1 hour),

- **Adaptive bitrate**:
  Player loads master.m3u8, measures bandwidth via first segment download, selects resolution (20 Mbps → 1080p, 1 Mbps → 480p), switches mid-stream if bandwidth changes (seamless, no buffering),

- **Startup optimization**:
  DNS prefetch (saves 50ms), HTTP/2 connection reuse (saves 100ms per segment), preload first segment while showing intro, reduced segment size 4-sec for first segment (faster initial load), Breakdown: DNS 50ms + TLS 100ms + Manifest 200ms + First segment 500ms = 850ms total (under <2sec target). Result: Fast startup globally, adaptive to network conditions, scalable to millions of concurrent viewers.

---

### Q2: How do you track progress for millions of students watching videos and taking quizzes?

Event-driven progress tracking via Kafka:

- **Client batching**:
  Video player sends heartbeat every 10 sec with position, batches 3 events (30 sec) before POST /progress/events, reduces API calls 3× (1M students × 6 events/min = 6M → 2M API calls/min),

- **Kafka publishing**:
  Progress Service publishes each event to 'video.progress' topic (100 partitions by user_id hash), responds 200 OK immediately (async, doesn't block video),

- **Consumer processing**:
  10 consumer instances consume from 100 partitions (10K events/sec per consumer = 100K events/sec total capacity), Upsert: INSERT ... ON CONFLICT DO UPDATE SET last_position_sec=140 (resume point), max_position_sec=GREATEST(existing, 140) (farthest watched, never decreases on rewind),

- **Completion**:
  if max_position_sec >= duration × 0.95 → UPDATE status='COMPLETED',

- **Idempotency**:
  Kafka consumer may reprocess events (restart, lag), upsert is idempotent (same event processed twice → same DB state),

- **Scaling**:
  Kafka partitions scale to millions events/day, DB sharded by user_id (user's progress on same shard), consumer auto-scales based on lag. Quiz progress: Submit → auto-grade MCQ → if passed UPDATE progress status='COMPLETED' for quiz lesson. Result: Handles millions of events/day, low latency (<1 sec DB write), resume functionality works reliably, idempotent (safe to retry).

---

### Q3: How do you prevent double enrollment when payment succeeds but webhook is duplicated or lost?

Payment idempotency with multiple safeguards:

- **Client-side idempotency**:
  Client generates enrollmentRequestId (UUID), includes in Stripe payment intent metadata, if user clicks 'Enroll' twice → same ID → Stripe returns existing payment_intent (no duplicate charge),

- **Webhook deduplication**:
  Stripe sends 'payment_intent.succeeded' webhook, Payment Service: Verifies signature (prevents fake webhooks), Updates DB: payment status='COMPLETED', Publishes Kafka 'payment.completed', If webhook sent twice (network retry): Both events reach Kafka (duplicate),

- **Enrollment consumer idempotency**:
  Consumes 'payment.completed', Checks DB: SELECT COUNT(*) WHERE user_id AND course_id, If count > 0 → already enrolled → SKIP (no duplicate enrollment), If count = 0 → INSERT enrollment (first time), UNIQUE constraint on (user_id, course_id) prevents race conditions,

- **Webhook lost scenario**:
  Payment succeeds (Stripe status='succeeded') but webhook delayed/lost, Polling backup: Payment Service polls Stripe API every 5 min for PENDING payments: GET /v1/payment_intents/{id}, If status='succeeded' but local='PENDING' → manually update DB + publish Kafka (same flow as webhook), Catches stuck payments within 5 min,

- **Retry safety**:
  All paths converge: DB check prevents duplicate enrollment, Stripe idempotency prevents duplicate charge, Kafka consumer handles out-of-order/duplicate events. Result: Exactly-once enrollment even with webhook failures/duplicates, payment consistency maintained, no double charging.

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Video Ingestion** | Pre-signed S3 Multipart Uploads | Uploading via Application Microservices | Direct S3 upload prevents application pods from saturating CPU/RAM and network sockets on multi-gigabyte video files. |
| **Video Streaming** | HLS / DASH Multi-bitrate over CDN | Single MP4 file progressive download | HLS enables adaptive bitrate switching under erratic network conditions; CDN edge caching minimizes origin egress latency to $<2\text{s}$. |
| **Catalog Search** | Denormalized Elasticsearch via CDC Aggregator | Real-time PostgreSQL relational `JOIN` queries | Joining 5+ normalized tables at search time bottlenecks relational DB at high QPS; Elasticsearch delivers sub-$50\text{ms}$ faceted search. |
| **Payment & Entitlement**| Transactional Outbox + Debezium CDC | Dual synchronous HTTP calls to Payment and Enrollment DBs | Eliminates distributed transaction failures where money is charged but course entitlement is lost. Guarantees eventual consistency without 2PC locking. |
| **Progress Telemetry** | Kafka Ingest Buffer + 60s Batch Flusher | Direct synchronous writes to PostgreSQL per 10s ping | Collapses $50,000\text{ writes/sec}$ of heartbeat telemetry into $<850\text{ writes/sec}$, preserving database stability and preventing WAL churn. |
| **Enterprise Isolation** | Subdomain routing + Shared/Dedicated Sharding | Entirely separate application deployment per customer | Allows single code-base deployment while maintaining strict security isolation, custom branding, and corporate SSO per enterprise tenant. |
