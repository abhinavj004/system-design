# System Design: News Aggregator Platform (Google News / Feedly / Apple News)

> **Source Reference**: [Dodesimo System Design Notes: Design News Aggregator](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20News%20Aggregator.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG Standard).  
> **Topic**: Architecting a Hyperscale News Ingestion, Deduplication, Regional Clustering, and Sub-50ms Feed Delivery System.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Decouple the write-heavy **Data Collection Service** from the read-heavy, low-latency **Feed Service**. The collector ingests articles across 10,000+ publishers via **Tiered RSS Polling (with ETags)**, **Publisher Webhooks**, and **Intelligent Web Scraping**. Thumbnails are downloaded to **Object Storage (S3) + Global CDN** for standardized delivery. New articles trigger **Change Data Capture (CDC)** to update precomputed **Redis Sorted Sets (`ZSET`)** per region with no TTL (capping top 1,000 via `ZREMRANGEBYRANK`). Feeds are served with **composite cursor pagination `(published_at, article_id)`** or **monotonic ULIDs** to eliminate pagination drift.

| Metric / Dimension | Production Target / Value (Dodesimo Scale) |
| :--- | :--- |
| **Active Users** | **100 Million Daily Active Users (DAU)** (spikes to 500 Million) |
| **Publishers / Sources** | 10,000+ publishers polled based on tiered update frequency |
| **Feed Latency SLA** | **$< 200\text{ ms}$** end-to-end feed load time |
| **Freshness SLA** | Articles appear in user feeds within **$< 30\text{ minutes}$** of publication |
| **Consistency Model** | **Availability over Consistency (AP)**; Eventual consistency |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **The Pagination Drift Trap**:
   - *Trap*: *"Why not just use standard `OFFSET` and `LIMIT` for infinite scrolling?"*
   - *Defense*: In a fast-moving news feed where hundreds of articles are published per minute, `OFFSET 20` shifts the window, causing users to see duplicate stories or skip new breaking items. We use **composite cursor pagination `(published_at, article_id)`** or **monotonic ULIDs**, guaranteeing stable pagination boundaries.
2. **Cache Stampede on Expired Feeds**:
   - *Trap*: *"If you cache regional feeds in Redis with a 30-minute TTL, what happens when it expires?"*
   - *Defense*: A fixed TTL causes a **thundering herd** where millions of concurrent readers slam the database on expiration. Instead, we use **CDC-driven evergreen caches**: Feed generation workers listen to DB change streams and push new articles directly into Redis Sorted Sets (`ZADD`), keeping the top 1,000 articles fresh with zero TTL expiration spikes.
3. **Overwhelming Publisher Servers & IP Bans**:
   - *Defense*: Send `If-None-Match` (HTTP ETag) and `If-Modified-Since` headers so publishers return `304 Not Modified` with zero body when no new articles exist. Combine this with per-domain token bucket rate limiting.

> **Golden Architectural Rule**: Google News does not host full articles; it serves lightweight cards and redirects to origin publishers. Never run heavy live recommendation scoring on the read path—mix precomputed category feeds using lightweight user preference vectors.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Aggregated News Feed**: Users can browse an aggregated feed of news articles sourced from thousands of global and local publishers.
2. **Infinite Scroll (Pagination)**: Users can scroll smoothly and infinitely through their feed without duplicate or skipped articles.
3. **Outbound Article Redirection**: Clicking an article redirects the user to the original publisher's website (Google News does not host full article copy).
4. **Freshness Guarantee**: Breaking and syndicated articles must appear in user feeds within 30 minutes of being published.
5. **Category & Regional Feeds**: Users can browse articles by region (e.g. US, UK, India) and by category (e.g. Technology, Sports, Politics).

---

### 1.2 Non-Functional Requirements (NFR)
1. **High Availability over Strict Consistency (AP)**: The system must remain available even under network partitions. Eventual consistency of 1–2 minutes for new articles is completely acceptable.
2. **Hyperscale Audience**: Must support **100 Million DAU** with surge capacity up to **500 Million users** during major world events.
3. **Sub-200ms Feed Latency**: Feeds must load within $< 200\text{ ms}$ globally across mobile and web.
4. **Publisher Politeness**: Crawlers must respect publisher limits and avoid IP blacklisting.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Read Traffic Estimations
- **Daily Active Users (DAU)**: $100,000,000$ users.
- **Feed Requests per User**: Average 5 visits/day.
- **Total Daily Feed Requests**: $100\text{M} \times 5 = 500,000,000\text{ requests/day}$.
- **Average Read QPS**:
  $$\text{Average Read QPS} = \frac{500,000,000}{86,400\text{ seconds}} \approx 5,800\text{ QPS}$$
- **Peak Read QPS ($3\times$ surge during breaking news)**:
  $$\text{Peak Read QPS} \approx 17,500 - 20,000\text{ QPS}$$

#### Ingestion & Storage Estimations
- **Source Publishers**: $10,000$ active publisher feeds.
- **Articles Published per Day**: $\approx 500,000\text{ articles/day}$ ($5.8\text{ articles/sec}$).
- **Metadata Size per Article**:
  - `title`, `summary`, `canonical_url`, `author`, `publisher_id`, `published_at`, `thumbnail_url` $\approx 2\text{ KB}$.
- **Daily Metadata Storage**:
  $$500,000 \times 2\text{ KB} = 1\text{ GB / day} \approx 365\text{ GB / year}$$
- **Thumbnail Image Storage**:
  - Downloaded lead image resized to 3 responsive variants (small, medium, large) $\approx 200\text{ KB}$ total.
  - Daily Image Storage: $500,000 \times 200\text{ KB} = 100\text{ GB / day} \approx 36.5\text{ TB / year}$ (stored in Object Storage S3).

---

## 2. Step 2: Identification of Core Entities & Data Modeling

```
┌──────────────────┐        1:N        ┌──────────────────┐
│    Publisher     │──────────────────▶│     Article      │
├──────────────────┤                   ├──────────────────┤
│ publisher_id(PK) │                   │ article_id (PK)  │
│ name             │                   │ publisher_id(FK) │
│ website_url      │                   │ title            │
│ feed_url (RSS)   │                   │ summary          │
│ region           │                   │ canonical_url    │
│ polling_priority │                   │ thumbnail_url    │
│ etag / last_mod  │                   │ published_at     │
└──────────────────┘                   │ region           │
                                       │ category         │
                                       │ ulid (monotonic) │
                                       └──────────────────┘
```

### Relational Schema (PostgreSQL)

```sql
CREATE TABLE publishers (
    publisher_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    website_url VARCHAR(512) NOT NULL,
    feed_url VARCHAR(512) NOT NULL,
    region VARCHAR(10) NOT NULL,
    priority VARCHAR(10) DEFAULT 'MEDIUM', -- HIGH (5m), MEDIUM (30m), LOW (3h)
    etag VARCHAR(128),
    last_modified TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE articles (
    article_id VARCHAR(64) PRIMARY KEY,
    publisher_id VARCHAR(64) NOT NULL REFERENCES publishers(publisher_id),
    title VARCHAR(512) NOT NULL,
    summary TEXT,
    canonical_url VARCHAR(1024) NOT NULL,
    thumbnail_url VARCHAR(512),
    published_at TIMESTAMPTZ NOT NULL,
    region VARCHAR(10) NOT NULL,
    category VARCHAR(50) NOT NULL,
    ulid VARCHAR(26) UNIQUE NOT NULL,      -- Universally Unique Lexicographically Sortable Identifier
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Composite index for fast regional feed querying and cursor pagination
CREATE INDEX idx_articles_region_published ON articles(region, published_at DESC, article_id DESC);
CREATE INDEX idx_articles_region_category ON articles(region, category, published_at DESC);
```

---

## 3. Step 3: API & Interface Design

### 3.1 Get Aggregated News Feed (Cursor-based)
```http
GET /api/v1/feed?region=US&category=technology&limit=20&cursor=2026-10-24T12:00:00Z_art_88192
Accept: application/json

Response: 200 OK
{
  "articles": [
    {
      "article_id": "art_88192",
      "title": "Breakthrough in Fusion Energy Announced",
      "summary": "Scientists at the National Laboratory achieved net energy gain...",
      "publisher": {
        "id": "pub_01",
        "name": "Tech Daily"
      },
      "canonical_url": "https://techdaily.com/breakthrough-fusion",
      "thumbnail_url": "https://cdn.news.com/thumbs/art_88192_m.webp",
      "published_at": "2026-10-24T11:58:10Z"
    }
  ],
  "next_cursor": "2026-10-24T11:45:00Z_art_88170",
  "has_more": true
}
```

### 3.2 Outbound Click Tracking & Redirection
```http
GET /api/v1/articles/{article_id}/click?user_id=usr_992

Response: 302 Found
Location: https://techdaily.com/breakthrough-fusion
```
*(The server logs the click analytics event to Kafka asynchronously and issues an HTTP 302 redirect directly to the publisher's site).*

### 3.3 Publisher Ingestion Webhook (Breaking News)
```http
POST /api/v1/webhooks/article-published
Authorization: Bearer <publisher_api_token>
Content-Type: application/json
{
  "title": "Breaking: Market Rally Reaches Record High",
  "summary": "Global indices surge following economic policy announcement...",
  "canonical_url": "https://financialtimes.example.com/record-high",
  "image_url": "https://financialtimes.example.com/images/hero.jpg",
  "published_at": "2026-10-24T12:05:00Z",
  "region": "US",
  "category": "business"
}

Response: 202 Accepted
```

---

## 4. Step 4: High-Level Architecture (HLD)

### Component Interaction Topology

```
[ News Publishers ] (RSS / Webhooks / Web Scraping)
         │
         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                Data Collection Service                                 │
│  (Tiered RSS Scheduler + Webhook Receiver + Scraper Fallback + Politeness ETag Engine)  │
└────────────────────────────────────────────────────────────────────────────────────────┘
         │                                                        │
         │ Original Thumbnails                                    │ Articles & Metadata
         ▼                                                        ▼
┌─────────────────────────┐                             ┌─────────────────────────┐
│ Image Processing Worker │                             │  Ingestion Kafka Topic  │
│  (Resizes to 3 sizes)   │                             └─────────────────────────┘
└─────────────────────────┘                                           │
         │                                                            │
         ▼                                                            ▼
┌─────────────────────────┐                             ┌─────────────────────────┐
│ Object Storage (S3/GCS) │                             │   PostgreSQL Database   │
└─────────────────────────┘                             │  (Master Data Store)    │
         │                                              └─────────────────────────┘
         ▼                                                            │
┌─────────────────────────┐                                           ▼ (Change Data Capture)
│   Global CDN Edge       │                             ┌─────────────────────────┐
│ (Low-latency Images)    │                             │  Feed Generation Worker │
└─────────────────────────┘                             └─────────────────────────┘
                                                                      │
                                                Precompute Top Feeds  │ (ZADD feed:{region})
                                                                      ▼
                                                        ┌─────────────────────────┐
                                                        │   Redis Sorted Sets     │
                                                        │  (Evergreen Top Feeds)  │
                                                        └─────────────────────────┘
                                                                      ▲
                                                                      │ Read Feed (<20ms)
                                                        ┌─────────────────────────┐
[ Mobile & Web Clients ] ──▶ [ API Gateway / CDN ] ──▶  │      Feed Service       │
                                                        │ (Regional / Stateless)  │
                                                        └─────────────────────────┘
```

---

### 4.1 Why Separate Feed Service from Data Collection Service?

In an interview, justify splitting the architecture into two dedicated services:
1. **Read-Heavy vs. Write-Heavy Workloads**: The Feed Service serves **500 Million read requests/day** with strict sub-200ms latency SLAs. The Data Collection Service performs heavy outbound HTTP crawling, XML parsing, HTML scraping, and image downloads.
2. **Asynchronous & Variable Update Frequencies**: News crawling happens periodically (every 5 to 30 minutes) or in batch bursts, while feed reading is continuous and user-driven.
3. **Fault Isolation**: If an external publisher's RSS feed is slow or malformed, the Data Collection Service absorbs the delay without impacting user feed load speeds.

---

### 4.2 Step-by-Step Architectural Evolution (Whiteboard Script)

#### Phase 1: The Naive Baseline MVP (Monolith + Offset Pagination)
- **Whiteboard**: `Client` ──► `API Server` ──► `PostgreSQL Database`.
- **Explanation**: *"The API server runs a periodic cron job polling RSS feeds, saves them to PostgreSQL, and serves feeds via `SELECT * FROM articles WHERE region = 'US' ORDER BY published_at DESC LIMIT 20 OFFSET 20`."*
- **Bottlenecks**:
  1. *Pagination Drift*: As new articles are inserted, offset numbers shift, causing users to see duplicate stories or miss new articles.
  2. *Database CPU Collapse*: 20,000 concurrent feed queries scanning a growing relational database bring the database down.

#### Phase 2: Decouple Ingestion from Feed Delivery
- **Whiteboard**: Split into **Data Collection Service** and **Feed Service**, separated by a primary database.
- **Explanation**: *"We isolate heavy outbound crawling from user-facing feed queries. Feed Service scales horizontally to handle read traffic."*

#### Phase 3: Implement Stable Cursor-Based Pagination
- **Whiteboard**: Replace `OFFSET` with a **Composite Cursor `(published_at, article_id)`** or **Monotonic ULID**.
- **Explanation**: *"We eliminate pagination drift by querying relative to a fixed cursor: `WHERE (published_at, article_id) < (:cur_time, :cur_id)`. This provides deterministic, $O(1)$ indexed reads."*

#### Phase 4: Precompute Regional Feeds in Redis Sorted Sets via CDC
- **Whiteboard**: Add **Redis Sorted Sets (`ZSET`)** populated by a **Feed Generation Worker** using **Change Data Capture (CDC)**.
- **Explanation**: *"To achieve sub-50ms reads without cache stampedes, we eliminate TTLs on regional feeds. When new articles are inserted into the DB, CDC workers push them into Redis sorted sets (`ZADD score=published_at`). `ZREMRANGEBYRANK` keeps only the top 1,000 articles."*

#### Phase 5: Multi-Tiered Ingestion (RSS + Webhooks + Scrapers)
- **Whiteboard**: Add **Publisher Webhooks** for instant breaking news and **Intelligent Web Scraping** as a fallback for publishers without RSS feeds.
- **Explanation**: *"To guarantee articles appear within 30 minutes, major partners push news immediately via webhooks. High-priority feeds are polled every 5 minutes with HTTP ETags."*

#### Phase 6: Regional Deployment & Hybrid Dynamic Personalization
- **Whiteboard**: Deploy Feed Services and Redis Read Replicas across multi-region datacenters (US, EU, APAC).
- **Explanation**: *"Since news consumption is naturally regional, deploying near the user cuts latency to $<50\text{ ms}$. User personalization blends precomputed category feeds via lightweight preference vectors in memory."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: Pagination Consistency (Offset vs. Cursors vs. ULIDs)

How do we implement infinite scrolling without duplicate or skipped articles?

#### Approach 1 (Bad): Offset-Based Pagination
```sql
SELECT * FROM articles WHERE region = 'US' ORDER BY published_at DESC LIMIT 20 OFFSET 40;
```
- **The Fatal Flaw (Pagination Drift)**:
  - User reads Page 1 (Articles 1–20).
  - While reading, 5 new breaking articles are ingested.
  - User scrolls to Page 2 (`OFFSET 20`).
  - Because 5 new rows were prepended, articles #16 to #20 from Page 1 are now pushed into Page 2!
  - **Result**: The user sees duplicate stories. Conversely, if articles are deleted, stories are skipped entirely.

#### Approach 2 (Good): Timestamp-Based Cursor
```sql
SELECT * FROM articles WHERE region = 'US' AND published_at < :last_seen_timestamp ORDER BY published_at DESC LIMIT 20;
```
- **Limitation**: Multiple syndicated articles often share the **exact same publication millisecond**. If 3 articles share `12:00:00.000` and the page boundary cuts across them, the query misses articles sharing the boundary timestamp.

#### Approach 3 (Better): Composite Cursor `(published_at, article_id)`
```sql
SELECT * FROM articles 
WHERE region = 'US' 
  AND (published_at, article_id) < (:cursor_timestamp, :cursor_id) 
ORDER BY published_at DESC, article_id DESC 
LIMIT 20;
```
- **Why it Works**: The `article_id` acts as a deterministic tie-breaker. Backed by a composite index `(region, published_at DESC, article_id DESC)`, this executes in $O(\log N)$ time with zero pagination drift.

#### Approach 4 (Champion / Modern Standard): Monotonic ULID Cursors
- Use **ULIDs (Universally Unique Lexicographically Sortable Identifiers)**:
  - 48-bit timestamp + 80-bit randomness = 128 bits.
  - Naturally sortable chronologically: newer articles always have a higher lexicographical ULID.
  - Query simplifies to:
    ```sql
    SELECT * FROM articles WHERE region = 'US' AND ulid < :cursor_ulid ORDER BY ulid DESC LIMIT 20;
    ```
  - Single-column index on `(region, ulid DESC)` maximizes database index cache locality.

---

### 5.2 Deep Dive 2: Low-Latency Feed Delivery (Redis TTL vs. Evergreen CDC Feeds)

#### Problem with Simple Redis Cache with TTL (30 Mins):
- Storing `feed:US` with a 30-minute expiration causes two fatal problems:
  1. *Cache Inconsistency*: Breaking news published at minute 2 is hidden until minute 30.
  2. *Thundering Herd*: At minute 30, the key expires. 50,000 concurrent mobile apps miss the cache and hit PostgreSQL simultaneously, crashing the database.

#### The Production Solution: Real-Time Evergreen Caches via CDC
- Regional feeds are maintained as **Redis Sorted Sets (`ZSET`)** with **NO TTL**:
  - Key: `feed:region:US`
  - Score: Epoch timestamp (`published_at`)
  - Member: `article_id`

```
  [ PostgreSQL Database ] ──(WAL / Debezium CDC)──▶ [ Feed Generation Worker ]
                                                            │
                                                            │ ZADD feed:region:US <timestamp> <article_id>
                                                            │ ZREMRANGEBYRANK feed:region:US 0 -1001
                                                            ▼
                                                [ Redis Sorted Set (ZSET) ]
                                                (Always keeps top 1,000 IDs)
```

1. When a new article is committed to PostgreSQL, **Change Data Capture (CDC)** triggers the Feed Generation Worker.
2. The worker pushes the article ID to the regional sorted set:
   ```text
   ZADD feed:region:US 1761307200 "art_88192"
   ```
3. To bound Redis RAM usage, trim the set to maintain only the top 1,000 most recent articles:
   ```text
   ZREMRANGEBYRANK feed:region:US 0 -1001
   ```
4. **Reading the Feed**:
   ```text
   ZREVRANGEBYSCORE feed:region:US <cursor_score> -inf LIMIT 0 20
   ```
   Followed by an `MGET` to fetch article metadata cards from a Redis Hash table. Read latency drops to $< 5\text{ ms}$!

---

### 5.3 Deep Dive 3: Guaranteeing Article Freshness (< 30 Mins)

How do we ingest breaking news across 10,000 publishers within 30 minutes without overwhelming servers?

#### 1. Tiered Polling Strategy + HTTP ETags:
- Categorize publishers into 3 tiers:
  - **Tier 1 (High Priority - AP, Reuters, BBC)**: Polled every **5 minutes**.
  - **Tier 2 (Medium Priority - Regional dailies)**: Polled every **30 minutes**.
  - **Tier 3 (Low Priority - Niche blogs)**: Polled every **2–3 hours**.
- **HTTP Conditional GET (ETags & Last-Modified)**:
  - Crawler sends: `If-None-Match: "686897696a"` and `If-Modified-Since: Sat, 24 Oct 2026 12:00:00 GMT`.
  - If no new stories were posted, the publisher returns **`304 Not Modified` with 0 bytes body**.
  - **Result**: $85\%$ of polling requests consume virtually zero publisher bandwidth and compute.

#### 2. Publisher Webhooks (`POST /webhooks/article-published`):
- For major media partners, provide an authenticated webhook API. When an editor hits "Publish" in WordPress or their CMS, a webhook notifies our ingestion pipeline in $< 2\text{ seconds}$.

#### 3. Intelligent Web Scraping (Fallback):
- For publishers without RSS or Webhooks:
  - Scraper monitors target category pages (e.g. `homepage`, `/world`).
  - Compares page hashes against a URL fingerprint database.
  - Extracts title, snippet, and canonical link using CSS selectors (`.story-headline`, `.article-summary`).

---

### 5.4 Deep Dive 4: Thumbnail Image Architecture

Why download our own copies of publisher images to Object Storage (S3) + CDN?
1. **Broken Links & Hotlinking Restrictions**: Publishers frequently change CDN paths, enforce anti-hotlinking referrer checks, or delete older images.
2. **Speed & Reliability**: Publisher origin servers can take 2–5 seconds to serve an image. Serving images from our own global CDN ensures sub-50ms thumbnail loads.
3. **Responsive Size Standardization (`srcset`)**:
   - Original publisher images can be 5MB uncompressed JPEGs.
   - An asynchronous image worker converts each lead image into modern WebP/AVIF across 3 fixed resolutions:
     - `small` (150x150, for mobile feed lists) $\approx 15\text{ KB}$.
     - `medium` (400x250, for tablet cards) $\approx 45\text{ KB}$.
     - `large` (800x450, for desktop hero cards) $\approx 120\text{ KB}$.

---

### 5.5 Deep Dive 5: Breaking News & Traffic Spikes

During global breaking news, feed read traffic spikes from 5,800 QPS to 20,000+ QPS:
1. **Geographic Distribution**: Deploy Feed Services across edge regions (US-East, US-West, EU-Central, APAC). 90% of news consumption is regional, so requests terminate locally.
2. **Redis Read Replicas**:
   - The Redis Primary handles writes from CDC workers.
   - Read requests are distributed across a pool of **Redis Read Replicas** managed by Redis Sentinel.
   - Adding 3 read replicas scales throughput to $300,000+\text{ QPS}$ without database hits.
3. **Stateless Autoscaling**: Feed Services run in Kubernetes pods with Horizontal Pod Autoscaling (HPA) scaling on CPU $> 65\%$.

---

### 5.6 Deep Dive 6: Category-Based News Filtering

Users want feeds filtered by category (e.g., Technology, Sports, Politics).

#### Approaches Evaluated:
1. **Bad (Database Query Filtering)**:
   - Querying PostgreSQL with `WHERE region = 'US' AND category = 'sports'` forces queries to hit disk and invalidates simple caches.
2. **Better (Separate Redis Sorted Set per Category)**:
   - Maintain `feed:US:sports`, `feed:US:politics`, etc.
   - *Issue*: 25 categories $\times$ 10 regions = 250 sorted sets. Updating 250 sets per article complicates cache invalidation and multiplies Redis memory usage.
3. **Best (In-Memory Filter on Regional Cache)**:
   - The top 1,000 regional articles are already cached in Redis memory.
   - The Feed Service retrieves the top 200 article metadata cards from the regional set in a single `MGET` and filters by `category == 'sports'` **in application memory** in $< 1\text{ ms}$.
   - Zero additional Redis sets required; zero memory duplication.

---

### 5.7 Deep Dive 7: Personalization Strategies

How do we personalize feeds for 100 Million users without destroying latency?

```
               [ User Profile: 60% Tech, 30% Business, 10% Politics ]
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           ▼ (Take 12 items)            ▼ (Take 6 items)             ▼ (Take 2 items)
  [ feed:region:US:tech ]     [ feed:region:US:business ]   [ feed:region:US:politics ]
           │                            │                            │
           └────────────────────────────┼────────────────────────────┘
                                        ▼
                  [ Dynamic In-Memory Mixer & Deduplicator ]
                                        │
                                        ▼
                      [ Final 20 Personalized Articles ]
```

1. **Bad (Real-Time ML Scoring on the Read Path)**:
   - Scoring 10,000 articles through a neural network on every user feed request adds $> 500\text{ ms}$ latency and costs millions in GPU compute.
2. **Good (Dedicated Precomputed Cache per User)**:
   - Precompute and store a separate Redis sorted set for every single user.
   - *Fatal Flaw*: 100M active users $\times$ 200 stories = billions of entries, exploding RAM costs while serving stale content to inactive users.
3. **Great (Dynamic Hybrid Personalization via Preference Vectors)**:
   - Maintain a lightweight **Preference Vector** per user (e.g. `Tech: 0.60`, `Business: 0.30`, `Politics: 0.10`).
   - When the user requests their personalized feed:
     - The Feed Service fetches 12 articles from `feed:tech`, 6 from `feed:business`, and 2 from `feed:politics`.
     - Blends, deduplicates, and sorts them in memory.
   - Combines sub-20ms speed with highly effective topic personalization at near-zero incremental RAM cost.

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Pagination Strategy** | Composite Cursor `(published_at, id)` / ULID | Offset-based (`OFFSET / LIMIT`) | Offsets cause pagination drift (duplicates and skipped stories) in dynamic news feeds. Cursors ensure deterministic ordering. |
| **Feed Caching Model** | Evergreen Redis Sorted Sets via CDC (No TTL) | Time-to-Live (TTL) Cache | TTL caches cause thundering herds on expiry and hide breaking news. CDC keeps top 1,000 articles fresh continuously. |
| **Ingestion Pipeline** | Tiered RSS + HTTP ETags + Webhooks + Scrapers | Pure fixed-interval web scraping | ETags eliminate 85% of redundant bandwidth; webhooks provide sub-second breaking news alerts. |
| **Image Hosting** | Downloaded to S3 + Global CDN (`srcset`) | Direct hotlinking to publisher images | Prevents broken links, bypasses publisher referrer blocks, and delivers standardized responsive thumbnails. |
| **Category Delivery** | In-memory filtering from top regional cache | 250 separate Redis sets per category | Avoids multiplying Redis RAM costs by 10x; in-memory filtering takes $< 1\text{ ms}$. |
| **Feed Personalization**| Dynamic In-Memory Preference Vector Mixing | Real-time Deep Neural Net Scoring | Live scoring adds $>500\text{ ms}$ latency; vector mixing delivers personalized feeds in $<30\text{ ms}$. |

---

### Top 4 Interview Follow-Up Questions (Rapid-Fire Defenses)

### Q1: How do you handle syndicated news where 50 outlets publish the exact same AP/Reuters story?
- **Answer**: Pass article text through **64-bit Locality-Sensitive Hashing (SimHash)** during ingestion. If an incoming article has a Hamming distance $\le 3$ compared to an already ingested story, group them into a single cluster card ("Covered by 50 sources") rather than cluttering the feed with duplicate stories.

### Q2: What happens if a publisher's RSS feed is temporarily unreachable or returning HTTP 500?
- **Answer**: Implement an **exponential backoff retry strategy** with jitter in the Data Collection Service. The publisher is temporarily downgraded in priority. Previously ingested articles from that publisher remain intact in the database and cache.

### Q3: How do you prevent crawlers from being banned by anti-bot systems like Cloudflare?
- **Answer**: Register official crawler User-Agents with publishers, respect `robots.txt`, enforce per-domain rate limits (max 1 req every 2 seconds per domain), and distribute crawler outbound IPs across multiple subnets.

### Q4: How do you handle "breaking news" banners on the homepage?
- **Answer**: When an article is tagged as `is_breaking: true` by editors or high-confidence clustering, publish an event to **Redis Pub/Sub**. The event pushes to an in-memory application cache on all Feed Service pods, updating the homepage banner globally within $< 5\text{ seconds}$.
