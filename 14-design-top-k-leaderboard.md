# System Design 14: Design Top K Leaderboard / Real-Time Ranking System

> **Source**: [Interview with Bunny - Episode 14](https://www.youtube.com/watch?v=KOvDWEGVXec)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/14](https://www.interviewwithbunny.com/systemdesign/14)  
> **Topic**: Designing a Real-Time Top K Leaderboard & Ranking System (YouTube Trending, Spotify Charts, Global Gaming Leaderboard)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Problem Scope & Core Mental Model
In distributed systems, the **Top K Leaderboard / Ranking System** solves the problem of dynamically finding the highest-ranked $K$ items (e.g., top 10 trending YouTube videos, top 50 Spotify songs, top 100 gaming players) out of millions or billions of candidates based on dynamic dimensions (views, likes, player scores, upvotes).

This problem is the production-scale distributed systems analogue of the classic algorithmic problem: **LeetCode 347 (Top K Frequent Elements)**. While LeetCode solves this in a single process using a Min-Heap or TreeMap in $O(N \log K)$ time, production system design requires solving this under massive write ingestion (1M+ events/sec), strict sub-100ms latency budgets, and multi-tenant dimensional slicing (region, genre, time window).

---

### 1.2 Functional Requirements (FR)
1. **High-Throughput Score / Metric Ingestion**: The system must ingest events continuously (e.g., game score increment, video view, song listen, post like) from users and external services.
2. **Top K Leaderboard Retrieval**: Clients can query the top $K$ items ($1 \le K \le 10,000$) for a specific leaderboard (e.g., game tournament, country-specific music chart).
3. **Time-Window Filtering**: Support slicing rankings over flexible temporal windows:
   - Real-time rolling windows (last 1 hour, last 24 hours).
   - Fixed historical windows (daily, weekly, monthly, last 90 days, all-time).
4. **Surrounding Rank Lookup ("Players Around Me")**: Given a user/item ID, return the current absolute rank along with $K$ adjacent peers (e.g., user is rank 50; return ranks 48 to 52).
5. **Real-Time Live Updates (Soft/Near Real-Time)**: For live gaming competitions, client devices can receive live leaderboard score updates via persistent bi-directional connections (WebSockets).

---

### 1.3 Non-Functional Requirements (NFR)
1. **Ultra-Low Read Latency**: Top $K$ leaderboard queries must return in **$< 100\text{ms}$** (p99) to power fast home screens and gaming overlays.
2. **High Ingestion Write Throughput**: Ingestion pipeline must handle peak loads of **1,000,000 requests per second (RPS)** without dropping events.
3. **Availability over Consistency (CAP Theorem - AP System)**:
   - The system strictly prioritizes **High Availability (AP)**.
   - Returning a leaderboard that is 2–5 seconds stale is completely acceptable; returning HTTP 500 or failing to render a homepage is catastrophic.
   - The system guarantees **Eventual Consistency** (soft real-time updates within $100\text{ms} - 500\text{ms}$).
4. **Accuracy vs. Probabilistic Computation**:
   - The primary design guarantees **100% deterministic accuracy** for competitive gaming and standard charts.
   - For ultra-high volume streaming platforms (e.g., YouTube global video views), a probabilistic estimation mode (Count-Min Sketch / Heavy Hitters) is provided as an architectural extension.
5. **Fault Tolerance & Durability**: The system must survive catastrophic node crashes without permanently losing historical score data or precomputed aggregates.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Traffic & Ingestion Throughput
- **Global Ingestion Rate**: $1,000,000 \text{ write requests/sec}$ (Peak events across global gaming matches or YouTube views).
- **Read Query Rate**: Top $K$ queries are queried by mobile clients, homepage carousels, and dashboards.
  - Active global users querying leaderboards: $50,000 \text{ QPS}$ (Read-to-Write ratio $\approx 1:20$).

#### Storage Estimations (5-Year Horizon)
- Raw Event Payload:
  - `event_id`: 16 bytes (UUID)
  - `entity_id` (video_id / player_id): 16 bytes (UUID / BIGINT)
  - `leaderboard_id`: 8 bytes
  - `metric_delta` / `score`: 8 bytes (BIGINT)
  - `region_code`: 4 bytes (e.g., `US`, `IN`)
  - `timestamp`: 8 bytes
  - Metadata / headers: 20 bytes
  - Total per raw event $\approx 80 \text{ bytes}$.
- **Daily Raw Storage**:
  $$\text{Storage/day} = 1,000,000 \text{ events/sec} \times 86,400 \text{ sec} \times 80 \text{ bytes} \approx 6.91 \text{ TB / day}$$
- Raw events are buffered in Kafka and persisted to cold data lakes (Cassandra / Parquet in S3) for auditability.
- **Aggregated Storage (Top K Precomputed Data)**:
  - 10,000 distinct leaderboards (regions $\times$ categories $\times$ time frames).
  - Storing Top 1,000 items per leaderboard:
    $$\text{Aggregated Record} = 1,000 \text{ items} \times 64 \text{ bytes} \approx 64 \text{ KB per leaderboard}$$
    $$10,000 \text{ leaderboards} \times 64 \text{ KB} \approx 640 \text{ MB total in-memory cache!}$$
  - Aggregated data is exceptionally compact and easily fits entirely within RAM.

#### Memory / Cache Estimations
- Active Redis Sorted Sets holding top 100,000 candidates per active tournament/region:
  - Key overhead + SkipList node + Hash table entry $\approx 128 \text{ bytes}$ per member.
  - $100,000 \text{ candidates} \times 128 \text{ bytes} \approx 12.8 \text{ MB}$ per leaderboard partition.
  - 500 active competitive leaderboards concurrently active:
    $$500 \times 12.8 \text{ MB} \approx 6.4 \text{ GB RAM}$$
  - A modest 3-node Redis cluster with 32 GB RAM per node handles real-time leaderboards with extensive headroom.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|       Entity       |        |    Leaderboard     |        |     ScoreEvent     |
+--------------------+        +--------------------+        +--------------------+
| entity_id (PK,UUID)| 1    * | id (PK, VARCHAR)   | 1    * | event_id (PK, UUID)|
| type (USER|VIDEO)  |<------>| name (VARCHAR)     |<------>| leaderboard_id (FK)|
| display_name (VAR) |        | category (VARCHAR) |        | entity_id (FK)     |
| country_code (VAR) |        | metric_type (ENUM) |        | delta_score (BIG)  |
| created_at (TIME)  |        | window_type (ENUM) |        | timestamp (TIME)   |
+--------------------+        | is_active (BOOL)   |        +--------------------+
                              +--------------------+
                                        │ 1
                                        │
                                        ▼ *
                              +--------------------+
                              |  LeaderboardEntry  |
                              +--------------------+
                              | leaderboard_id(FK) |
                              | entity_id (FK)     |
                              | current_score(BIG) |
                              | current_rank (INT) |
                              | updated_at (TIME)  |
                              +--------------------+
```

### Entity Descriptions
1. **Entity**: The subject being ranked (e.g., `GamerProfile`, `YouTubeVideo`, `SpotifyTrack`).
2. **Leaderboard**: Metadata defining the ranking domain (e.g., `gaming_global_season_4`, `youtube_trending_india_24h`, `spotify_top50_us`).
3. **ScoreEvent**: Immutable event log representing an incremental score change (+50 XP, +1 Video View, +1 Upvote).
4. **LeaderboardEntry**: Point-in-time materialized record containing the computed cumulative score and rank.

---

## 3. Step 3: API Design (Public REST & WebSocket Endpoints)

### 3.1 Ingest Score / View Event
```http
POST /api/v1/scores
Authorization: Bearer <service_token>
Content-Type: application/json

Request Body:
{
  "leaderboard_id": "lb_apex_legends_s9_global",
  "entity_id": "usr_77a98b42-e1c0-4f56",
  "score_delta": 150,
  "metric_type": "SCORE",              // "SCORE" | "VIEW" | "LIKE"
  "region": "IN",
  "timestamp": "2026-10-18T14:30:00Z"
}

Response: 202 Accepted
{
  "status": "QUEUED",
  "message": "Score update queued for asynchronous stream ingestion",
  "event_id": "evt_99182371-bcde-4122"
}
```
> **Status 202 Accepted Rationale**: At 1,000,000 RPS, the score ingestion API cannot wait for synchronous database commits or distributed cache updates. It pushes the event to Kafka and returns `202 Accepted` within $< 15\text{ms}$.

---

### 3.2 Query Top K Leaderboard
```http
GET /api/v1/leaderboards/{leaderboard_id}?time_window=24h&region=IN&limit=10&page=1
Authorization: Bearer <user_token>

Response: 200 OK
{
  "leaderboard_id": "lb_apex_legends_s9_global",
  "time_window": "24h",
  "region": "IN",
  "total_participants": 2450890,
  "last_updated": "2026-10-18T14:32:05Z",
  "data": [
    { "rank": 1, "entity_id": "usr_9981", "display_name": "ShadowNinja", "score": 98450 },
    { "rank": 2, "entity_id": "usr_4412", "display_name": "BunnyGamer", "score": 96210 },
    { "rank": 3, "entity_id": "usr_1029", "display_name": "Valkyrie", "score": 94100 },
    ...
  ],
  "pagination": {
    "current_page": 1,
    "limit": 10,
    "has_next": true
  }
}
```

---

### 3.3 Query Surrounding User Rank ("Players Around Me")
```http
GET /api/v1/leaderboards/{leaderboard_id}/entities/{entity_id}/surrounding?time_window=30d&k=5
Authorization: Bearer <user_token>

Response: 200 OK
{
  "target_entity_id": "usr_5510",
  "target_rank": 47,
  "target_score": 54200,
  "surrounding_range": {
    "from_rank": 45,
    "to_rank": 49
  },
  "peers": [
    { "rank": 45, "entity_id": "usr_1102", "display_name": "TitanX", "score": 54800 },
    { "rank": 46, "entity_id": "usr_8821", "display_name": "Viper", "score": 54500 },
    { "rank": 47, "entity_id": "usr_5510", "display_name": "CurrentUser", "score": 54200, "is_self": true },
    { "rank": 48, "entity_id": "usr_3319", "display_name": "Blaze", "score": 53900 },
    { "rank": 49, "entity_id": "usr_9011", "display_name": "Ghost", "score": 53750 }
  ]
}
```

---

### 3.4 Real-Time Live Push (WebSocket Interface)
For live tournament arenas, clients initiate a WebSocket handshake:
```http
GET /ws/v1/leaderboards/{leaderboard_id}
Upgrade: websocket
Connection: Upgrade
```

Server-to-Client Frame (Broadcast stream):
```json
{
  "event": "LEADERBOARD_DELTA",
  "leaderboard_id": "lb_apex_legends_s9_global",
  "timestamp": 1792345925000,
  "changes": [
    { "old_rank": 4, "new_rank": 2, "entity_id": "usr_4412", "new_score": 96210 }
  ]
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Clients: Mobile Apps, Web, Gaming Consoles ]
                       │
                       ▼
         [ DNS / Edge Cloudflare CDN ]
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
│         (SSL Termination, Token Auth, Rate Limiting)        │
└─────────────────────────────────────────────────────────────┘
          │                                           │
  (POST /api/v1/scores)                     (GET /api/v1/leaderboards)
          │                                           │
          ▼                                           ▼
┌────────────────────────┐                 ┌────────────────────────┐
│  Score Ingestion Svc   │                 │    Ranking Service     │
│  (Stateless Workers)   │                 │   (Top K Aggregator)   │
└────────────────────────┘                 └────────────────────────┘
          │                                           │
          ▼                                           │
┌────────────────────────┐                            │
│  Kafka Message Broker  │                            │
│ (Partitioned by LB_ID) │                            │
└────────────────────────┘                            │
     │                │                               │
     ▼                ▼                               ▼
┌──────────────┐ ┌──────────────────┐      ┌────────────────────────┐
│ Cassandra /  │ │ Stream Processor │----->│   Redis Sorted Sets    │
│ ScyllaDB     │ │ (Apache Flink)   │      │   (Primary ZSET Cache) │
│ (Score Event │ └──────────────────┘      └────────────────────────┘
│ Raw Storage) │          │                           │
└──────────────┘          ▼                           ▼
                 ┌──────────────────┐      ┌────────────────────────┐
                 │ Aggregated DB    │<-----│ WebSocket Push Service │
                 │ (TimescaleDB /   │      │ (Pub/Sub Live Broad-   │
                 │ InfluxDB / PG)   │      │  cast to Clients)      │
                 └──────────────────┘      └────────────────────────┘
```

### High-Level Data Flow Walkthrough
1. **Ingestion Path (Write Path)**:
   - A game client or microservice submits a score change to the API Gateway.
   - The **Score Ingestion Service** validates authentication, generates an event ID, and forwards the event directly to an Apache Kafka topic.
   - **Kafka** acts as a resilient buffer absorbing the 1,000,000 write RPS.
2. **Stream Processing & Memory Update Path**:
   - Stream Consumers read from Kafka.
   - For real-time leaderboards, consumers execute an atomic `ZINCRBY` against a **Redis Sorted Set (ZSET)** cluster.
   - Concurrently, events are persisted into a distributed **Cassandra/ScyllaDB** cluster for durability and audit trail.
3. **Retrieval Path (Read Path)**:
   - Client queries the **Ranking Service** for the Top $K$ list.
   - Ranking Service directly queries the Redis Sorted Set via `ZREVRANGE 0 (K-1) WITHSCORES`.
   - Result is returned to the client within $< 10\text{ms}$.
4. **Historical & Precomputed Path**:
   - For complex historical aggregations (e.g., "Top videos of January 2025 in France"), **Apache Flink** computes tumbling/sliding window rollups and writes them into an **Aggregated Time-Series Database**.
   - Historical queries bypass Redis and read directly from the Aggregated DB.

---

## 5. Step 5: Low-Level Design & Deep Dives

Bunny outlines **three progressive architectural stages** to solve the Top K leaderboard, starting from a pure in-memory approach, evolving through batch precomputation, and culminating in a production-ready **Hybrid Architecture**.

---

### 5.1 Evolution Stage 1: Pure In-Memory Redis Sorted Set (ZSET)

#### Core Data Structure: SkipList + Hash Table
In Redis, a Sorted Set (`ZSET`) is internally implemented using a combination of two data structures:
1. **Hash Table**: Maps `member` $\rightarrow$ `score` providing $O(1)$ lookups.
2. **Skip List**: Keeps members sorted by score in ascending order, enabling $O(\log N)$ insertions, deletions, and updates, and $O(\log N + K)$ range queries.

```
Level 3:  [Head] ------------------------------> [Node 80] --------------------> NIL
Level 2:  [Head] --------------> [Node 40] ------> [Node 80] ------> [Node 95] -> NIL
Level 1:  [Head] -> [Node 10] -> [Node 40] ------> [Node 80] ------> [Node 95] -> NIL
Level 0:  [Head] -> [Node 10] -> [Node 25] -> [Node 40] -> [Node 60] -> [Node 80] -> [Node 95]
```

#### Redis Key Schema Design
To support multi-dimensional queries without massive database scans, keys encode dimension and time frame:
```
leaderboard:{metric}:{region}:{time_window}
```
Examples:
- `lb:views:IN:24h`
- `lb:score:GLOBAL:s9`
- `lb:likes:US:all_time`

#### Key Redis Commands Used
- **Update Score**:
  ```redis
  ZINCRBY lb:score:GLOBAL:s9 150 "usr_77a98b42"
  ```
  *(Increments the score of `usr_77a98b42` by 150 in $O(\log N)$ time)*.
- **Fetch Top 10**:
  ```redis
  ZREVRANGE lb:score:GLOBAL:s9 0 9 WITHSCORES
  ```
  *(Returns top 10 members in descending score order in $O(\log N + K)$ time)*.
- **Get User Rank**:
  ```redis
  ZREVRANK lb:score:GLOBAL:s9 "usr_77a98b42"
  ```
  *(Returns 0-based rank from top in $O(\log N)$ time)*.
- **Get Surrounding Peers ("Around Me")**:
  ```redis
  // 1. Get rank
  user_rank = ZREVRANK lb:score:GLOBAL:s9 "usr_77a98b42" // e.g., 47
  // 2. Fetch range [user_rank - 2, user_rank + 2]
  ZREVRANGE lb:score:GLOBAL:s9 45 49 WITHSCORES
  ```

#### Bottlenecks of Pure In-Memory Stage
1. **Single Node & Memory Limits**: Storing billions of historical records in RAM causes Out-Of-Memory (OOM) crashes.
2. **Single Point of Failure (SPOF)**: If the Redis node restarts without persistent snapshotting, all calculated rankings are erased, requiring expensive replays.
3. **Partitioning by Region**: To scale horizontally, cluster by country (`lb:score:IN:*` on Shard 1, `lb:score:US:*` on Shard 2). However, cross-region or global leaderboards still require merging.

---

### 5.2 Evolution Stage 2: Batch & Stream Precomputation (Cassandra + Apache Flink)

When historical windows (30 days, 90 days, all-time) must be queried across billions of records, pure in-memory storage becomes cost-prohibitive.

```
[Raw Events] ──> [Kafka] ──> [Cassandra Event Store (Raw append-only)]
                               │
                               ▼
                    [Apache Flink Stream Job]
                 (Sliding / Tumbling Window Rollups)
                               │
                               ▼
               [Aggregated DB (TimescaleDB / InfluxDB)]
                               │
                               ▼
                       [Ranking Service]
```

#### Database Schema: Raw Ingestion Store (Cassandra)
Cassandra is chosen for raw event logging because of its **LSM-Tree write-optimized architecture**, zero locking, and linear horizontal scalability.
```sql
CREATE KEYSPACE leaderboard_analytics 
WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east': 3, 'ap-south': 3};

CREATE TABLE leaderboard_analytics.raw_score_events (
    leaderboard_id text,
    time_bucket text,          -- e.g., '2026-10-18-14' (hourly bucket to prevent huge partitions)
    event_timestamp timestamp,
    entity_id text,
    score_delta bigint,
    metadata text,
    PRIMARY KEY ((leaderboard_id, time_bucket), event_timestamp, entity_id)
) WITH CLUSTERING ORDER BY (event_timestamp DESC, entity_id ASC);
```

#### Precomputed Aggregated Schema (TimescaleDB / Relational)
```sql
CREATE TABLE precomputed_leaderboard (
    leaderboard_id VARCHAR(64) NOT NULL,
    time_window VARCHAR(16) NOT NULL,    -- '1h', '24h', '7d', '30d', 'all_time'
    region VARCHAR(8) NOT NULL,
    rank INT NOT NULL,
    entity_id VARCHAR(64) NOT NULL,
    accumulated_score BIGINT NOT NULL,
    computed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (leaderboard_id, time_window, region, rank)
);

CREATE INDEX idx_entity_rank ON precomputed_leaderboard (leaderboard_id, entity_id);
```

#### Stream Processing Logic (Apache Flink)
Flink runs stateful stream processing over Kafka:
- **Tumbling Windows**: For fixed intervals (e.g., 1-hour chart).
- **Sliding Windows**: For rolling 24-hour charts (e.g., 24-hour window sliding every 5 minutes).
- **Watermarks**: Handles late-arriving events up to 60 seconds without corrupting rankings.

#### Limitations of Pure Batch/Stream Stage
- Introduces processing lag (5 seconds to 1 minute).
- Cannot support instant, frame-by-frame live score leaderboards for esports.

---

### 5.3 Evolution Stage 3: The Production Hybrid Architecture (Lambda / Kappa Pattern)

Bunny demonstrates that the industry standard (used by YouTube, Spotify, and Twitch) is a **Hybrid Architecture** combining:
1. **Hot Path (Real-time Redis ZSET)**: Handles live streaming events, real-time rolling 24-hour windows, and instant WebSocket pushes. Keys are configured with strict TTLs (e.g., 24 to 72 hours).
2. **Cold/Warm Path (Kafka $\rightarrow$ Flink $\rightarrow$ Aggregated DB)**: Computes and archives durable 7-day, 30-day, and all-time leaderboards.
3. **Disaster Recovery Rehydration**: If Redis crashes, a cold-start worker hydrates the Redis ZSET using the latest snapshot from the Aggregated DB, then replays the tail of the Kafka log from the checkpoint offset.

```
                           ┌────────────────────────────┐
                           │      Kafka Score Topic     │
                           └────────────────────────────┘
                              │                      │
                   (Hot Path) │                      │ (Cold / Analytics Path)
                              ▼                      ▼
                 ┌──────────────────────┐  ┌──────────────────────┐
                 │ Redis Ingestion Svc  │  │ Apache Flink Engine  │
                 └──────────────────────┘  └──────────────────────┘
                              │                      │
                              ▼                      ▼
                 ┌──────────────────────┐  ┌──────────────────────┐
                 │  Redis Sorted Sets   │  │ Cassandra / ScyllaDB │
                 │  (TTLs: 24h - 72h)   │  │   (Raw Durability)   │
                 └──────────────────────┘  └──────────────────────┘
                              │                      │
                    Read Miss │                      │ Rollups & Aggregations
                   / Fallback │                      ▼
                              │            ┌──────────────────────┐
                              └───────────>│ Aggregated Time-     │
                                           │ Series Store         │
                                           └──────────────────────┘
```

#### Read Decision Matrix in Ranking Service
```python
def get_leaderboard(leaderboard_id, window, region, k):
    # Rule 1: Real-time queries hit Redis Sorted Sets
    if window in ["realtime", "1h", "24h"]:
        redis_key = f"lb:{leaderboard_id}:{region}:{window}"
        result = redis_client.zrevrange(redis_key, 0, k - 1, withscores=True)
        if result:
            return format_response(result)
        # Fallback to Aggregated DB if cache expired or recovering
        return read_from_aggregated_db(leaderboard_id, window, region, k)
    
    # Rule 2: Multi-day historical queries hit precomputed Aggregated DB
    else:
        return read_from_aggregated_db(leaderboard_id, window, region, k)
```

---

### 5.4 Advanced Deep Dive: Handling Score Ties & Determinism
In competitive gaming and music charts, identical scores are common (e.g., multiple players with 10,000 XP).
If two players have the exact same score, how is rank determined deterministically?

#### Solution: Composite Fractional Score Encoding
Redis sorts members with identical scores lexicographically by member name. To enforce a **first-to-reach-score-wins** rule:
$$\text{Effective Score} = \text{Score} + \left(1 - \frac{\text{Timestamp}}{10^{13}}\right)$$
- An earlier timestamp adds a slightly larger fractional value than a later timestamp.
- **Example**:
  - Player A hits score 10,000 at timestamp `1700000000`: Fractional boost = $0.83000$. Total = `10000.83000`.
  - Player B hits score 10,000 at timestamp `1700000100`: Fractional boost = $0.82999$. Total = `10000.82999`.
  - Player A automatically ranks above Player B inside Redis without extra sorting logic!

---

### 5.5 Advanced Deep Dive: Probabilistic Estimation for Billion-Scale Feeds (Count-Min Sketch + Min-Heap)

When ranking items at hyper-scale (e.g., YouTube global video views with billions of items where maintaining an exact Redis SkipList requires terabytes of memory), Bunny highlights the **Heavy Hitters (Top K)** probabilistic pattern:

```
Stream of Events (1M views/sec)
         │
         ▼
┌───────────────────────────────┐
│     Count-Min Sketch (CMS)    │ ─── Estimate frequency in O(1) space
└───────────────────────────────┘
         │
         ▼
┌───────────────────────────────┐
│  In-Memory Min-Heap (Size K)  │ ─── Retains top K items in O(log K)
└───────────────────────────────┘
```

1. **Count-Min Sketch**: A 2D array of depth $d$ and width $w$ with $d$ hash functions. Each event increments $d$ counters. Point lookup returns the minimum of the $d$ counters (an upper bound estimation with guaranteed error bounds $\epsilon$).
2. **Min-Heap of Size $K$**:
   - The heap stores only the top $K$ items observed so far.
   - For every incoming event, look up its estimated frequency in the CMS.
   - If estimated frequency $>$ root of the Min-Heap (the current $K$-th item), replace the root and call `heapify()`.
3. **Space Complexity**: Fixed space $O(K + d \times w)$ independent of stream size (a few megabytes for billions of events).

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: Design a real-time gaming leaderboard that shows top 100 players globally and by region. How do you handle millions of score updates per second?

Real-time gaming leaderboard architecture: Requirements:

- **Point 1**: Top 100 global,

- **Point 2**: Top 100 per region (US, EU, APAC),

- **Point 3**: Millions of score updates/sec,

- **Point 4**: <100ms query latency,

- **Handle score ties. Architecture**:
  Kafka (score events) → Redis Consumer (real-time ZSET) + DB Consumer (persistence) → Redis Cluster (reads) + PostgreSQL (backup). Score update flow:

- **Player scores 1000 points in game → Game server sends**:
  POST /api/v1/scores with {player_id: 'P123', score_delta: 1000, region: 'US', match_id: 'M456', timestamp},

- **Score Service validates**:
  Check player exists, check match active, check for duplicate (Redis dedup with 5-min TTL),

- **Publish to Kafka**:
  'score.updated' topic with {player_id, score_delta: 1000, region: 'US', timestamp}, partition by player_id hash → ensures ordering per player. Redis Consumer (real-time updates):

- **Point 1**: Consumes from Kafka (100 consumer instances, 1 per partition),

- **Updates Redis ZSETs**:
  ZINCRBY leaderboard:global:alltime 1000 P123 (global leaderboard), ZINCRBY leaderboard:US:alltime 1000 P123 (regional leaderboard), ZINCRBY leaderboard:global:daily:{YYYY-MM-DD} 1000 P123 (daily leaderboard),

- **Lag**:
  <100ms from event to Redis update (Kafka consumer lag monitoring). DB Consumer (persistence):

- **Point 1**: Separate consumer group writes to PostgreSQL,

- **Point 2**: INSERT INTO scores (player_id, score_delta, region, timestamp, match_id),

- **Eventual consistency (lag ~1 second, acceptable for backup/analytics). Query flow**:
  User requests top 100: GET /api/v1/leaderboards/global/top?limit=100. Ranking Service:

- **Check Redis**:
  ZREVRANGE leaderboard:global:alltime 0 99 WITHSCORES (get top 100 with scores), Time: <1ms (Redis in-memory lookup),

- **Enrich data**:
  For each player_id, fetch player details: MGET player:P123 player:P456 ... (batch fetch from Redis player cache), If cache miss → query DB: SELECT username, avatar_url FROM players WHERE player_id IN (...),

- **Response**:
  [{rank: 1, player_id: 'P123', username: 'ProGamer', score: 1500000}, {rank: 2, ...}, ...],

- **Total latency**:
  <10ms (1ms ZREVRANGE + 5ms MGET + 2ms serialization). Regional leaderboard: GET /api/v1/leaderboards/US/top?limit=100, Same flow but query: ZREVRANGE leaderboard:US:alltime 0 99. Handle score ties:

- **Timestamp tiebreaker**:
  When scores equal, player who reached score first ranks higher,

- **Implementation**:
  Store score as float: score + (timestamp / 1e12), Example: Player A: 1000 points at t=1674382200 → 1000.000001674382200, Player B: 1000 points at t=1674382300 → 1000.000001674382300, Player A ranks higher (earlier timestamp),

- **ZADD leaderboard**:
  global:alltime {score_with_timestamp} P123. Scaling for millions of updates/sec:

- **Kafka**:
  100 partitions × 10K events/sec/partition = 1M events/sec, add more partitions to scale beyond,

- **Redis Cluster**:
  10 master nodes (each handles 100K ZINCRBY/sec) = 1M ZINCRBY/sec, shard by leaderboard_id: hash(leaderboard:global:alltime) mod 10 → node 5,

- **Redis Consumer**:
  100 instances (1 per Kafka partition), auto-scale based on Kafka lag (if lag > 1 min → add more consumers). Redis optimization:

- **Pipelining**:
  Batch updates: PIPELINE; ZINCRBY leaderboard:global:alltime 1000 P123; ZINCRBY leaderboard:US:alltime 1000 P123; EXEC; (send 2 commands in 1 network round trip),

- **Lua script**:
  EVALSHA to update multiple ZSETs atomically (prevents partial updates if one fails),

- **Memory**:
  Top 100 leaderboard = 100 entries × 100 bytes = 10 KB (tiny), Global leaderboard (10M players) = 1 GB (manageable). Disaster recovery:

- **Redis crash**:
  All ZSET data lost (in-memory only),

- **Rebuild from DB**:
  Background job queries: SELECT player_id, SUM(score_delta) as total_score FROM scores GROUP BY player_id, Populate Redis: FOR EACH player: ZADD leaderboard:global:alltime {total_score} {player_id}, Time: Rebuild 10M players takes ~10 minutes (acceptable),

- **Redis persistence**:
  Enable RDB snapshots (every 5 min) or AOF (append-only file) for faster recovery. Advanced features:

- **Player's rank**:
  GET /api/v1/leaderboards/global/rank/P123 → ZREVRANK leaderboard:global:alltime P123 returns rank (0-indexed), Example: rank=42 → display as #43,

- **Percentile**:
  Player in top 1%? → total_players = ZCARD leaderboard:global:alltime, percentile = (rank / total_players) × 100,

- **Surrounding players**:
  ZREVRANGE leaderboard:global:alltime {rank-5} {rank+4} WITHSCORES (show 5 above + player + 4 below). Monitoring:

- **Kafka consumer lag**:
  Alert if lag > 1 min (leaderboard stale),

- **Redis memory**:
  Alert if > 80% (risk of eviction),

- **Query latency**:
  p99 < 10ms (SLA),

- **Consistency check**:
  Hourly job compares Redis top-100 vs DB top-100, alert if mismatch. Production example: Fortnite uses similar architecture, 250M+ players globally, millions of concurrent matches, real-time leaderboards updated every few seconds, Redis Cluster (100+ nodes), Kafka (1000+ partitions), sub-10ms query latency for top 100.

---

### Q2: How would you design a YouTube Trending page that shows top 50 trending videos? Explain the time-decay algorithm.

YouTube Trending page design with time-decay scoring: Requirements:

- **Point 1**: Top 50 trending videos (updated hourly),

- **Point 2**: Balance recency and engagement (new viral video ranks higher than old popular video),

- **Point 3**: Regional trending (US, UK, India, etc.),

- **Category trending (Music, Gaming, News). Time-decay algorithm (Hacker News style)**:
  Formula: trending_score = engagement_score / (age_in_hours + 2)^gravity, Parameters:

- **engagement_score**:
  Weighted sum of interactions, engagement_score = views + (likes × 10) + (comments × 20) + (shares × 30), Weights: Shares > Comments > Likes > Views (stronger engagement signal weighted higher),

- **age_in_hours**:
  Time since video published, age_in_hours = (NOW() - published_at) / 3600 (convert seconds to hours),

- **gravity**:
  Decay rate (typically 1.5-2.0), Higher gravity = faster decay (older content drops faster), YouTube likely uses gravity ≈ 1.8. Example calculation: Video A (old viral): Published 7 days ago (168 hours), 10M views, 500K likes, 50K comments, 10K shares, engagement_score = 10000000 + (500000×10) + (50000×20) + (10000×30) = 16300000, trending_score = 16300000 / (168+2)^1.8 = 16300000 / 1173 ≈ 13895. Video B (new rising): Published 3 hours ago, 500K views, 50K likes, 5K comments, 1K shares, engagement_score = 500000 + (50000×10) + (5000×20) + (1000×30) = 1130000, trending_score = 1130000 / (3+2)^1.8 = 1130000 / 11.8 ≈ 95763, Video B ranks HIGHER (trending now despite lower absolute engagement). Architecture: Batch computation + Redis caching. Batch job (hourly):

- **Trigger**:
  Cron runs every hour at :00 (12:00, 13:00, 14:00),

- **Query recent videos**:
  SELECT video_id, views, likes, comments, shares, published_at FROM videos WHERE published_at > NOW() - INTERVAL '7 days' (only videos from last 7 days eligible for trending),

- **Calculate trending score**:
  FOR EACH video: age_in_hours = (NOW() - published_at) / 3600, engagement_score = views + (likes×10) + (comments×20) + (shares×30), trending_score = engagement_score / POW(age_in_hours + 2, 1.8),

- **Sort and store top 1000**:
  ORDER BY trending_score DESC LIMIT 1000,

- **Write to Redis**:
  DEL leaderboard:trending (clear old), ZADD leaderboard:trending {trending_score} {video_id} (add all 1000 videos), EXPIRE leaderboard:trending 7200 (2-hour TTL, in case batch job fails next run). Regional trending:

- **Point 1**: Separate batch job per region,

- **Filter**:
  WHERE region IN ('US', 'CA', 'MX') for North America trending,

- **Store**:
  ZADD leaderboard:trending:US {trending_score} {video_id}. Category trending:

- **Filter**:
  WHERE category='Music',

- **Store**:
  ZADD leaderboard:trending:Music {trending_score} {video_id}. Query flow: User opens Trending page: GET /api/v1/trending?region=US&limit=50. Ranking Service:

- **Check Redis**:
  ZREVRANGE leaderboard:trending:US 0 49 WITHSCORES (get top 50), Cache hit (99% of requests): Return from Redis (<1ms), Cache miss (rare, during batch job or Redis restart): Fallback to DB query (recalculate trending scores on-the-fly, slower ~500ms),

- **Enrich video details**:
  For each video_id, fetch metadata: MGET video:V123 video:V456 ... (title, thumbnail, channel),

- **Response**:
  [{rank: 1, video_id: 'V123', title: 'New Music Video', trending_score: 95763, views: 500K, ...}]. Real-time vs batch trade-off:

- **Batch (hourly update)**:
  Simpler implementation, less compute (recalculate once per hour, not on every query), Staleness: Trending list can be up to 1 hour old (acceptable for most users),

- **Real-time (per-query calculation)**:
  Always fresh scores, High compute cost (recalculate for 1M videos on every query → 100ms query time), YouTube uses batch approach (hourly or even daily updates for trending). Optimization:

- **Incremental computation**:
  Instead of recalculating all videos every hour, only update videos with new engagement (events in last hour), Kafka stream: Consume view/like/comment events, update trending score in real-time (hybrid),

- **Materialized view**:
  CREATE MATERIALIZED VIEW trending_videos AS SELECT video_id, ..., REFRESH MATERIALIZED VIEW CONCURRENTLY (non-blocking), Query trending view instead of raw videos table (10× faster). Alternative: Reddit Hot algorithm: Formula: hot_score = log10(engagement) + (age_in_seconds / 45000), Logarithm: Diminishing returns (100 vs 1000 engagement has less impact than 10 vs 100), Linear time decay: Older posts decay steadily, Use case: When you want engagement to matter more than recency (Reddit front page stays stable longer than YouTube trending). Monitoring:

- **Batch job duration**:
  Alert if job takes >10 min (normally 5 min),

- **Trending consistency**:
  Compare manual check vs automated trending (validate viral videos appear),

- **Regional coverage**:
  Ensure all regions have trending data (no empty leaderboards). Advanced: Personalized trending:

- **User's watch history**:
  SELECT categories_watched, channels_subscribed FROM user_behavior WHERE user_id={user_id},

- **Boost relevant categories**:
  If user watches Music videos 80% of time, boost Music videos in trending, Formula: personalized_score = trending_score × (1 + category_affinity),

- **ML model**:
  Train model to predict 'will user click this trending video', Features: trending_score, user_watch_history, video_category, time_of_day. Production: YouTube Trending page updated daily or hourly (not real-time), uses combination of engagement signals + time decay, regional and category-specific trending, reviewed by human moderators (prevent manipulation, ensure quality), ML-based personalization for logged-in users.

---

### Q3: Compare Redis ZSET vs database-based leaderboard approaches. When would you choose one over the other?

Redis ZSET vs Database leaderboard comparison: Redis ZSET approach: Data structure: Sorted set (ZSET) stores (member, score) pairs, automatically sorted by score. Operations:

- **ZADD leaderboard {score} {member}**:
  Add/update entry O(log N),

- **ZINCRBY leaderboard {increment} {member}**:
  Increment score O(log N),

- **ZREVRANGE leaderboard 0 9**:
  Get top-10 O(log N + K),

- **ZREVRANK leaderboard {member}**:
  Get rank O(log N),

- **ZSCORE leaderboard {member}**:
  Get score O

- **. Example**:
  ZADD leaderboard 1000 player1, ZINCRBY leaderboard 500 player1 (now 1500), ZREVRANGE leaderboard 0 2 WITHSCORES → [(player1, 1500), (player2, 1200), (player3, 1000)]. Pros:

- **Blazing fast**:
  All operations <1ms (in-memory), top-100 query in ~1ms even with 10M players,

- **Simple**:
  Just Redis commands, no SQL, no indexes, no query optimization,

- **Atomic**:
  ZINCRBY is thread-safe (no race conditions),

- **Built-in ranking**:
  ZREVRANK gives rank without scanning,

- **Auto-sorted**:
  No manual sorting needed. Cons:

- **Memory only**:
  No persistence by default (RDB/AOF mitigates but not 100% durable),

- **Single machine limit**:
  One ZSET must fit in RAM of single node (can't shard a single ZSET), For 100M players × 100 bytes = 10 GB (expensive Redis server needed),

- **No time-series**:
  Only current state, can't query 'What was the leaderboard last week?',

- **Limited queries**:
  Can't do complex filters (e.g., 'Top players from California who started in 2024'),

- **Cost**:
  Redis memory expensive (RAM > disk storage by 10-100×). Database approach (PostgreSQL/MySQL): Schema: CREATE TABLE leaderboard (player_id varchar PRIMARY KEY, score bigint, updated_at timestamp); CREATE INDEX idx_score ON leaderboard(score DESC); Query top-10: SELECT player_id, score FROM leaderboard ORDER BY score DESC LIMIT 10; Update score: UPDATE leaderboard SET score = score + {increment} WHERE player_id = {player_id}; Get rank: SELECT COUNT(*) FROM leaderboard WHERE score > (SELECT score FROM leaderboard WHERE player_id = {player_id}); (counts how many players have higher score). Pros:

- **Persistent**:
  Data on disk, survives crashes (ACID transactions),

- **Scalable storage**:
  Disk cheaper than RAM (10 GB disk << 10 GB RAM), can store 1B+ players,

- **Time-series**:
  Add timestamp column, query historical leaderboards ('Top players on 2024-01-01'),

- **Complex queries**:
  Filter by region, category, date range, etc.,

- **Mature ecosystem**:
  Backups, replication, monitoring well-established. Cons:

- **Slower**:
  Top-10 query ~50-100ms (disk I/O, index scan), 100× slower than Redis,

- **Rank calculation expensive**:
  Get rank query scans all players with score > player's score → O(N) time for large N, For 10M players, rank query takes ~500ms (unacceptable for real-time leaderboards),

- **Concurrent updates**:
  UPDATE locks row, high contention for popular players (score updated 1000s of times/sec),

- **Index overhead**:
  Score index needs rebalancing on every UPDATE (slower writes). Hybrid approach (Redis + DB): Best of both worlds. Architecture: Writes: Score event → Kafka →

- **Redis Consumer**:
  ZINCRBY leaderboard {increment} {player} (real-time),

- **DB Consumer**:
  INSERT INTO score_events (player_id, score_delta, timestamp) (persistent log). Reads:

- **Try Redis**:
  ZREVRANGE leaderboard 0 9 (fast, <1ms),

- **If Redis miss**:
  Query DB: SELECT player_id, SUM(score_delta) FROM score_events GROUP BY player_id ORDER BY SUM DESC LIMIT 10 (slow, ~100ms), rebuild Redis from DB result. Consistency: Redis reflects latest scores (<1 sec lag from Kafka), DB is source of truth (eventual consistency, ~1-5 sec lag), Periodic reconciliation: Nightly job compares Redis vs DB, rebuilds Redis if mismatch. Disaster recovery: Redis crash → rebuild from DB (takes ~10 min for 10M players, acceptable). Decision matrix: Choose Redis ZSET when:

- **Point 1**: Need low latency (<10ms for top-K queries),

- **Point 2**: Leaderboard fits in memory (<100M players → <10 GB),

- **Point 3**: Real-time updates critical (gaming, live sports),

- **Point 4**: Simple use case (global top-K, no complex filters),

- **Can tolerate data loss risk (use RDB/AOF for mitigation). Choose Database when**:
  

- **Point 1**: Need persistence (can't lose leaderboard data),

- **Point 2**: Large scale (>100M players, >10 GB data),

- **Point 3**: Historical queries needed ('Top players last month'),

- **Point 4**: Complex filters (region, category, time ranges),

- **Lower budget (disk storage 10× cheaper than RAM). Choose Hybrid when**:
  

- **Point 1**: Need BOTH low latency AND persistence,

- **Point 2**: Production system with millions of users,

- **Point 3**: Can manage complexity (Kafka + Redis + DB),

- **Multi-dimensional leaderboards (regional, category, time windows). Materialized views (advanced DB approach)**:
  CREATE MATERIALIZED VIEW top_1000_players AS SELECT player_id, score FROM leaderboard ORDER BY score DESC LIMIT 1000; REFRESH MATERIALIZED VIEW top_1000_players; (refresh hourly). Query: SELECT * FROM top_1000_players LIMIT 10; (~1ms, pre-computed). Trade-off: Freshness vs performance (view can be 1 hour stale). Sharding (scale DB approach): Shard by player_id hash, not by score (score-based sharding causes hotspots → top players all on same shard). Query top-K: Scatter-gather across all shards → merge results → return top-10. Expensive but scales to billions of players. Production examples: Gaming (Fortnite, PUBG): Hybrid (Redis for real-time, DB for persistence), Music (Spotify Top 50): Batch DB + Redis cache (hourly batch, cache top-1000), Social media (Twitter Trends): Redis ZSET (short-lived trending, OK to lose data), Finance (Stock tickers): Database (need ACID transactions, audit trail). Summary: Redis = fast but volatile, Database = slow but durable, Hybrid = best of both but complex. Choose based on latency requirements, data size, durability needs, and budget.

---

## 7. Architectural Trade-Off Analysis

| Dimension | Approach 1: Redis Sorted Set | Approach 2: Cassandra + Flink Batch | Approach 3: Hybrid Architecture (Selected) | Alternative: Count-Min Sketch |
| :--- | :--- | :--- | :--- | :--- |
| **Latency (Reads)** | $< 5\text{ms}$ (RAM) | $50\text{ms} - 200\text{ms}$ (DB query) | $< 5\text{ms}$ for real-time; $< 50\text{ms}$ for history | $< 2\text{ms}$ (In-memory heap) |
| **Write Throughput** | Up to $100\text{K}$ writes/sec per node | $1\text{M}+$ writes/sec via LSM log | $1\text{M}+$ writes/sec buffered via Kafka | Massive ($10\text{M}+$ ops/sec in RAM) |
| **Accuracy** | 100% Deterministic | 100% Deterministic | 100% Deterministic | Probabilistic (Approximation with $\epsilon$ error) |
| **Data Durability** | Low (Volatile without persistence) | Extreme (Cassandra replication) | Extreme (Cassandra raw + Redis TTLs) | Volatile |
| **Historical Queries** | Very Poor (OOM risk) | Native (Time-series aggregations) | Native (Aggregated DB handles past windows) | Not supported |
| **Operational Cost** | High RAM costs for billions of keys | High disk / compute costs | Balanced (RAM only for hot 24h data) | Lowest resource footprint |

---

## 8. Failure Modes & Resiliency Strategies

1. **Redis Node Failure & Split-Brain**:
   - Deployed as a Redis Sentinel / Cluster setup with active read replicas.
   - Automatic failover within 3 seconds.
   - In catastrophic cluster failure, the Ranking Service temporarily fails over reads to the Aggregated DB while a background job rehydrates Redis from the Cassandra stream.
2. **Kafka Partition Hot-Spotting**:
   - If a viral video or celebrity player receives 80% of all events, a single Kafka partition can lag.
   - **Remedy**: Compound partition keys: `hash(leaderboard_id + "_" + (event_timestamp % 10))`. Distributes load across 10 sub-partitions, with the consumer aggregating sub-partitions before writing to Redis.
3. **Thundering Herd on Leaderboard Expiry**:
   - When a daily leaderboard expires at midnight UTC, millions of users simultaneously query the new leaderboard.
   - **Remedy**: Warm the cache proactively 5 minutes before midnight UTC using Flink tumbling window snapshots; use cache mutex locks to prevent stampeding the Aggregated DB.
