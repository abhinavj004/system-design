# System Design 01: Design TinyURL / Bitly (URL Shortener)

> **Source**: [Interview with Bunny - Episode 1](https://www.youtube.com/watch?v=Y-BO_4XNw8c)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/01](https://www.interviewwithbunny.com/systemdesign/01)  
> **Topic**: Designing a Highly Scalable, Low-Latency URL Shortening Service (TinyURL / Bitly)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **URL Shortening (Encoding)**: Given a long URL (e.g., `https://example.com/very/long/path?param=123`), the system generates a unique, significantly shorter alias (e.g., `https://tiny.url/aZ9xK2q`).
2. **URL Redirection**: Navigating to the short URL redirects the user to the original long URL with sub-millisecond latency.
3. **Custom Short Alias**: Users can optionally specify a custom short alias (e.g., `tiny.url/my-custom-link`). The system must validate uniqueness.
4. **URL Expiration (TTL)**: URLs can have an expiration timestamp (e.g., 30 days, 90 days, or 2 years). Expired links must no longer redirect.
5. **Analytics & Tracking**: System records metrics on every access (total click count, timestamp, referring domain, user geographical location, device type).

---

### 1.2 Non-Functional Requirements (NFR)
1. **Extreme Low Latency**: Redirection must be blazing fast ($< 5\text{ms}$ on cache hit, $< 20\text{ms}$ on cache miss).
2. **High Availability (99.99%)**: Redirection must never fail; a downed shortener breaks downstream links worldwide.
3. **Consistency vs. Availability (CAP Theorem)**:
   - **Trade-off**: The system heavily favors **Availability (AP)** over Strong Consistency. If a newly created short link takes a few milliseconds to propagate to read replicas, it is acceptable because users typically share the link after creating it.
4. **Read-Heavy System**: Typical URL shorteners experience a $100:1$ Read-to-Write ratio.
5. **Security & Non-Predictability**: Generated short URLs must not be sequential or easily guessable to prevent scraping or enumeration attacks.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Traffic Estimations
- **New URL Creations (Writes)**: 100 Million new URLs created per month.
  $$\text{Write QPS} = \frac{100,000,000}{30 \times 24 \times 3600} \approx 40 \text{ writes/sec} \quad (\text{Peak} \approx 100 \text{ writes/sec})$$
- **URL Redirections (Reads)**: Read-to-Write ratio = $100 : 1$.
  $$\text{Read QPS} = 40 \times 100 = 4,000 \text{ reads/sec} \quad (\text{Peak} \approx 10,000 \text{ reads/sec})$$

#### Storage Estimations (5-Year Projection)
- Each URL record size:
  - `short_url`: 7 bytes
  - `long_url`: 500 bytes (average)
  - `user_id`: 16 bytes (UUID)
  - `created_at`: 8 bytes
  - `expires_at`: 8 bytes
  - Total per record $\approx 550 \text{ bytes}$
- Storage per month: $100\text{M} \times 550 \text{ bytes} \approx 55 \text{ GB / month}$.
- Storage over 5 years: $55 \text{ GB} \times 12 \times 5 \approx 3.3 \text{ TB}$.
- Storage fits comfortably on a modest database cluster.

#### Memory / Caching Estimations (80-20 Rule)
- 20% of the daily URLs generate 80% of the traffic.
- Daily read requests: $4,000 \times 86,400 \approx 345 \text{ Million requests/day}$.
- Cached URLs per day: $20\% \times 345\text{M} \times 550\text{ bytes} \approx 38 \text{ GB RAM}$.
- 38 GB of cache fits in a standard Redis cluster.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|        User        |        |     URLMapping     |        |   ClickAnalytics   |
+--------------------+        +--------------------+        +--------------------+
| id (UUID)          | 1    * | short_url (PK, 7ch)| 1    * | id (UUID)          |
| email (VARCHAR)    |<------>| long_url (TEXT)    |<------>| short_url (FK)     |
| password (VARCHAR) |        | user_id (UUID, FK) |        | clicked_at (TIME)  |
| created_at (TIME)  |        | created_at (TIME)  |        | ip_address (VAR)   |
+--------------------+        | expires_at (TIME)  |        | country_code (VAR) |
                              +--------------------+        | user_agent (TEXT)  |
                                                            +--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Create Short URL
```http
POST /api/v1/urls
Authorization: Bearer <api_token>
Content-Type: application/json

Request Body:
{
  "long_url": "https://www.google.com/search?q=system+design+interview+with+bunny",
  "custom_alias": "bunny-sd-01",       // optional
  "expires_at": "2028-10-18T12:00:00Z"   // optional (defaults to 2 years)
}

Response: 201 Created
{
  "short_url": "https://tiny.url/bunny-sd-01",
  "long_url": "https://www.google.com/search?q=system+design+interview+with+bunny",
  "created_at": "2026-10-18T12:00:00Z",
  "expires_at": "2028-10-18T12:00:00Z"
}
```

---

### 3.2 Redirection Endpoint
```http
GET /{short_url}

Response: 302 Found (or 301 Moved Permanently)
Location: https://www.google.com/search?q=system+design+interview+with+bunny
```
> **Bunny's Key Insight (301 vs. 302 Redirection)**:
> - **301 Moved Permanently**: The browser permanently caches the redirect response. Subsequent clicks go straight to the destination URL without hitting our server. *Pros*: Reduces server load. *Cons*: **Completely blinds the system to click analytics, referrer tracking, and geolocation**.
> - **302 Found (Temporary Redirect)**: The browser is forced to query our server on **every single click**. *Pros*: We can accurately record analytics and update metrics. *Recommendation*: Use **302 Found** (or `307 Temporary Redirect`).

---

### 3.3 Get URL Analytics
```http
GET /api/v1/urls/{short_url}/analytics
Authorization: Bearer <api_token>

Response: 200 OK
{
  "short_url": "bunny-sd-01",
  "total_clicks": 142780,
  "top_countries": { "US": 45000, "IN": 38000, "DE": 12000 },
  "referrers": { "youtube.com": 95000, "direct": 30000, "twitter.com": 17780 }
}
```

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/wbrgibwrbgqwjbrgowqgnbeow.jpg?updatedAt=1769455579711)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 01](https://www.interviewwithbunny.com/systemdesign/01))*

### Component Interaction Topology
```
[ Clients / Browsers ]
         │
         ▼
[ DNS / Cloudflare Edge CDN ] (Static assets, edge rate limiting)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway / Load Balancer                 │
│         (SSL Termination, Rate Limiting, Route Splitting)   │
└─────────────────────────────────────────────────────────────┘
          │                                           │
  (Write: /api/v1/urls)                       (Read: /{short_url})
          │                                           │
          ▼                                           ▼
┌───────────────────────┐                  ┌───────────────────────┐
│ URL Generator Service │                  │ Redirection Service   │
│ (Snowflake Generator) │                  │  (Stateless Workers)  │
└───────────────────────┘                  └───────────────────────┘
          │          ▲                                │          │
          │          │ (Assign Worker ID)             │          │ (Async Click
          │   ┌──────────────┐                        │          │   Events)
          │   │  ZooKeeper   │                        │          ▼
          │   │ Coordinator  │                        │   ┌──────────────┐
          │   └──────────────┘                        │   │ Apache Kafka │
          ▼                                           │   └──────────────┘
┌────────────────────────────────────────┐            │          │
│         Redis Cache Cluster            │◀───────────┘          ▼
│   (LRU Cache for Hot Short Keys)       │            ┌──────────────────────┐
└────────────────────────────────────────┘            │ Analytics Consumer   │
                   │                                  │     Service          │
                   ▼                                  └──────────────────────┘
┌────────────────────────────────────────┐                       │
│    PostgreSQL / MySQL Database         │                       ▼
│   (B-Tree Index on short_url [PK])     │            ┌──────────────────────┐
└────────────────────────────────────────┘            │ ClickHouse / OLAP DB │
                   ▲                                  └──────────────────────┘
                   │
         [ Background Cron Job ] ──(Dual Deletion: Prunes Expired URLs from DB + Redis)
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Evolution of the Unique ID Generation (Bunny's 5 Approaches)

Bunny details the progression of ID generation methods to explain why naive designs fail in production:

#### Approach 1: MD5 Hash Truncation
- Compute `MD5(long_url)` (128-bit hash / 32 hex chars) $\to$ Base62 encode $\to$ take first 7 chars.
- **The Failure**: **Hash Collisions**. Two different URLs can produce the same 7-character prefix. Resolving collisions requires database lookups (`SELECT * FROM urls WHERE short_url = ...`). Each lookup adds 10–15ms of latency, creating massive read/write amplification.

#### Approach 2: Server In-Memory Counter
- Increment an integer counter (`0, 1, 2, ...`) locally in server RAM and encode to Base62.
- **The Failure**: In a distributed microservice setup behind a Load Balancer, Server 1 and Server 2 both start their counters at 0 $\to$ **duplicate IDs generated immediately across instances**.

#### Approach 3: Centralized Redis Counter (`INCR`)
- Point all servers to a centralized Redis instance executing atomic `INCR`.
- **The Failure**: Single Point of Failure (SPOF) and a severe network I/O bottleneck under 100,000+ QPS.

#### Approach 4 & 5 (The Final Design): Twitter Snowflake + Apache ZooKeeper Coordinator
Bunny combines **Twitter Snowflake** with **Apache ZooKeeper** for deterministic, zero-collision ID generation:

```
+-------------------------------------------------------------------------+
|                              64-bit Snowflake ID                         |
+-------------------------------------------------------------------------+
| 1 bit (Unused: 0) | 41 bits (Timestamp in ms) | 10 bits (Worker ID) | 12 bits (Local Counter) |
+-------------------------------------------------------------------------+
```

1. **ZooKeeper ZNodes**: When a `URL Generator Service` instance boots up, it registers an ephemeral zNode with ZooKeeper.
2. **Worker ID Assignment**: ZooKeeper allocates a unique **10-bit Worker ID** (supporting up to 1,024 worker nodes).
3. **64-bit Snowflake Generation**:
   - `41 bits`: Current epoch timestamp in milliseconds ($\approx 69$ years of lifespan).
   - `10 bits`: Worker ID assigned by ZooKeeper.
   - `12 bits`: Local sequence counter (0 to 4,095 IDs per millisecond per worker).
4. **Base62 Encoding**: The resulting 64-bit integer is converted directly into a **7-character Base62 string**:
   $$62^7 \approx 3.52 \text{ Trillion unique combinations}$$
5. **Key Benefits**:
   - **Zero network roundtrips**: Each server generates IDs purely in memory.
   - **Guaranteed globally unique**: Time + Worker ID + Sequence prevents collisions across instances.
   - **Crash Resilient**: If a server dies and restarts, ZooKeeper reassigns or re-registers the worker node, and timestamps prevent counter collisions.

---

### 5.2 Custom Alias Handling
If the user passes `custom_alias`:
1. The service bypasses the Snowflake generator.
2. It executes a point lookup in **Redis** $\to$ if absent, queries **PostgreSQL** (`SELECT 1 FROM urls WHERE short_url = :custom_alias`).
3. If found: Returns `409 Conflict` ("Custom alias already in use").
4. If available: Inserts the custom mapping into PostgreSQL and pre-populates Redis.

---

### 5.3 Database Selection & Indexing Strategy
At `[76:45]`, Bunny gives a pragmatic recommendation on database selection:
- While large-scale blogs frequently suggest Cassandra, Bunny emphasizes:
  > *"You do not have to overcomplicate this. A standard **PostgreSQL or MySQL** database is more than sufficient. Our 5-year data size is only ~3.3 TB, which comfortably fits on a modern relational cluster."*
- **Primary Key & Indexing**:
  ```sql
  CREATE TABLE urls (
      short_url VARCHAR(7) PRIMARY KEY, -- B-Tree index by default
      long_url TEXT NOT NULL,
      user_id UUID,
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
      expires_at TIMESTAMP WITH TIME ZONE
  );
  
  -- Secondary index for user lookups
  CREATE INDEX idx_urls_user_id ON urls(user_id);
  ```
- **Why B-Tree on `short_url` is Critical**:
  - The redirection path performs point lookups (`SELECT long_url FROM urls WHERE short_url = ?`).
  - A primary key B-Tree index provides $O(\log N)$ lookup time, completing in $< 10\text{ms}$ on cache misses.

---

### 5.4 Caching Layer & The Dual-Deletion Cron Job Trap

#### Caching Strategy (Cache-Aside with Redis)
- **Read Path**: Check Redis first (`GET url:{short_url}`). On hit, redirect immediately ($< 2\text{ms}$). On miss, fetch from PostgreSQL, write to Redis with TTL, and redirect.
- **Eviction Policy**: **LRU (Least Recently Used)**.

#### The Dual-Deletion Cron Job Trap
Bunny highlights a subtle production bug regarding link expiration (`[71:58]`):
- A background **Cron Job** runs periodically to purge expired records:
  ```sql
  DELETE FROM urls WHERE expires_at < NOW();
  ```
- **The Trap**: If the Cron Job *only* deletes records from the database, hot links that are actively cached in Redis will **continue resolving indefinitely**!
- **Bunny's Solution**: The cleanup process must enforce **Dual-Deletion**:
  1. The Cron Job identifies expired keys in PostgreSQL.
  2. It deletes the record from PostgreSQL AND issues an explicit `DEL url:{short_url}` to the Redis cluster.
  3. Alternatively, Redis keys must be set with an explicit `EXPIREAT` matching the link's `expires_at` timestamp.

---

### 5.5 Analytics Pipeline (Asynchronous Kafka Streaming)
Writing click metrics synchronously in the redirection path degrades redirect speed:
1. `GET /{short_url}` returns `302 Found` immediately.
2. The Redirection Service publishes an event asynchronously to an **Apache Kafka** topic:
   ```json
   {
     "short_url": "aZ9xK2q",
     "timestamp": 1729251000,
     "ip_address": "192.0.2.1",
     "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0)...",
     "referrer": "https://t.co"
   }
   ```
3. A consumer service streams events into an OLAP data store (**ClickHouse** or **Elasticsearch**) to power user analytics dashboards.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)

### Q1: How do you generate unique short URLs? Compare the three main approaches (ZooKeeper Counter, Hash-Based MD5/SHA-1, and Twitter Snowflake).

In real-world interviews, the unique ID generator is the central algorithmic battleground. Interviewers will push candidates to evaluate the trade-offs between centralized coordination, decentralized throughput, and URL length:

```
+----------------------------------------------------------------------------------------------------+
|                                    ID GENERATION DECISION MATRIX                                   |
+-------------------+----------------------------+-----------------------+---------------------------+
| Feature           | Approach 1: ZK Counter     | Approach 2: Hash (MD5)| Approach 3: Snowflake+ZK  |
+-------------------+----------------------------+-----------------------+---------------------------+
| Uniqueness        | Guaranteed (Atomic Incr)   | Probabilistic (Coll.) | Guaranteed (Time+Worker)  |
| URL Length        | 6–7 chars (Shortest!)      | 7–8 chars (Fixed cut) | 10–11 chars (64-bit int)  |
| Network Latency   | 2–3 ms per ID (Coord.)     | 0 ms local (DB retry) | 0 ms (Local In-Memory)    |
| Deduplication     | No (New URL per request)   | Yes (Deterministic)   | No (New URL per request)  |
| Single Point Fail | Dependent on ZK Quorum     | None (Stateless)      | None (ZK only for boot)   |
| Max Throughput    | ~50K–100K IDs/sec (Cluster)| CPU-bound (Hashing)   | 4,000,000+ IDs/sec        |
+-------------------+----------------------------+-----------------------+---------------------------+
```

#### 1. Approach 1: Distributed Atomic Counter (ZooKeeper / Centralized KGS)
- **Mechanism**: A centralized coordinator (e.g., Apache ZooKeeper) maintains an atomic sequential integer (`/counters/url_counter`). App servers fetch a range of numbers (e.g., 100,000 IDs per lease) and increment locally, converting the integer to Base62 (`[0-9a-zA-Z]`).
  - Mathematical Capacity: $62^6 \approx 56.8 \text{ Billion}$, $62^7 \approx 3.52 \text{ Trillion}$ combinations.
- **Pros**: Yields the absolute shortest possible URLs (6–7 characters).
- **Cons**: Requires centralized coordination. If an application server crashes, its allocated chunk of unused sequential IDs is permanently burned (creating gaps, though acceptable).
- **Industry Usage**: Bitly uses a centralized counter/range allocator.

#### 2. Approach 2: Hash Truncation (MD5 / SHA-1)
- **Mechanism**: Compute `MD5(long_url)` or `SHA1(long_url)` and truncate the first 7 characters.
- **The Collision Trap**: While SHA-1 output space is $2^{160}$, truncating to 7 Base62 characters limits the space to $\approx 3.52 \times 10^{12}$. Under the **Birthday Paradox**, collisions emerge after millions of entries.
- **Collision Resolution Overhead**: Requires querying the database (`SELECT 1 FROM urls WHERE short_url = :hash`). If a collision occurs, the worker must re-hash with a counter salt (`hash(long_url + salt)`), adding 10–50ms database roundtrips per write.
- **Pros**: Purely deterministic deduplication—if multiple users shorten the exact same URL, the system naturally maps them to the same short key, saving up to 10–15% storage.
- **Cons**: High latency write amplification due to collision verification queries.

#### 3. Approach 3: Twitter Snowflake + ZooKeeper Worker ID Allocation (Our Chosen Architecture)
- **Mechanism**: 64-bit self-contained integer generated completely in-memory:
  - `41 bits`: Epoch millisecond timestamp ($\approx 69$ years).
  - `10 bits`: Worker ID (1,024 unique generator nodes, leased via ZooKeeper ephemeral zNodes at server boot).
  - `12 bits`: Sequence counter (4,096 IDs per millisecond per worker).
- **Pros**:
  - **Zero runtime network overhead**: No database or ZooKeeper roundtrip required during URL generation.
  - **Extreme Throughput**: $1,024 \times 4,096 \times 1,000 \approx 4.19 \text{ Million IDs/second}$.
  - **Time-Ordered**: Naturally monotonically increasing for sequential storage indexing.
- **Cons**: A full 64-bit integer encoded in Base62 produces 10–11 characters (`aBc123XyZ`). 
- **Production Optimization**: To achieve a 7-character URL, compact the bit structure by truncating the epoch to 32 bits (seconds granularity) and using a 7-bit worker ID (128 nodes), or use ZooKeeper range-based counter generation if strict 6-character brevity is required.

---

### Q2: How do you handle 1 Billion redirects per day with low latency (<100ms SLA, p99)?

#### 1. Mathematical Traffic Breakdown
- **1 Billion redirects/day**:
  $$\text{Average QPS} = \frac{1,000,000,000}{86,400} \approx 11,574 \text{ req/sec}$$
  $$\text{Peak QPS (10}\times\text{ Traffic Spike)} \approx 115,000 \text{ req/sec}$$

#### 2. Multi-Tiered Low-Latency Serving Architecture

```
[ Client Request ]
       │
       ▼
[ Cloudflare / CDN Edge ] ──────────(Hit: Top 1,000 viral URLs) ──► 302 Found (< 20ms)
       │ (Miss)
       ▼
[ L4/L7 Load Balancers ]
       │
       ▼
[ Redirection Workers ]
       │
       ├─► [ Bloom Filter Guard ] ──(Negative Result) ─────────────► 404 Not Found (< 1ms, No DB/Cache IO)
       │
       ├─► [ Redis Cache Cluster ] ─(90%+ Cache Hit) ──────────────► 302 Found (< 2ms)
       │          │ (Cache Miss ~10%)
       │          ▼
       ├─► [ PostgreSQL Read Replicas (PgBouncer) ] ──────────────► 302 Found (< 15ms)
       │          │
       │          └─► [ Async Redis Backfill (TTL 24h) ]
       │
       └─► [ Kafka Click Stream ] ────────────────────────────────► Async Analytics Pipeline
```

1. **Edge CDN Caching for Viral URLs**:
   - The top 1,000 viral links generate up to 80% of click volume during major events (Super Bowl, breaking news).
   - Route these hot keys to CDN Edge PoPs (Cloudflare Workers / CloudFront) to return cached redirects in $< 20\text{ms}$. To avoid losing analytics telemetry, edge nodes emit access logs asynchronously via HTTP batch beacons to Kafka.
2. **Redis In-Memory Caching (Sub-2ms SLA)**:
   - Cache keys: `url:{short_url}` $\to$ Value: JSON payload `{long_url, expires_at}`.
   - **90%+ Hit Ratio**: 90% of requests are satisfied within 1–2ms directly from Redis memory.
   - Eviction: **LRU (Least Recently Used)** with a default sliding TTL of 24 hours.
3. **Bloom Filter for Negative Cache Penetration Protection**:
   - Attackers or bots brute-forcing non-existent short keys (`tiny.url/zzzzzzz`) will cause continuous cache misses and hammer the database.
   - A 5 MB Bloom filter with 10 million registered short keys provides a $<1\%$ false-positive rate. If the Bloom filter returns `false`, the worker immediately responds with `404 Not Found` in $<0.1\text{ms}$, sparing both Redis and PostgreSQL.
4. **Database Read Replicas & Connection Pooling**:
   - The remaining 10% cache misses ($11,500 \times 10\% \approx 1,150 \text{ QPS}$, peak $\approx 11,500 \text{ QPS}$) query PostgreSQL.
   - 1 Primary + 5 Read Replicas: Each replica handles only $\approx 2,300 \text{ QPS}$ at peak.
   - Primary Key B-Tree Index on `short_url` guarantees $O(\log N)$ lookup in $<5\text{ms}$.
   - **PgBouncer Connection Pooling**: Prevents backend connection starvation by multiplexing thousands of incoming application connections across 100 dedicated database connections.
5. **Decoupled Asynchronous Analytics (Kafka)**:
   - Writing click counts directly to PostgreSQL or Redis on every click would destroy read throughput.
   - Redirection workers fire fire-and-forget events to **Apache Kafka**. Background stream processors (ClickHouse / Flink) aggregate analytics asynchronously, ensuring zero impact on user redirection latency.
6. **The Dual-Deletion Invalidation Trap**:
   - To prevent expired links from returning stale redirects out of Redis indefinitely, the expiration cron job must delete expired keys from PostgreSQL and issue an explicit `DEL` command to Redis (or set Redis key TTLs to synchronize with `expires_at`).

---

### Q3: How do you prevent abuse (spam URLs, malicious links) and implement rate limiting?

URL shorteners are primary targets for spam, phishing, and server-side request forgery (SSRF). A production-grade service enforces defense-in-depth across the write and read paths:

```
                                  MULTI-LAYER ABUSE DEFENSE
                                  
 [ Incoming POST /urls ]
           │
           ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 1: Protocol & Syntax Sanitization                │
 │ - Whitelist: http://, https://                         │
 │ - Block: javascript:, data:, file:// (XSS / LFI)       │
 │ - Max Long URL length: 2,048 chars                     │
 └────────────────────────────────────────────────────────┘
           │
           ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 2: Network & SSRF Defense                        │
 │ - DNS Pre-Resolution: Check destination IP             │
 │ - Reject RFC 1918 Private Subnets:                     │
 │   10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.1 │
 │ - Reject Cloud Metadata: 169.254.169.254               │
 └────────────────────────────────────────────────────────┘
           │
           ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 3: Anti-Phishing & Domain Heuristics             │
 │ - Levenshtein Distance Check against top 10K domains   │
 │   (e.g., 'paypa1.com' vs 'paypal.com' -> Dist = 1)     │
 │ - Anti-Chaining: Block shortening existing shorteners  │
 │   (bit.ly, tiny.url, t.co) to stop redirect laundering │
 └────────────────────────────────────────────────────────┘
           │
           ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 4: Real-time Threat Intelligence                 │
 │ - Google Safe Browsing API / PhishTank integration     │
 │ - Reject known malware, phishing, social engineering   │
 └────────────────────────────────────────────────────────┘
           │
           ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 5: Multi-Tiered Rate Limiting & Bot Scoring      │
 │ - Anonymous IP: Sliding Window (100 URLs / hour / IP)  │
 │ - Authenticated: Daily Quota (1,000 URLs / day / user) │
 │ - Google reCAPTCHA v3: Invisible bot scoring (< 0.5)   │
 └────────────────────────────────────────────────────────┘
           │
           ▼
 [ Allow Short URL Creation ]
```

#### 1. Multi-Tiered Rate Limiting Architecture
- **Anonymous Per-IP Rate Limiting**:
  - Implemented via Redis sliding window counter or token bucket (`rate_limit:ip:{ip_address}`).
  - Limit: 100 URL creations per hour per IP. Returns `429 Too Many Requests` with `Retry-After` header.
- **Per-User Quotas (Authenticated)**:
  - Authenticated accounts receive elevated quotas (e.g., 1,000 URLs/day), encouraging users to register and enabling traceability.
- **Adaptive Bot Scoring (Google reCAPTCHA v3)**:
  - Invisible scoring (0.0 to 1.0). If an IP generates $> 10$ requests per minute or exhibits bot-like characteristics (score $< 0.5$), the client is challenged with an interactive captcha before accepting creation requests.

#### 2. Malicious URL Detection & Phishing Prevention
- **Threat Intelligence Lookup**:
  - Every submitted URL is checked against the **Google Safe Browsing API** or **PhishTank** before short key generation. URLs flagged for malware or social engineering are rejected immediately with `400 Bad Request`.
- **Levenshtein Distance Domain Similarity**:
  - Phishing attacks often replace lookalike characters (e.g., `paypa1.com` instead of `paypal.com`, or `g00gle.com`).
  - Calculate the edit distance against a static dictionary of the top 10,000 popular domains. Any domain with an edit distance of 1 from a protected financial or tech domain is blocked or flagged for manual compliance review.
- **Anti-Chaining / Anti-Laundering**:
  - Malicious actors often daisy-chain shorteners (`tiny.url` $\to$ `bit.ly` $\to$ `ow.ly` $\to$ malware) to evade domain scanners. The system inspects and rejects destination URLs pointing to known public shorteners.

#### 3. Server-Side Request Forgery (SSRF) Defense
- Attackers can exploit link preview generators or internal shortener integrations to probe internal cloud infrastructure.
- Before accepting a URL, the system performs a synchronous DNS lookup and verifies that the resolved IP does **not** belong to:
  - RFC 1918 Private Networks: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
  - Loopback / Localhost: `127.0.0.0/8`.
  - AWS/GCP Metadata IP: `169.254.169.254`.

#### 4. Community Reporting & Quarantine Workflow
- Each redirect landing page provides a "Report Abuse" link.
- If a URL receives $\ge 10$ unique user abuse reports, an automated circuit breaker trips:
  1. Sets `is_flagged = true` in PostgreSQL.
  2. Evicts `url:{short_url}` from Redis.
  3. Subsequent clicks display a warning interstitial page: *"This link has been flagged as suspicious"*, halting victim exposure until a human moderator reviews the link.

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **ID Generation Algorithm** | Twitter Snowflake + ZooKeeper Worker ID | MD5 Hash Truncation | Eliminates hash collision detection loops; purely in-memory generation with zero DB latency. |
| **Database Selection** | Relational SQL (PostgreSQL / MySQL) | NoSQL (Cassandra / DynamoDB) | 3.3 TB fits easily on relational storage; B-Tree index on `short_url` (PK) guarantees fast point lookups. |
| **HTTP Redirection** | `302 Found` / `307 Temporary Redirect` | `301 Moved Permanently` | Forces browsers to hit backend on every click, preserving 100% accurate click analytics. |
| **Caching Layer** | Redis Cluster with LRU | Local App Memory Cache | Centralized cache sharing across stateless web nodes behind the load balancer. |
| **Link Expiration Cleanup** | Dual-Deletion Cron Job (DB + Redis) | Single DB Delete Cron | Prevents expired hot links from persisting and resolving out of Redis cache indefinitely. |
| **Analytics Processing** | Asynchronous Kafka Event Stream | Synchronous DB Write on Click | Decouples analytics logging from the critical sub-5ms redirection latency path. |
| **Negative Query Guard** | Bloom Filter (5 MB for 10M keys) | Direct DB Lookup on Miss | Prevents cache penetration attacks and wasted DB queries for non-existent short keys. |
| **Abuse Defense** | Multi-Tiered (Safe Browsing + SSRF Guard + Levenshtein) | Simple IP Rate Limiting | Comprehensive security against phishing, link chaining, and internal network reconnaissance. |

