# System Design 25: Design News Aggregator Platform (Google News / Feedly / Apple News)

> **Source Reference**: [Dodesimo System Design Notes: Design News Aggregator](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20News%20Aggregator.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](file:///d:/system%20design%20generation/00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG Standard).  
> **Topic**: Architecting a Hyperscale News Ingestion, Deduplication, Clustering, and Feed Delivery System (Google News, Feedly, Apple News).

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Clarifying Questions & Core Scope
- **User Personas**:
  - *Anonymous / Guest Readers*: Browse trending, regional, and category-specific feeds without logging in.
  - *Authenticated Readers*: Follow specific topics, personalize category affinities, bookmark stories, and mute undesired publishers.
  - *News Publishers*: Syndicate articles via standard RSS 2.0, Atom XML, or Publisher News APIs.
  - *System Crawlers / Ingestion Workers*: Periodically discover, poll, parse, sanitize, and cluster syndicated news.
- **Content Boundary**:
  - The platform **does not host full-text article bodies**. To respect copyright law, paywalls, and publisher monetization, the aggregator stores only metadata: headline, 2–3 sentence editorial snippet, publish timestamp, lead thumbnail, author, and canonical outbound URL.
  - When a user clicks a story, they are redirected to the publisher's origin website via an outbound tracking gateway.
- **Availability vs. Consistency (CAP Theorem)**:
  - This is an **AP (Available / Partition Tolerant)** system.
  - Reading the latest news feed must achieve sub-100ms latency globally with 99.99% availability.
  - Eventual consistency is entirely acceptable: if an article crawled at `12:00:00` appears in user feeds at `12:01:30`, the 90-second delay has zero negative business impact.
- **Explicit Out of Scope**:
  - Paywall bypass / full article scraping.
  - In-app comment sections and social discussion forums (handled in Episode 13: Social Media).
  - Video stream encoding / transcoding (handled in Episode 4: Netflix OTT).

---

### 1.2 Functional Requirements (FR)
1. **Multi-Source RSS/Atom Ingestion**: Ingest and poll news metadata continuously from 10,000+ registered publisher RSS/Atom feeds and REST syndication APIs with configurable polling intervals.
2. **Article Deduplication & Story Clustering**: Detect near-duplicate news reports from wire services (e.g., Reuters, Associated Press, Bloomberg) syndicated across hundreds of media outlets, grouping them into a single unified "Story Cluster" with a primary headline and alternative coverage links.
3. **Infinite-Scroll Regional & Categorized Feed**: Serve infinite-scrolling chronological and algorithmic news feeds filtered by geographic region (e.g., US, IN, UK) and topic taxonomy (e.g., Technology, Business, World, Science, Sports).
4. **Resilient Pagination with Zero Drift**: Enable deterministic infinite scrolling via composite cursor pagination, preventing duplicate stories or skipped articles as new content is ingested continuously.
5. **Asset Ingestion & Image CDN Delivery**: Asynchronously download, sanitize, resize, and store lead article thumbnails in private object storage, serving optimized WebP images via CDN to prevent publisher hotlinking failures.
6. **Click Tracking & Outbound Redirection**: Seamlessly route article clicks through a low-latency redirection gateway (`302 Found`) to capture engagement telemetry and CTR metrics before sending the user to the publisher's site.

---

### 1.3 Non-Functional Requirements (NFR)
1. **CAP Posture**: **AP (Availability + Partition Tolerance)**. The read path must never fail due to database cross-region replication lag. Stale feeds are served from edge caches during network partitions.
2. **Latency Budgets**:
   - `GET /api/v1/feed`: P95 $< 80\text{ ms}$, P99 $< 150\text{ ms}$.
   - Outbound Click `302 Found` Redirect: P99 $< 30\text{ ms}$.
   - Ingestion-to-Feed Freshness (Crawl Pipeline): Breaking news available in feed within $< 2\text{ minutes}$.
3. **High Read-to-Write Asymmetry**: Extreme read-heavy skew ($\approx 1,000:1$ read:write ratio).
4. **Politeness & Publisher Protection**: The crawler must adhere to `robots.txt`, respect HTTP `ETag` and `If-Modified-Since` conditional headers, honor RSS `<ttl>` directives, and enforce strict per-domain token bucket rate limiting to prevent DDoS attacks against publisher origins.
5. **Fault Tolerance & Deduplication Idempotency**: Ingesting the same RSS item multiple times must be strictly idempotent with zero storage duplication.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Traffic & Throughput Math
- **Daily Active Users (DAU)**: $100\text{ Million}$ (with peak breaking-event spikes up to $500\text{ Million}$).
- **Feed Refresh Frequency**: Average 10 feed page loads/scrolls per DAU per day.
  $$\text{Total Daily Feed Reads} = 100,000,000 \times 10 = 1,000,000,000\text{ requests/day}$$
  $$\text{Average Feed Read QPS} = \frac{1,000,000,000\text{ requests}}{86,400\text{ seconds}} \approx 11,574\text{ req/sec}$$
  $$\text{Peak Feed Read QPS (3}\times\text{ peak factor)} \approx 35,000\text{ req/sec}$$
  *(During emergency breaking news events with 500M DAU, peak QPS reaches $\approx 175,000\text{ req/sec}$)*.

- **Syndicated Publisher Ingestion Scale**:
  - Registered Publisher RSS Feeds: $10,000\text{ feeds}$.
  - Polling Cadence: Average every $15\text{ minutes}$ (Tier 1 major news outlets every $2\text{ minutes}$; niche blogs every $60\text{ minutes}$).
  - Total Poll Operations/Day:
    $$\text{Polls/Day} = 10,000 \times \frac{1,440\text{ min/day}}{15\text{ min}} = 960,000\text{ crawl polls/day} \approx 11.1\text{ polls/sec}$$
  - HTTP 304 (Not Modified) Ratio: $\approx 75\%$ of polls return zero new articles.
  - New Articles Discovered per Day:
    - Average 50 new articles per publisher per day:
      $$\text{Total Articles/Day} = 10,000 \times 50 = 500,000\text{ articles/day}$$
      $$\text{Article Ingestion Write QPS} = \frac{500,000}{86,400} \approx 5.8\text{ articles/sec} \quad (\text{Peak: } 50\text{ articles/sec})$$

- **Outbound Click Tracking QPS**:
  - Average user clicks on 2 articles per day:
    $$\text{Total Outbound Clicks} = 100,000,000 \times 2 = 200,000,000\text{ clicks/day}$$
    $$\text{Average Outbound Click QPS} = \frac{200,000,000}{86,400} \approx 2,315\text{ clicks/sec} \quad (\text{Peak: } 7,000\text{ clicks/sec})$$

#### Storage Capacity (5-Year Horizon)
- **Article Metadata Size**:
  - `article_id` (16 bytes UUIDv7), `cluster_id` (16 bytes), `publisher_id` (16 bytes), `title` (200 bytes), `summary` (400 bytes), `canonical_url` (256 bytes), `thumbnail_url` (128 bytes), `simhash` (8 bytes), timestamps and flags (64 bytes) $\approx 1.2\text{ KB}$ per record.
  - Daily Metadata Growth:
    $$\text{Daily Metadata} = 500,000 \times 1.2\text{ KB} = 600\text{ MB/day} \approx 219\text{ GB/year}$$
    $$\text{5-Year Metadata Storage} = 219\text{ GB} \times 5 \approx 1.1\text{ TB}$$
  *(Even with B-tree indices and replication factor of 3, total database footprint is $< 5\text{ TB}$, easily hosted in PostgreSQL with sharded read replicas or ScyllaDB/Cassandra)*.

- **Media Thumbnail Storage (S3 + CloudFront CDN)**:
  - $70\%$ of articles contain a valid image thumbnail: $350,000\text{ images/day}$.
  - Converted to optimized WebP format ($640 \times 360$ resolution at 80% quality): $\approx 40\text{ KB}$ per thumbnail.
  - Daily Thumbnail Storage:
    $$\text{Daily Image Storage} = 350,000 \times 40\text{ KB} = 14\text{ GB/day} \approx 5.11\text{ TB/year}$$
    $$\text{5-Year Thumbnail Storage} = 5.11\text{ TB} \times 5 \approx 25.55\text{ TB}$$
  - S3 Object Storage handles 25.5 TB effortlessly at minimal cost.

#### In-Memory Cache Sizing (Redis Cluster)
- **Regional & Categorical Leaderboards**:
  - 50 target countries $\times$ 10 categories (Tech, Politics, Sports, etc.) = $500\text{ distinct feed streams}$.
  - Each feed maintains top 500 recent/trending article references in a Redis Sorted Set (`ZSET`).
  - Size per item: `article_id` (16 bytes) + score (8 bytes) + metadata pointer (40 bytes) $\approx 64\text{ bytes}$.
  - $500\text{ streams} \times 500\text{ items} \times 64\text{ bytes} \approx 16\text{ MB}$ (Extremely light).
- **Personalized Active User Feed Cache**:
  - Cache top 100 precomputed article IDs for top 20% active users (20 Million users):
    $$20,000,000 \times (100 \times 16\text{ bytes}) \approx 32\text{ GB of RAM}$$
  - With Redis memory overhead and $3\times$ replication:
    $$\text{Total Redis Cluster RAM} \approx 32\text{ GB} \times 1.5 \times 3 \approx 144\text{ GB}$$
  - Easily provisioned across a 3-shard Redis Cluster (each master node holding $\approx 48\text{ GB}$).

---

## 2. Step 2: Identification of Core Entities & Data Modeling

### 2.1 Domain Entities Overview

```
+---------------------------------------------------------------------------------------------------+
|                                      CORE DOMAIN ENTITIES                                         |
+---------------------------------------------------------------------------------------------------+

     +------------------------+                     +------------------------+
     |       Publisher        | 1                 * |        Article         |
     +------------------------+-------------------->+------------------------+
     | publisher_id (PK)      |                     | article_id (PK, UUIDv7)|
     | name, domain           |                     | cluster_id (FK)        |
     | rss_url, crawl_cadence |                     | publisher_id (FK)      |
     | etag, last_modified    |                     | title, canonical_url   |
     | reputation_score       |                     | summary, snippet       |
     | region, status         |                     | published_at           |
     +------------------------+                     | simhash (64-bit int)   |
                                                    +------------------------+
                                                                 | 1
                                                                 |
                                      +--------------------------+--------------------------+
                                      | 1                                                   | 1
                                      v                                                     v
                         +------------------------+                            +------------------------+
                         |      StoryCluster      |                            |     MediaThumbnail     |
                         +------------------------+                            +------------------------+
                         | cluster_id (PK, UUIDv7)|                            | thumbnail_id (PK)      |
                         | lead_article_id (FK)   |                            | article_id (FK)        |
                         | cluster_headline       |                            | s3_cdn_url             |
                         | cluster_simhash (64bit)|                            | width, height, format  |
                         | article_count          |                            | byte_size, status      |
                         | first_published_at     |                            +------------------------+
                         +------------------------+
                                      | 1
                                      v *
                         +------------------------+
                         |     ClusterMember      |
                         +------------------------+
                         | cluster_id (PK, FK)    |
                         | article_id (PK, FK)    |
                         | is_lead (BOOLEAN)      |
                         +------------------------+
```

### 2.2 Relational Data Schemas (PostgreSQL / CockroachDB DDL)

```sql
-- 1. Publishers Registry
CREATE TABLE publishers (
    publisher_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(150) NOT NULL,
    domain VARCHAR(255) NOT NULL UNIQUE,
    rss_feed_url VARCHAR(2048) NOT NULL,
    crawl_interval_sec INT NOT NULL DEFAULT 900,
    etag VARCHAR(255),
    last_modified_header VARCHAR(255),
    last_crawled_at TIMESTAMPTZ,
    reputation_score NUMERIC(3, 2) NOT NULL DEFAULT 1.00,
    region VARCHAR(10) NOT NULL DEFAULT 'US',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_publishers_next_crawl ON publishers (last_crawled_at, is_active);

-- 2. Story Clusters (Aggregated wire-service topic grouping)
CREATE TABLE story_clusters (
    cluster_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lead_article_id UUID, -- References articles(article_id) set post insertion
    cluster_headline VARCHAR(500) NOT NULL,
    category_id INT NOT NULL,
    region VARCHAR(10) NOT NULL DEFAULT 'US',
    cluster_simhash BIGINT NOT NULL,
    article_count INT NOT NULL DEFAULT 1,
    first_published_at TIMESTAMPTZ NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL,
    trending_score DOUBLE PRECISION NOT NULL DEFAULT 0.0
);
CREATE INDEX idx_clusters_region_time ON story_clusters (region, category_id, first_published_at DESC);
CREATE INDEX idx_clusters_trending ON story_clusters (region, trending_score DESC);

-- 3. Ingested Articles
CREATE TABLE articles (
    article_id UUID PRIMARY KEY, -- Time-ordered UUIDv7
    cluster_id UUID REFERENCES story_clusters(cluster_id) ON DELETE SET NULL,
    publisher_id UUID NOT NULL REFERENCES publishers(publisher_id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    original_url VARCHAR(2048) NOT NULL,
    canonical_url VARCHAR(2048) NOT NULL UNIQUE,
    summary TEXT NOT NULL,
    content_snippet VARCHAR(1000),
    author VARCHAR(150),
    published_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    category_id INT NOT NULL,
    region VARCHAR(10) NOT NULL DEFAULT 'US',
    simhash BIGINT NOT NULL,
    click_count BIGINT NOT NULL DEFAULT 0,
    is_lead_story BOOLEAN NOT NULL DEFAULT FALSE
);

-- Crucial Composite Index for Drift-Free Cursor-Based Pagination
CREATE INDEX idx_articles_composite_cursor ON articles (region, category_id, published_at DESC, article_id DESC);
CREATE INDEX idx_articles_simhash ON articles (simhash);

-- 4. Media Thumbnails
CREATE TABLE media_thumbnails (
    thumbnail_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id UUID NOT NULL REFERENCES articles(article_id) ON DELETE CASCADE,
    source_url VARCHAR(2048) NOT NULL,
    cdn_s3_url VARCHAR(1024) NOT NULL,
    width INT NOT NULL,
    height INT NOT NULL,
    format VARCHAR(10) NOT NULL DEFAULT 'webp',
    byte_size INT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PROCESSED', -- PENDING, PROCESSED, FAILED
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_thumbnails_article_id ON media_thumbnails (article_id);

-- 5. User Preferences & Affinities
CREATE TABLE user_preferences (
    user_id UUID PRIMARY KEY,
    preferred_region VARCHAR(10) NOT NULL DEFAULT 'US',
    topic_affinities JSONB NOT NULL DEFAULT '{"tech": 0.5, "business": 0.5, "sports": 0.5}',
    muted_publishers UUID[] DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. Step 3: API & Interface Design

All public APIs follow strict RESTful semantics, communicating over HTTPS with JSON payloads. Client authentication is supported via Bearer JWTs; anonymous guests provide an `X-Guest-UUID` header for session-level personalization.

### 3.1 Retrieve Aggregated News Feed (Infinite Scroll)
Fetches an aggregated news feed filtered by geographic region and category with composite cursor pagination.

- **HTTP Method**: `GET`
- **Path**: `/api/v1/feed`
- **Request Headers**:
  ```http
  Authorization: Bearer <jwt_token>  [Optional]
  X-Guest-ID: guest_8fa73b42d619 [Optional]
  Accept: application/json
  ```
- **Query Parameters**:
  - `region` (string, required): ISO-3166 country code (`US`, `IN`, `GB`). Defaults to Geo-IP inference.
  - `category` (string, optional): Content filter (`all`, `technology`, `business`, `science`, `sports`).
  - `limit` (int, optional): Items per batch. Default: `20`, Max: `50`.
  - `cursor` (string, optional): Base64-encoded composite cursor token (`timestamp_ms:article_id`).

- **Response (`200 OK`)**:
  ```json
  {
    "status": "success",
    "data": {
      "items": [
        {
          "article_id": "art_018dc675927b",
          "title": "OpenAI Announces Next-Generation Reasoning Architecture",
          "summary": "The new model demonstrates unprecedented performance across mathematical reasoning and multi-step coding benchmarks.",
          "published_at": "2026-10-18T13:15:00Z",
          "category": "technology",
          "region": "US",
          "publisher": {
            "publisher_id": "pub_techcrunch_01",
            "name": "TechCrunch",
            "domain": "techcrunch.com",
            "reputation_score": 0.95
          },
          "thumbnail": {
            "cdn_url": "https://img.newsplatform.cdn.com/thumbs/2026/10/18/018dc675.webp",
            "width": 640,
            "height": 360,
            "format": "webp"
          },
          "redirect_url": "https://api.newsplatform.com/api/v1/r/art_018dc675927b",
          "cluster": {
            "cluster_id": "cluster_771e89101422",
            "total_coverage_count": 42,
            "more_coverage_url": "/api/v1/clusters/cluster_771e89101422"
          }
        }
      ],
      "pagination": {
        "limit": 20,
        "next_cursor": "MTc2MDc5MzMwMDAwMDphcnRfMDE4ZGM2NzU5Mjdi=",
        "has_more": true
      }
    }
  }
  ```

---

### 3.2 Outbound Click Tracking & Redirection Gateway
Enables privacy-conscious click attribution and CTR ranking while immediately redirecting the client browser to the publisher's origin article.

- **HTTP Method**: `GET`
- **Path**: `/api/v1/r/{article_id}`
- **Query Parameters**:
  - `src` (string, optional): Context source (e.g., `feed`, `trending`, `cluster_modal`).
- **Response**:
  - **Status Code**: `302 Found` *(Explicitly chosen over 301 Moved Permanently)*.
  - **Response Headers**:
    ```http
    HTTP/1.1 302 Found
    Location: https://techcrunch.com/2026/10/18/openai-next-gen-reasoning/
    Cache-Control: no-cache, no-store, must-revalidate
    Set-Cookie: np_session=...; Path=/; HttpOnly; Secure; SameSite=Lax
    ```

> [!IMPORTANT]
> **Why `302 Found` instead of `301 Moved Permanently`?**
> A `301 Moved Permanently` response is aggressively and permanently cached by modern client browsers and ISP forward proxies. Subsequent clicks by the user on that article would bypass our redirection server completely, breaking CTR counting, telemetry recording, and real-time trending algorithms. A `302 Found` forces the browser to hit our redirection gateway on every single click, guaranteeing 100% telemetry capture with sub-30ms redirect latency.

---

### 3.3 Query Story Cluster Full Coverage
Allows users to explore how diverse media outlets across the political or editorial spectrum covered the same underlying breaking news event.

- **HTTP Method**: `GET`
- **Path**: `/api/v1/clusters/{cluster_id}`
- **Response (`200 OK`)**:
  ```json
  {
    "status": "success",
    "cluster_id": "cluster_771e89101422",
    "lead_headline": "Global Central Banks Announce Coordinated Interest Rate Policy",
    "total_publishers": 38,
    "articles": [
      {
        "publisher_name": "Reuters",
        "domain": "reuters.com",
        "title": "Central banks coordinate rate cuts to stabilize liquidity",
        "published_at": "2026-10-18T11:00:00Z",
        "redirect_url": "https://api.newsplatform.com/api/v1/r/art_018dc6501122"
      },
      {
        "publisher_name": "Bloomberg",
        "domain": "bloomberg.com",
        "title": "Fed and ECB issue joint declaration on monetary framework",
        "published_at": "2026-10-18T11:04:00Z",
        "redirect_url": "https://api.newsplatform.com/api/v1/r/art_018dc6513344"
      }
    ]
  }
  ```

---

## 4. Step 4: High-Level Architecture

The news aggregator architecture is decomposed into two cleanly separated planes:
1. **Asynchronous Ingestion & Processing Plane (Write-Heavy, Batch/Stream)**: Crawls external RSS feeds, normalizes XML/JSON, dedupes near-duplicate content, downloads thumbnails, and populates the database and search indices.
2. **Synchronous Feed Delivery & Redirection Plane (Read-Heavy, Latency-Critical)**: Delivers precomputed and cached feeds to 100M+ users with sub-100ms response times.

```
                                      HIGH-LEVEL ARCHITECTURE
===================================================================================================

  [10,000+ News Publishers] (RSS 2.0 / Atom XML / News APIs)
            ▲
            │ HTTP Conditional GET (ETag / If-Modified-Since)
            ▼
  +-----------------------------------------------------------------------------------------------+
  |                        DATA INGESTION & CRAWLING PLANE (ASYNCHRONOUS)                         |
  +-----------------------------------------------------------------------------------------------+
  |                                                                                               |
  |  +--------------------+         +--------------------+         +---------------------------+  |
  |  |   Feed Scheduler   |-------->|   Kafka Topic:     |-------->|   Polite Crawl Workers    |  |
  |  |  (Distributed Cron)|         |   `crawl-tasks`    |         |  (Domain Rate-Limiting)   |  |
  |  +--------------------+         +--------------------+         +---------------------------+  |
  |                                                                              │                |
  |                                                                              ▼ Raw XML/JSON   |
  |  +-----------------------------------------------------------------------------------------+  |
  |  |                              PARSING & NORMALIZATION PIPELINE                           |  |
  |  |  - XML Sanitizer & Content Extractor     - NLP Classifier (Category Tagging)            |  |
  |  +-----------------------------------------------------------------------------------------+  |
  |           │                                                         │                         |
  |           ▼                                                         ▼                         |
  |  +-------------------------------+             +-------------------------------------------+  |
  |  | SimHash Deduplication Worker  |             |      Media Thumbnail Ingestion Worker     |  |
  |  | (64-bit Hamming Distance <=3) |             |  (Download, Sanitize, WebP 640x360 Resize)|  |
  |  +-------------------------------+             +-------------------------------------------+  |
  |           │                                                         │                         |
  +-----------┼---------------------------------------------------------┼-------------------------+
              │                                                         │
              ▼                                                         ▼
     +-------------------+    +--------------------+            +-----------------+
     | PostgreSQL Cluster|    | Elasticsearch Clust|            | AWS S3 Bucket   |
     | (Primary Metadata)|    | (Full-Text Search) |            | (WebP Thumbs)   |
     +-------------------+    +--------------------+            +-----------------+
              ▲                                                         │
              │ Writes Recent Feed Items                                │ CDN Origin
              ▼                                                         ▼
     +-------------------+                                      +-----------------+
     |   Redis Cluster   |                                      | CloudFront CDN  |
     |  (ZSET Feed Cache)|                                      | (Cached Assets) |
     +-------------------+                                      +-----------------+
              ▲                                                         ▲
              │ Feed Read Queries (<10ms)                               │ Thumbnail Fetch
  +-----------┼---------------------------------------------------------┼-------------------------+
  |           │                FEED DELIVERY PLANE (SYNCHRONOUS)        │                         |
  +-----------┼---------------------------------------------------------┼-------------------------+
  |           ▼                                                         │                         |
  |  +--------------------+         +--------------------+              │                         |
  |  | Feed Serving Pods  |<------->|   API Gateway /    |              │                         |
  |  | (Cursor Pagination)|         |   Cloudflare Edge  |              │                         |
  |  +--------------------+         +--------------------+              │                         |
  |                                           ▲                         │                         |
  |  +--------------------+                   │                         │                         |
  |  | 302 Redirection    |-------------------+                         │                         |
  |  | Click Gateway Pods |                                             │                         |
  |  +--------------------+                                             │                         |
  |           │ (Logs Click Event)                                      │                         |
  |           ▼                                                         │                         |
  |  +--------------------+                                             │                         |
  |  | Kafka: `user-clicks|                                             │                         |
  |  +--------------------+                                             │                         |
  +---------------------------------------------------------------------┼-------------------------+
                                                                        │
                                                                        ▼
                                                          [100M+ Mobile / Web Readers]
```

### 4.1 Ingestion & Crawling Flow Breakdown
1. **Feed Scheduler**: A distributed job scheduler (Temporal / Quartz) scans the `publishers` table every 60 seconds, identifying feeds due for polling based on their `crawl_interval_sec`. It publishes `crawl_task` messages to Apache Kafka partitioned by `hash(domain)`.
2. **Polite Crawl Workers**: Stateless worker nodes consume crawl tasks. Each worker verifies per-domain token bucket allowances in Redis. It issues an HTTP `GET` with `If-None-Match: <etag>` and `If-Modified-Since: <last_crawl_time>`. If the publisher returns `304 Not Modified`, execution completes immediately with zero parsing overhead.
3. **Parsing & Normalization**: If `200 OK` is received, the XML parser extracts `<title>`, `<link>`, `<description>`, `<pubDate>`, and `<enclosure>` or `og:image` metadata. A lightweight NLP model classifies the article into standard categories (Tech, Business, etc.).
4. **Deduplication & Clustering Worker**: The worker computes a 64-bit SimHash of the headline and summary. It queries the Locality Sensitive Hashing (LSH) index in Redis. If Hamming distance $\le 3$, it links the article to an existing `StoryCluster`; otherwise, it initiates a new cluster.
5. **Media Worker**: Downloads the source thumbnail, verifies image magic bytes (preventing malicious payloads), resizes it to $640 \times 360$ WebP format, uploads it to AWS S3, and writes the persistent CDN URL to the database.

### 4.2 Feed Serving & Redirection Flow Breakdown
1. **Client Feed Request**: The client issues `GET /api/v1/feed?region=US&category=tech&cursor=<token>`.
2. **Feed Serving Service**: Decodes the composite cursor `(published_at, article_id)`.
3. **Cache-Aside Read**: Checks the Redis Sorted Set `feed:region:US:category:tech`. If cached, it retrieves the slice in $< 5\text{ms}$. If cache misses, it executes an indexed composite query against PostgreSQL, repopulates Redis, and returns the JSON payload.
4. **Click Attribution**: When the user taps a story, the browser requests `/api/v1/r/{article_id}`. The Redirection Gateway looks up the canonical URL in Redis (hot cache), fires a lightweight asynchronous event to Kafka `user-clicks`, and immediately returns HTTP `302 Found` with `Location: <publisher_url>`.

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Politeness Engine & Adaptive Crawling

A news aggregator polling 10,000+ external publishers risks either overwhelming publisher origin servers or getting its crawler IP addresses blacklisted. The crawling engine must implement strict politeness, caching, and rate limiting.

#### 1. Conditional HTTP Headers (ETag & If-Modified-Since)
Publishers adhering to HTTP/1.1 specifications supply `ETag` (entity tags) and `Last-Modified` timestamps. 

```
Aggregator Crawler Worker                                 Publisher Origin Server
        │                                                           │
        │ HTTP GET /rss.xml                                         │
        │ If-None-Match: "5d8c-5f8a0"                               │
        │ If-Modified-Since: Sun, 18 Oct 2026 12:00:00 GMT          │
        │ User-Agent: NewsPlatformBot/2.0 (+https://newsplatform.com/bot)
        │──────────────────────────────────────────────────────────>│
        │                                                           │
        │ Check internal checksum or file mtime                     │
        │ [Content Unchanged]                                       │
        │                                                           │
        │ HTTP/1.1 304 Not Modified                                 │
        │ Date: Sun, 18 Oct 2026 12:15:00 GMT                       │
        │ [Zero Payload Body Transferred]                           │
        │<──────────────────────────────────────────────────────────│
```

- **Network Savings**: By terminating on `304 Not Modified`, over $75\%$ of crawl operations transmit zero XML body bytes, slashing aggregator ingress bandwidth and publisher egress bandwidth by millions of gigabytes per month.

#### 2. Domain-Level Distributed Token Bucket
To ensure that concurrent worker pods never strike the same publisher simultaneously:
- **Redis Token Bucket**: Each publisher domain has a dedicated Redis key: `rate:publisher:{domain}`.
- Max burst: 2 concurrent requests.
- Refill rate: 1 token every 5 seconds.
- If tokens are exhausted, the crawl task is placed back into the Kafka delayed retry queue with exponential backoff.

#### 3. Production Crawl Worker Implementation (Python / AsyncIO)

```python
import asyncio
import httpx
import hashlib
from typing import Optional, Dict, Any

class PoliteCrawlWorker:
    def __init__(self, redis_client, db_pool):
        self.redis = redis_client
        self.db = db_pool
        self.client = httpx.AsyncClient(
            timeout=10.0,
            headers={"User-Agent": "NewsPlatformBot/2.0 (+https://newsplatform.com/bot)"}
        )

    async def acquire_domain_token(self, domain: str) -> bool:
        """Atomic Token Bucket Rate Limiting per Publisher Domain via Redis Lua script"""
        lua_script = """
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local current = tonumber(redis.call('get', key) or "0")
        if current + 1 > limit then
            return 0
        else
            redis.call('incrby', key, 1)
            if current == 0 then
                redis.call('expire', key, 5) -- 5-second cooldown per domain
            end
            return 1
        end
        """
        allowed = await self.redis.eval(lua_script, 1, f"rate:publisher:{domain}", 1)
        return bool(allowed)

    async def crawl_publisher(self, publisher: Dict[str, Any]) -> Optional[str]:
        domain = publisher["domain"]
        feed_url = publisher["rss_feed_url"]
        
        # 1. Enforce Politeness Rate Limiting
        if not await self.acquire_domain_token(domain):
            # Back off and re-queue task
            return "RATE_LIMITED_DEFERRED"

        # 2. Build Conditional Headers
        headers = {}
        if publisher.get("etag"):
            headers["If-None-Match"] = publisher["etag"]
        if publisher.get("last_modified_header"):
            headers["If-Modified-Since"] = publisher["last_modified_header"]

        try:
            # 3. Issue Conditional GET Request
            response = await self.client.get(feed_url, headers=headers)

            if response.status_code == 304:
                # Content not modified, update last_crawled_at and return
                await self.update_crawl_success(publisher["publisher_id"], None, None)
                return "304_NOT_MODIFIED"

            elif response.status_code == 200:
                new_etag = response.headers.get("ETag")
                new_last_modified = response.headers.get("Last-Modified")
                xml_payload = response.text

                # 4. Dispatch XML payload to Processing Queue
                await self.dispatch_to_parsing_pipeline(publisher["publisher_id"], xml_payload)
                await self.update_crawl_success(publisher["publisher_id"], new_etag, new_last_modified)
                return "200_INGESTED"

            elif response.status_code == 429:
                # Back off dynamically if publisher requests throttling
                await self.apply_exponential_backoff(publisher["publisher_id"])
                return "429_BACKOFF"

        except httpx.RequestError as exc:
            # Handle timeout, DNS failure, connection drops gracefully
            await self.record_crawl_failure(publisher["publisher_id"], str(exc))
            return "FAILED"

    async def update_crawl_success(self, publisher_id: str, etag: Optional[str], last_mod: Optional[str]):
        query = """
        UPDATE publishers 
        SET etag = COALESCE($1, etag),
            last_modified_header = COALESCE($2, last_modified_header),
            last_crawled_at = NOW()
        WHERE publisher_id = $3;
        """
        await self.db.execute(query, etag, last_mod, publisher_id)
```

---

### 5.2 Deep Dive 2: Article Deduplication & Story Clustering (SimHash LSH)

When major breaking news occurs (e.g., an earthquake, central bank rate decision, or corporate acquisition), news agencies like **Reuters**, **Associated Press (AP)**, and **Bloomberg** transmit syndicated wires. Hundreds of regional newspapers publish this exact story with identical text or trivial headline variations.

Showing 50 identical articles in a user's feed causes feed exhaustion and poor user engagement. We must deduplicate near-identical articles and cluster related stories.

```
       [Reuters Wire Article]                  [Regional Newspaper A]                 [Regional Newspaper B]
     "Fed cuts rates by 25 bps"              "Fed cuts rates by 25 bps"            "US Central Bank lowers rates"
                 │                                       │                                       │
                 ▼                                       ▼                                       ▼
      SimHash: 0xA4F2...3C10                  SimHash: 0xA4F2...3C11                  SimHash: 0xA4F2...7C10
                 │                                       │                                       │
                 └───────────────────────────────────────┼───────────────────────────────────────┘
                                                         │
                                        Hamming Distance Check (d <= 3)
                                                         │
                                                         ▼
                                          [Identical Story Detected!]
                                                         │
                                                         ▼
                                            +-------------------------+
                                            |   Unified StoryCluster  |
                                            | Headline: Reuters Wire  |
                                            | Secondary Outlets: 42   |
                                            +-------------------------+
```

#### Why Cryptographic Hashes (SHA-256 / MD5) Fail
Cryptographic hash functions are engineered to exhibit the **avalanche effect**: altering a single comma or word in a 5,000-word article changes 50% of the output bits, resulting in completely different hash values. Cryptographic hashing is completely unusable for fuzzy similarity detection.

#### SimHash Algorithm Mechanics
SimHash is a Locality Sensitive Hash (LSH) function where similar text documents produce hashes with minimal Hamming distance (number of differing bits).

```
Step 1: Tokenize & Weight
Text: "Fed cuts rates by 25 bps"
Tokens & Weights (TF-IDF):
- "fed": 4
- "cuts": 3
- "rates": 4
- "25": 2
- "bps": 3

Step 2: 64-bit Hash per Token (MurmurHash3)
hash("fed")   = 1001...01  (weight +4)
hash("cuts")  = 0101...10  (weight +3)
hash("rates") = 1100...11  (weight +4)
hash("25")    = 0011...00  (weight +2)
hash("bps")   = 1010...10  (weight +3)

Step 3: Vector Summation
Initialize 64-dimensional vector V = [0, 0, ..., 0]
For each token hash:
  For bit i in 0..63:
    if bit == 1: V[i] += weight
    if bit == 0: V[i] -= weight

Step 4: Final 64-bit SimHash
For bit i in 0..63:
  if V[i] > 0: SimHash[i] = 1
  else:        SimHash[i] = 0
```

#### Fast Hamming Distance Lookup Table via Block Partitioning
Given millions of articles, comparing a new article's 64-bit SimHash against all existing hashes via $O(N)$ scanning is too slow.

**The Pigeonhole Principle Solution**:
- Two 64-bit numbers have a Hamming distance $d \le 3$.
- Divide the 64-bit SimHash into $k = 4$ chunks of 16 bits each:
  $$\text{SimHash} = [B_1 \mid B_2 \mid B_3 \mid B_4]$$
- If two 64-bit hashes differ by at most 3 bits, **at least one of the 4 chunks must be mathematically identical**!
- We maintain 4 hash tables in Redis / memory:
  - Table 1 keyed by $B_1$
  - Table 2 keyed by $B_2$
  - Table 3 keyed by $B_3$
  - Table 4 keyed by $B_4$
- **Lookup Cost**: We only calculate full 64-bit Hamming distance for candidate articles sharing at least one identical 16-bit block. This reduces comparison candidates from $500,000$ to $< 50$, executing in $< 2\text{ms}$.

#### SimHash & LSH Clustering Implementation (Go)

```go
package clustering

import (
	"encoding/binary"
	"math/bits"
	"strings"
	"github.com/spaolacci/murmur3"
)

type SimHasher struct{}

func (s *SimHasher) ComputeSimHash64(text string) uint64 {
	words := strings.Fields(strings.ToLower(text))
	var v [64]int

	for _, word := range words {
		// Stopword filtration omitted for brevity
		h := murmur3.Sum64([]byte(word))
		weight := 1 // Can be weighted by TF-IDF

		for i := 0; i < 64; i++ {
			bit := (h >> i) & 1
			if bit == 1 {
				v[i] += weight
			} else {
				v[i] -= weight
			}
		}
	}

	var fingerprint uint64 = 0
	for i := 0; i < 64; i++ {
		if v[i] > 0 {
			fingerprint |= (1 << i)
		}
	}
	return fingerprint
}

func HammingDistance(a, b uint64) int {
	return bits.OnesCount64(a ^ b)
}

type ClusterManager struct {
	// 4 Partition Tables: chunk index -> map[16-bit block][]cluster_id
	index [4]map[uint16][]string
}

func (cm *ClusterManager) FindOrCreateCluster(articleID string, simhash uint64, threshold int) string {
	var chunks [4]uint16
	chunks[0] = uint16(simhash >> 48)
	chunks[1] = uint16((simhash >> 32) & 0xFFFF)
	chunks[2] = uint16((simhash >> 16) & 0xFFFF)
	chunks[3] = uint16(simhash & 0xFFFF)

	candidateClusterIDs := make(map[string]bool)

	// Step 1: Query LSH block buckets
	for i := 0; i < 4; i++ {
		if clusterList, exists := cm.index[i][chunks[i]]; exists {
			for _, cid := range clusterList {
				candidateClusterIDs[cid] = true
			}
		}
	}

	// Step 2: Test candidate clusters for Hamming distance <= threshold (3)
	for cid := range candidateClusterIDs {
		clusterSimhash := getClusterSimhashFromDB(cid)
		if HammingDistance(simhash, clusterSimhash) <= threshold {
			attachArticleToCluster(cid, articleID)
			return cid // Grouped with existing wire story
		}
	}

	// Step 3: No duplicate found, create new StoryCluster
	newClusterID := createNewCluster(articleID, simhash)
	for i := 0; i < 4; i++ {
		cm.index[i][chunks[i]] = append(cm.index[i][chunks[i]], newClusterID)
	}
	return newClusterID
}
```

---

### 5.3 Deep Dive 3: Infinite Scroll & Composite Cursor Pagination

Delivering a smooth infinite-scrolling news feed across web and mobile clients requires pagination that is both performant and resilient against real-time data ingestion.

#### The Failure of Offset-Based Pagination (`OFFSET / LIMIT`)
In traditional systems, pagination is implemented as:
```sql
SELECT * FROM articles 
WHERE region = 'US' 
ORDER BY published_at DESC 
LIMIT 20 OFFSET 40;
```

This pattern suffers from two fatal flaws at FAANG scale:

```
Timeline of Events (The Pagination Drift Hazard)
========================================================================================

Time T0: User loads Page 1 (Limit 5, Offset 0).
Articles returned: [A1, A2, A3, A4, A5]
----------------------------------------------------------------------------------------
Time T1: Publishers ingest 3 breaking news stories: [X, Y, Z].
The database table now looks like: [X, Y, Z, A1, A2, A3, A4, A5, A6, A7, ...]
----------------------------------------------------------------------------------------
Time T2: User scrolls down; Client requests Page 2 (Limit 5, Offset 5).
The DB skips the first 5 records ([X, Y, Z, A1, A2]) and returns:
Articles returned: [A3, A4, A5, A6, A7]
========================================================================================
RESULT: The user sees articles A3, A4, and A5 AGAIN! 
Content duplicate rate increases, confusing users and creating broken UI layouts.
```

Furthermore, **performance degrades quadratically** ($O(N)$): at `OFFSET 50000`, the database engine must scan, sort, and discard 50,000 index leaf nodes before reading the 20 requested records.

#### Timestamp-Only Cursors and Collision Vulnerability
Using a timestamp cursor avoids offset scanning:
```sql
SELECT * FROM articles 
WHERE region = 'US' AND published_at < :cursor_timestamp 
ORDER BY published_at DESC LIMIT 20;
```
**The Collision Flaw**: Multiple wire articles crawled in a batch or published by automated bots share the exact same microsecond timestamp. If articles 20, 21, and 22 share timestamp `12:00:00.000000`, article 20 is returned on Page 1. The next request with `published_at < 12:00:00.000000` **skips articles 21 and 22 entirely**, permanently hiding content from readers.

#### The FAANG Solution: Composite Cursor `(published_at, article_id)`
We construct a composite cursor using both the timestamp and the unique, monotonic `article_id` (UUIDv7 or auto-incrementing ID).

```sql
SELECT article_id, title, summary, published_at, publisher_id
FROM articles
WHERE region = :region
  AND (published_at, article_id) < (:cursor_published_at, :cursor_article_id)
ORDER BY published_at DESC, article_id DESC
LIMIT 20;
```

```
                        COMPOSITE B-TREE INDEX SEEK: O(log N)
========================================================================================
Index Structure: (region, published_at DESC, article_id DESC)

Root Node
   │
   ├── Seek directly to ('US', '2026-10-18 13:00:00', '018dc675...') ──> Leaf Node
                                                                           │
                                              Scan precisely next 20 items ├── [Item 1]
                                              Zero table scan.             ├── [Item 2]
                                              Zero skipped items.          └── [Item 20]
========================================================================================
```

- **Encoding the Cursor Token**:
  The composite cursor is serialized into an opaque base64 string before sending to clients:
  ```
  cursor_token = base64_encode(published_at_epoch_ms + ":" + article_id)
  Example: "MTc2MDc5MzMwMDAwMDowMThkYzY3NS05MjdiLTdhYjEtOGUwMS0wOTQ3MmYxMGNhMjM="
  ```
- **Zero Drift Guarantee**: Even if 1,000 new articles are ingested while the user reads page 1, the query seeks directly to the position in the B-Tree after `(:cursor_published_at, :cursor_article_id)`. The user never sees duplicates and never misses an article.

---

### 5.4 Deep Dive 4: Media Thumbnail Ingestion & WebP CDN Optimization

A news aggregator displays high-resolution thumbnails for every story. However, directly linking to publisher image URLs (`<img src="https://publisher.com/photo.jpg">`) fails catastrophically in production.

```
       WHY DIRECT PUBLISHER HOTLINKING FAILS
===================================================================================
1. Hotlinking HTTP 403 Forbidden: Major publishers (NYTimes, WSJ) detect the 
   `Referer` header and block third-party image embedding to protect bandwidth.
2. High Page Latency: Browsers wait on 20 different unoptimized external domains, 
   some with DNS lookups taking > 1,500ms.
3. Uncompressed Raw Images: Publisher images are often 5MB uncompressed PNGs/JPEGs. 
   Loading 20 items consumes 100MB of mobile data.
4. Security & Malware: Vulnerabilities like ImageTragick and malicious tracking pixels 
   leak user IP addresses to third parties.
===================================================================================
```

#### Production Image Processing Pipeline

```
  [RSS Ingestion]
        │
        ▼ (Source Image URL extracted)
  +-------------------------------------------------------------------------------+
  |                        MEDIA THUMBNAIL WORKER POOL                            |
  +-------------------------------------------------------------------------------+
  |                                                                               |
  |  1. Download via Sandboxed HTTP Client (Max size cap: 10 MB, Timeout: 5s)     |
  |                                                                               |
  |  2. Validate MIME Type via Magic Bytes (Reject if not valid JPEG/PNG/WebP)    |
  |                                                                               |
  |  3. Run Libvips / Sharp Transformation Pipeline:                              |
  |     - Strip all EXIF / GPS metadata (Privacy preservation)                    |
  |     - Smart Auto-Crop to 16:9 ratio using saliency detection                  |
  |     - Dual WebP Transcoding:                                                  |
  |         * Feed Thumbnail: 640 x 360 px @ 80% quality (~35 KB)                 |
  |         * Lead Hero Image: 1280 x 720 px @ 85% quality (~85 KB)               |
  |                                                                               |
  |  4. Upload to Amazon S3 Bucket:                                               |
  |     - Path: `s3://news-cdn-thumbs/2026/10/18/{article_id}_640x360.webp`       |
  |     - Headers: `Cache-Control: public, max-age=31536000, immutable`          |
  +-------------------------------------------------------------------------------+
        │
        ▼
  [CloudFront / Cloudflare Edge CDN]
        │
        ▼ (< 15ms Cached Thumbnail Delivery)
  [Mobile App / Web Viewport]
```

- **Storage & Bandwidth Reduction**: Transcoding external multi-megabyte images into uniform $640 \times 360$ WebP files reduces payload sizes by over $88\%$, saving terabytes of mobile bandwidth and ensuring consistent feed rendering speed.

---

### 5.5 Deep Dive 5: Hybrid Personalized Feed Generation Engine

While regional breaking news feeds are shared among all anonymous users in a country, authenticated users expect a personalized blend of breaking headlines and topics matching their personal reading history.

```
                  PERSONALIZED FEED GENERATION: CANDIDATE RETRIEVAL & RANKING
===================================================================================================

 [User Preferences & Reading History]                 [Precomputed Redis Feed Streams]
   - Category Affinities: {Tech: 0.8, Biz: 0.3}         - ZSET: `feed:region:US:breaking`
   - Read History: Bloom Filter (20K items)             - ZSET: `feed:category:tech`
                    │                                   - ZSET: `feed:category:business`
                    │                                                      │
                    └───────────────────────┬──────────────────────────────┘
                                            │
                                            ▼
                          +-----------------------------------+
                          |     Candidate Pool Retrieval      |
                          | (Fetch top 100 candidate articles)|
                          +-----------------------------------+
                                            │
                                            ▼
                          +-----------------------------------+
                          |      Deduplication & Filter       |
                          | (Query Bloom Filter: drop read)   |
                          | (Drop muted publisher IDs)        |
                          +-----------------------------------+
                                            │
                                            ▼
                          +-----------------------------------+
                          |      Lightweight Ranker (LR)      |
                          | Calculate Final Score for each:   |
                          | Score = w1*Affinity + w2*Freshness|
                          |       + w3*CTR_Boost              |
                          +-----------------------------------+
                                            │
                                            ▼
                          [Return Top 20 Ranked Articles]
===================================================================================================
```

#### 1. Real-Time Scoring Formula
For each candidate article $a$ and user $u$, the ranking score $S(u, a)$ is computed in-memory:

$$S(u, a) = \underbrace{w_1 \cdot \text{Affinity}(u, a.\text{category})}_{\text{Personalization Factor}} + \underbrace{w_2 \cdot e^{-\lambda (t_{\text{now}} - a.\text{published\_at})}}_{\text{Exponential Time Decay}} + \underbrace{w_3 \cdot \log(1 + a.\text{click\_count})}_{\text{Global Popularity / CTR}}$$

Where:
- $w_1 = 0.40, w_2 = 0.45, w_3 = 0.15$ (Freshness and relevance take precedence).
- $\lambda = \frac{\ln(2)}{t_{\text{half-life}}}$ where $t_{\text{half-life}} = 6\text{ hours}$ (News value degrades rapidly over time).

#### 2. Deduplicating Already-Read Stories via Redis Bloom Filter
Users dislike seeing articles they have already read. Storing thousands of read article IDs in a relational database table creates expensive SQL queries during feed generation.
- **Solution**: We assign each user a compact **Bloom Filter** in Redis (`BF.ADD` / `BF.EXISTS`).
- Size: A 20,000-bit Bloom filter with 5 hash functions maintains a $< 1\%$ false positive rate for 2,000 read articles while consuming only **2.5 KB of memory per user**.
- Before returning candidate feed items, the service executes a single multi-key pipeline check `BF.MEXISTS user:bloom:{user_id} article_1 article_2 ...` in $< 1\text{ms}$, stripping already-viewed stories.

---

## 6. Step 6: Comparative Architectural Trade-Off Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Feed Ingestion Model** | **Scheduled Polling with Conditional GET** | Real-Time Push (WebSub / PubSubHubbub) | While WebSub push is ideal, $< 5\%$ of worldwide news publishers support it. Polling with HTTP `304 Not Modified` conditional headers guarantees $100\%$ publisher coverage with minimal network overhead. |
| **Pagination Strategy** | **Composite Cursor `(published_at, article_id)`** | Offset-Based Pagination (`OFFSET 1000`) | Offset-based pagination causes severe pagination drift (duplicate/skipped articles) when new stories are crawled continuously, and suffers $O(N)$ query degradation. Composite cursor provides $O(\log N)$ seeks with zero drift. |
| **Deduplication Strategy** | **64-bit SimHash with 4-Block LSH Index** | Cryptographic Hash (MD5 / SHA-256) | Cryptographic hashing fails because a single word variation completely alters the hash. SimHash preserves textual locality, allowing instant Hamming distance grouping ($\le 3$) across syndicated wire services. |
| **Outbound Click Routing** | **HTTP `302 Found` Gateway** | HTTP `301 Moved Permanently` | `301` responses are permanently cached by client browsers, which bypasses subsequent clicks and breaks CTR tracking. `302 Found` forces every click through our telemetry pipeline with $< 30\text{ms}$ redirect latency. |
| **Image Delivery** | **Asynchronous S3 Ingestion + WebP CDN** | Direct Origin Hotlinking (`<img src="...">`) | Publisher hotlinking results in widespread HTTP 403 Forbidden errors, slow third-party DNS queries, and massive uncompressed payloads. Background S3 caching standardizes thumbnails to 40KB WebP files with 99.99% availability. |
| **Feed Storage & Caching** | **Redis Sorted Sets (`ZSET`) + Postgres** | Pure Relational DB Query on every read | Executing composite SQL queries across 100M DAU at 35,000 QPS would require hundreds of database replicas. Caching the top 500 article IDs per region/category in Redis `ZSET` serves reads in $< 5\text{ms}$. |
| **Seen Article Tracking** | **User Redis Bloom Filter** | Relational `read_history` Table JOIN | A relational JOIN against a user's read history table adds 50–100ms latency to every feed request. A 2.5 KB Redis Bloom filter evaluates 20 candidate articles in $< 1\text{ms}$ with zero SQL load. |

---

## 7. Operational Resilience & Failure Modes

### 7.1 Publisher Outages & Network Timeouts
- **Problem**: A news publisher's web server crashes, hangs open TCP connections, or responds with `500 Internal Server Error`.
- **Mitigation**:
  - Crawl workers enforce strict 5-second connection and 10-second read timeouts.
  - Resilience4j-style circuit breakers trip if a domain fails 3 consecutive polls, placing the domain into a 1-hour exponential backoff hibernation before retrying.

### 7.2 Breaking News Thundering Herd
- **Problem**: When a major world event occurs, millions of users open the app simultaneously, causing feed read QPS to spike from $11,000$ to $150,000\text{ req/sec}$.
- **Mitigation**:
  - **Edge CDN Caching**: The anonymous global feed `/api/v1/feed?region=US` is cached at CloudFront edge locations for 30 seconds (`Cache-Control: public, max-age=30, s-maxage=30, stale-while-revalidate=60`).
  - Edge caching absorbs $92\%$ of breaking news read traffic, shielding Redis and PostgreSQL from thundering herd spikes.

### 7.3 Crawl Poisoning & Malicious Content
- **Problem**: A malicious publisher syndicates HTML script injection attacks or links to phishing domains.
- **Mitigation**:
  - HTML content sanitization pipeline strips all `<script>`, `<iframe>`, and `object` tags via an allowlist-only parser (e.g., Bleach).
  - Outbound destination URLs are validated against the Google Safe Browsing API before insertion.
