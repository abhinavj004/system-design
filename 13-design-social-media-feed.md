# System Design 13: Design Social Media Feed Platform (Facebook / Instagram / Twitter)

> **Source**: [Interview with Bunny - Episode 13](https://www.youtube.com/watch?v=MQOz1yZaC54)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/13](https://www.interviewwithbunny.com/systemdesign/13)  
> **Topic**: Architecting a Hyperscale Social Network Feed Generation Engine (Facebook NewsFeed, Instagram Home Feed, Twitter/X Timeline)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Clarifying Questions & Core Scope
- **Platform Style**: Generalized social network combining text, images, and videos (Instagram-style directional following graph or Facebook-style bilateral friendship).
- **Core Problem Focus**: Low-latency, personalized, real-time home feed generation with infinite scrolling.
- **Explicit Out of Scope**: Direct peer-to-peer chat/messaging (covered in Episode 8: WhatsApp), Ephemeral 24-hour stories, and live video streaming (covered in Episode 4: Netflix).
- **Feed Freshness vs. Consistency**: Highly available, eventually consistent system. Sub-second read response time takes absolute priority over instantaneous global consistency.

---

### 1.2 Functional Requirements (FR)
1. **User Onboarding & Profile**: Users can register, log in (JWT-authenticated), and view or update profile metadata.
2. **Post Creation & Media Upload**: Users can author posts containing text and media attachments (photos, videos) with automated content moderation and policy sanitization.
3. **Follower / Social Graph**: Users can follow/unfollow other users and query their follower/following lists.
4. **Social Interactions**: Users can like/unlike posts and comments, and add nested text comments to posts.
5. **Real-Time Feed Generation**: Users can retrieve their personalized Home Feed containing relevant, recent posts from accounts they follow, sorted chronologically or ranked by engagement.
6. **Infinite Scrolling Pagination**: The feed supports cursor-based pagination to stream continuous 20–50 post batches as the user scrolls downwards.

---

### 1.3 Non-Functional Requirements (NFR)
1. **CAP Theorem Posture (AP - Availability & Partition Tolerance)**:
   - Social media feeds are strictly **AP systems**. If a regional network partition occurs, the app must still display cached or slightly stale feeds rather than crashing or throwing errors.
   - **Eventual Consistency**: It is entirely acceptable if a friend's new photo takes 10 to 60 seconds to populate across all followers' feeds.
2. **Aggressive Latency Budgets**:
   - Home Feed generation/retrieval: $< 200\text{ ms}$ for the initial 20-item viewport.
   - Post publication acknowledgment: $< 500\text{ ms}$ (asynchronous background ingestion).
3. **Hyperscale Traffic & Read-Heavy Skew**:
   - Extreme read-to-write skew ($100:1$). The architecture must optimize for near-instantaneous read fan-out without querying relational disk storage on every feed load.
4. **Fault Tolerance & Zero Event Loss**:
   - Guaranteed at-least-once ingestion for user engagement actions (likes, views, comments) via distributed commit logs (Apache Kafka).

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Active Scale & Daily Volume
- **Daily Active Users (DAU)**: $500\text{ Million}$.
- **Average Follows per User**: $200\text{ accounts}$.
- **Posts Created per Day**: $50\text{ Million new posts/day}$.
  $$\text{Post Write QPS} = \frac{50,000,000}{86,400\text{ s}} \approx 580\text{ posts/sec} \quad (\text{Peak: } 2,500\text{ writes/sec})$$
- **Feed Reads per Day**:
  - Each DAU checks their feed $10\text{ times/day}$:
  $$\text{Total Feed Reads} = 500,000,000 \times 10 = 5\text{ Billion reads/day}$$
  $$\text{Average Feed Read QPS} = \frac{5,000,000,000}{86,400\text{ s}} \approx 58,000\text{ reads/sec} \quad (\text{Peak: } 150,000\text{ reads/sec})$$
- **Engineering Conclusion**: A read QPS of $150,000/\text{sec}$ running relational `JOIN` queries across followers and posts would require thousands of sharded database nodes and result in catastrophic latency. Feeds **must be precomputed in in-memory caches (Redis)**.

#### Storage Growth (5-Year Projection)
- **Textual Post Metadata**:
  - $50\text{M posts/day} \times 600\text{ bytes/record} \approx 30\text{ GB/day} \approx 11\text{ TB/year} \to 55\text{ TB in 5 years}$.
- **Media Storage (S3 / Blob Storage + CDN)**:
  - Assume $30\text{ Million posts}$ have media (images/videos) with average compressed size of $2\text{ MB}$:
  $$30,000,000 \times 2\text{ MB} = 60\text{ TB/day} \approx 22\text{ PB/year} \to 110\text{ PB in 5 years}$$.

#### In-Memory Feed Cache Sizing (Redis)
- To maintain snappy scrolling, we cache the top $100\text{ post IDs}$ for daily active users.
- Each post reference: `post_id` (8 bytes) + `timestamp` (8 bytes) = 16 bytes.
- $100\text{ items} \times 16\text{ bytes} \approx 1.6\text{ KB per user}$.
- Caching feeds for all $500\text{ Million DAU}$:
  $$500,000,000 \times 1.6\text{ KB} \approx 800\text{ GB of RAM}$$.
- With Redis overhead and replication ($3\times$), total cluster RAM $\approx 2.4\text{ TB}$, easily hosted across a 16-node Redis cluster.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

```
+-----------------------------------------------------------------------------------+
|                                  CORE DOMAIN ENTITIES                             |
+-----------------------------------------------------------------------------------+

   +-----------------------+              1 : N             +-----------------------+
   |         User          |------------------------------->|         Post          |
   +-----------------------+                                +-----------------------+
   | user_id (PK, UUID)    |                                | post_id (PK, TimeUUID)|
   | username, email       |                                | author_id (FK)        |
   | password_hash         |                                | post_type (TXT/IMG/VID|
   | bio, avatar_url       |                                | content_text          |
   | is_celebrity (BOOL)   |                                | media_url, thumb_url  |
   | created_at            |                                | like_count, cmt_count |
   +-----------------------+                                | created_at            |
         | 1             | 1                                +-----------------------+
         |               |                                       | 1             | 1
    1 : N|               | 1 : N                            1 : N|          1 : N|
         v               v                                       v               v
+------------------+  +--------------------+               +-----------+  +---------+
|    Follower      |  |      Comment       |               | Reaction  |  | FeedItem|
+------------------+  +--------------------+               +-----------+  +---------+
| follow_id (PK)   |  | comment_id (PK)    |               | like_id   |  | user_id |
| follower_id (FK) |  | post_id (FK)       |               | post_id   |  | post_id |
| following_id (FK)|  | user_id (FK)       |               | user_id   |  | score   |
| status (ACCEPTED)|  | content_text       |               | type(HEART|  | gen_time|
| created_at       |  | created_at         |               | timestamp |  +---------+
+------------------+  +--------------------+               +-----------+
```

---

## 3. Step 3: API & Interface Design

### 3.1 Post Creation & Ingestion API

#### Create New Post
```http
POST /api/v1/posts
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "content_text": "Exploring system design with Bunny!",
  "media_type": "IMAGE",
  "media_url": "https://cdn.socialmedia.com/raw/u912_img_491.jpg",
  "thumbnail_url": "https://cdn.socialmedia.com/thumb/u912_img_491.jpg"
}
```
**Response (`202 Accepted`)**:
```json
{
  "post_id": "pst_11ea92b1-4901",
  "status": "QUEUED_FOR_MODERATION",
  "created_at": "2026-10-18T18:40:00Z"
}
```

---

### 3.2 Personalized Home Feed Retrieval API

#### Get Paginated Feed (Cursor-Based Pagination)
```http
GET /api/v1/feed?limit=20&cursor=eyJwb3N0X2lkIjogInBzdF8xMTAwMSIsICJ0aW1lIjogMTcyOTI3NTMwfQ==
Authorization: Bearer <jwt_token>
```
**Response (`200 OK`)**:
```json
{
  "data": [
    {
      "post_id": "pst_881902a-99ef",
      "author": {
        "user_id": "usr_alex_77",
        "username": "alex77",
        "avatar_url": "https://cdn.socialmedia.com/avatars/alex.jpg"
      },
      "content": "Designing high throughput systems!",
      "media_url": "https://cdn.socialmedia.com/images/arch_dia.png",
      "like_count": 1420,
      "comment_count": 89,
      "has_liked": true,
      "created_at": "2026-10-18T18:32:15Z"
    }
  ],
  "next_cursor": "eyJwb3N0X2lkIjogInBzdF84ODE5MDJhIiwgInRpbWUiOiAxNzI5Mjc1MTM1fQ==",
  "has_more": true
}
```

---

### 3.3 Follow Graph & Engagement Endpoints

#### Follow User
```http
POST /api/v1/users/{user_id}/follow
Authorization: Bearer <jwt_token>
```
**Response (`200 OK`)**: `{"status": "FOLLOWING", "following_id": "usr_bob_42"}`

#### Like Post
```http
POST /api/v1/posts/{post_id}/like
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "reaction_type": "LIKE" // LIKE, LOVE, HAHA, WOW
}
```
**Response (`200 OK`)**: `{"post_id": "pst_881902a-99ef", "total_likes": 1421}`

---

## 4. Step 4: High-Level Architecture (HLD)

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/jhvkjhvuiy79648754786f76r875.png?updatedAt=1756069190363)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 13](https://www.interviewwithbunny.com/systemdesign/13))*

### Component Interaction Topology
```
                            +─────────────────────────────────────────+
                            |     Clients (Web, iOS, Android App)     |
                            +─────────────────────────────────────────+
                                                 │
                                                 ▼
                            +─────────────────────────────────────────+
                            |     API Gateway & Reverse Proxy / LB    |
                            | (Auth, Rate Limiter, Round-Robin Route) |
                            +─────────────────────────────────────────+
                                                 │
                   ┌─────────────────────────────┼─────────────────────────────┐
                   ▼                             ▼                             ▼
   +───────────────────────────────+   +───────────────────+   +───────────────────────────────+
   |        Content Service        |   | Follower Service  |   |         Feed Service          |
   |      (Uploads & Authoring)    |   |  (Social Graph)   |   |   (Serves Timeline to User)   |
   +───────────────────────────────+   +───────────────────+   +───────────────────────────────+
                   │                             │                             │
                   ▼                             ▼ Reads/Writes                ▼ Reads Precomputed
   +───────────────────────────────+   +───────────────────+   +───────────────────────────────+
   |     Apache Kafka Cluster      |   |    Follower DB    |   |     Redis Feed Cache ZSET     |
   |  [raw-posts]  [filtered-posts]|   | (PostgreSQL Core) |   |  (Key: feed:{user_id} -> ZSET)|
   +───────────────────────────────+   +───────────────────+   +───────────────────────────────+
         │                   │                                                 ▲
         ▼                   ▼ Consumes Clean Posts                            │
+─────────────────+  +───────────────────────────────────+                     │
| AI Moderator    |  |     Fan-Out Engine Workers        |─────────────────────┤
| Service         |  | (Push Model for Standard Users)   | Updates User Cache  │
+─────────────────+  +───────────────────────────────────+                     │
         │ (If blocked)      │                                                 │
         ▼                   ├──────────────────┐                              │
+─────────────────+          ▼                  ▼                              │
| Notification Svc|  +───────────────+  +──────────────────────+               │
| (Push / FCM)    |  |   Posts DB    |  |   Blob Storage / S3  |               │
+─────────────────+  |  (Cassandra)  |  | (Images & Video CDN) |               │
                     +───────────────+  +──────────────────────+               │
                                                                               │
                                                                               │
   +───────────────────────────────+                                           │
   |    Backfill / Hybrid Worker   |───────────────────────────────────────────┘
   | (Pull Model for Celebrities & | Generates next 50 posts on infinite scroll
   |  Cold User Timeline Backfill) |
   +───────────────────────────────+
                   │
                   ▼ Reads from RAM
   +───────────────────────────────────+
   |   User Recent Posts Cache (Redis) |
   |  (Key: user_posts:{author_id})    |
   +───────────────────────────────────+
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Content Moderation & Ingestion Pipeline

To prevent malicious, copyright-infringing, or toxic content from polluting feeds, posts undergo asynchronous pipelined sanitization:

```
User Submits Post
        │
        ▼
Content Service writes to Kafka [raw-posts]
        │
        ▼
Moderator Service Worker (Consumer)
        │
        ├── NLP Toxicity Filter (Text check)
        ├── Computer Vision / Hash-Match (NSFW / Copyright image check)
        └── Virality & Spam Rate Throttling
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
    [VIOLATION]             [APPROVED]
        │                       │
        ▼                       ▼
Write to [blocked-posts]    Write to [filtered-posts]
        │                       │
        ▼                       ├──► Post Consumer Svc ──► Writes Cassandra & S3
Notification Service            │
alerts Author: "Post Blocked"   └──► Fan-Out Worker ──► Evaluates Feed Ingestion
```

---

### 5.2 The Feed Generation Dilemma: Fan-out on Write vs. Fan-out on Read

#### Why Runtime Relational Querying Crashes
```sql
-- TRIVIAL RUNTIME JOIN (ANTI-PATTERN AT SCALE):
SELECT p.* FROM posts p
JOIN followers f ON p.author_id = f.following_id
WHERE f.follower_id = :current_user_id
ORDER BY p.created_at DESC LIMIT 20;
```
If a user follows $1,000$ accounts, the database must scan millions of candidate posts, sort them across network partitions, and execute disk seeks on every page reload. At $58,000\text{ QPS}$, this crashes database connection pools.

---

#### Comparative Analysis: Push vs. Pull Models

```
         FAN-OUT ON WRITE (PUSH MODEL)                  FAN-OUT ON READ (PULL MODEL)

Author Posts ──► Write to Kafka                 Author Posts ──► Write to Author Post Log
                       │                                                 │
                       ▼                                                 │ (Do nothing else)
             Fetch All Followers                                         │
                       │                                                 │
         ┌─────────────┼─────────────┐                                   │
         ▼             ▼             ▼                                   │
     User A        User B        User C                                  │
     Feed ZSET     Feed ZSET     Feed ZSET                               │
                                                User Opens App ──► Fetch Followed Accounts
     PRO: Read is instant O(1) in Redis                            Fetch Latest Posts from Each
     CON: Massive write amplification for                          Merge Sort in Memory
          celebrities (60M followers)!                  PRO: Zero write amplification
                                                        CON: Severe read latency spikes
```

1. **Fan-out on Write (Push Model)**:
   - When user $A$ posts, background workers identify all followers of $A$ and append the new `post_id` directly into each follower's precomputed **Redis Sorted Set**.
   - **Pros**: Read operation is instant $O(1)$ (`ZREVRANGEBYSCORE feed:{user_id} +inf -inf LIMIT 0 20`).
   - **Cons (Celebrity Hotspot / Fan-out Stampede)**: If an account with $50\text{ Million}$ followers (e.g., Cristiano Ronaldo) posts, the push worker must perform $50\text{ Million}$ Redis writes and Kafka events, overwhelming storage nodes.
2. **Fan-out on Read (Pull Model)**:
   - Posts are simply stored in the author's personal post list. Feeds are constructed on-demand when the reader opens the app.
   - **Pros**: Zero write amplification for celebrities.
   - **Cons**: High read latency. A user following $500$ people must fetch $500$ separate timelines and merge-sort them in real time.

---

### 5.3 The Production Hybrid Fan-Out Architecture (Bunny's Pattern)

To achieve the best of both worlds, Bunny architects a **Hybrid Fan-Out Engine**:

```
                                  Post Approved by Moderator
                                               │
                                               ▼
                           Check Author Status: Is Author a Celebrity?
                                  (e.g., Followers > 25,000)
                                               │
                       ┌───────────────────────┴───────────────────────┐
                       ▼ NO                                            ▼ YES
        +─────────────────────────────+                 +─────────────────────────────+
        |  Standard User: PUSH Path   |                 |    Celebrity: PULL Path     |
        |  Fan-out on Write Worker    |                 |   Skip Fan-out to Followers |
        +─────────────────────────────+                 +─────────────────────────────+
                       │                                               │
                       ▼                                               ▼
          Push post_id to Follower's                      Write to Author's Recent
          Redis Feed Cache ZSET                           Posts Cache (user_posts:{id})
                       │                                               │
                       └───────────────────────┬───────────────────────┘
                                               │
                                               ▼
                         Reader Opens App: Feed Service Intercepts
                                               │
        ┌──────────────────────────────────────┴──────────────────────────────────────┐
        ▼                                                                             ▼
Read Precomputed Standard Feeds                               Pull Recent Posts from Followed
from Redis (feed:{user_id})                                   Celebrities from In-Memory Cache
        │                                                                             │
        └──────────────────────────────────────┬──────────────────────────────────────┘
                                               │
                                               ▼
                                 K-Way Merge Sort in Memory
                                 (Rank by Timestamp / ML Score)
                                               │
                                               ▼
                                 Stream Top 20 to User Screen
```

#### Redis Data Structures for Feeds
- **Feed Cache (`feed:{user_id}`)**:
  - Redis **Sorted Set (ZSET)**.
  - Member: `post_id` (string).
  - Score: Unix timestamp (float or integer epoch millisecond).
  - Kept capped at the latest $200\text{ posts}$ using `ZREMRANGEBYRANK feed:{user_id} 0 -201`.
- **Author Recent Posts Cache (`user_posts:{author_id}`)**:
  - Redis Sorted Set storing the author's last $100\text{ posts}$.

---

### 5.4 Infinite Scroll & Backfill Worker Service

What happens when a reader scrolls past the initial 100 precomputed posts?

```
User Scrolls Past Item 100
        │
        ▼
Client hits GET /api/v1/feed?cursor=last_post_id
        │
        ▼
Feed Service detects cache exhausted for active user
        │
        ▼
Triggers Async Backfill Service Worker
        │
        ├── 1. Reads user's top followed friends from Redis "Top Follower Cache"
        ├── 2. Scans author recent posts from Redis "User Recent Posts Cache"
        └── 3. Merges candidate posts using an in-memory Min-Heap (K-Way Merge)
                    │
                    ▼
Appends next 50-100 post IDs into Redis feed:{user_id} with 24-hour TTL
                    │
                    ▼
Returns 20 items to user; Infinite scroll remains seamless!
```

---

### 5.5 Database Choice & Schemas

#### Posts Database: Apache Cassandra / ScyllaDB
Bunny selects Cassandra because post publishing is write-heavy, horizontally scalable, and requires no relational table joins.

```cql
CREATE KEYSPACE social_network WITH replication = {
    'class': 'NetworkTopologyStrategy', 
    'us-east': 3, 
    'us-west': 3
};

-- Posts Table Partitioned by Author
CREATE TABLE social_network.posts (
    author_id uuid,
    post_id timeuuid,
    post_type text, -- 'TEXT', 'IMAGE', 'VIDEO'
    content_text text,
    media_url text,
    thumbnail_url text,
    like_count counter,
    comment_count counter,
    created_at timestamp,
    PRIMARY KEY (author_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

-- Post Lookup by Post ID (Global Query Table)
CREATE TABLE social_network.posts_by_id (
    post_id timeuuid PRIMARY KEY,
    author_id uuid,
    content_text text,
    media_url text,
    thumbnail_url text,
    created_at timestamp
);
```

#### Social Graph Database: PostgreSQL / MySQL
For managing follower relationships with transactional integrity and indexing:

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_celebrity BOOLEAN DEFAULT FALSE,
    follower_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE followers (
    follow_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    follower_id UUID NOT NULL REFERENCES users(user_id),
    following_id UUID NOT NULL REFERENCES users(user_id),
    status VARCHAR(20) DEFAULT 'ACCEPTED',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (follower_id, following_id)
);
CREATE INDEX idx_follower_following ON followers (following_id, follower_id);
CREATE INDEX idx_follower_user ON followers (follower_id, following_id);
```

---

### 5.6 Engagement Counters & Write Aggregation

High-profile posts receive tens of thousands of likes per second. Directly updating `like_count = like_count + 1` in SQL or Cassandra on every click causes row contention.

```
Client Clicks "Like"
        │
        ▼
API Gateway emits to Kafka [engagement-events]
        │
        ▼
Redis In-Memory Counter Ingestion
(HINCRBY post:likes:counters pst_881902a 1)
        │
        ▼
Flushed in Batches Every 10 Seconds
UPDATE posts SET like_count = like_count + :batch_increment WHERE post_id = ...
```

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: Why use fanout on write instead of fanout on read for normal users?

Read-heavy optimization: Social media has 100:1 read/write ratio. Users read feeds 100x more than posting. Fanout on write:

- **Point 1**: Fast reads - feed pre-generated in Redis, LRANGE returns 20 posts in <10ms,

- **Point 2**: Predictable latency - post creation takes 200ms to fanout to 500 followers, but every feed load is <50ms,

- **Better user experience - users don't wait for feed generation, instant scroll. Trade-off**:
  Write amplification (1 post = 500 writes) vs slow reads (query 100 users' posts, sort, merge = 500ms). For celebrities with 10M followers, fanout on write is impractical (1 post = 10M writes, takes minutes), so we use pull model where fans query celebrity's posts on-demand.

---

### Q2: How do you handle the celebrity/hotspot problem in feed generation?

Multi-tier approach:

- **Point 1**: User classification - Flagging users as celebrity (>5K followers) in User table,

- **Point 2**: Hybrid fanout - Push model for <5K followers (precompute feeds), Pull model for >5K (fetch on-demand),

- **Point 3**: Feed merging - During feed load, combine pushed posts (from normal users) + pulled posts (from celebrities) and sort by timestamp,

- **Point 4**: Caching - Celebrity posts cached separately in Redis with higher TTL (1 hour vs 10 min), all followers share same cache,

- **Pagination - Celebrity timelines paginated, followers see latest 100 posts max. Example**:
  User follows 90 normal users (pushed feeds) + 10 celebrities (pulled). Feed Service: LRANGE feed:{user_id} for pushed, fetch celebrity posts from PostDB, merge and rank. Performance: Combined latency 100ms (50ms pushed + 50ms pulled) vs pure pull would be 500ms.

---

### Q3: How do you ensure consistency between like counts in cache vs database?

Eventual consistency with reconciliation:

- **Write path - user likes post → write to Like DB (Cassandra), increment Redis counter INCR likes_count**:
  {post_id},

- **Point 2**: Background sync - cron job runs every 5 min, queries Like DB for actual count (SELECT COUNT(*) GROUP BY post_id), updates PostDB with authoritative count,

- **Point 3**: Cache invalidation - if Redis count differs from DB by >10%, invalidate cache and reload from DB,

- **Conflict resolution - DB is source of truth, Redis is best-effort cache. Edge cases**:
  Redis evicts key due to memory pressure → next read fetches from DB, subsequent reads cache again. Redis crashes → all counters lost → background job rebuilds from DB within 5 min. Accept window: Counters may be off by ±5% for <5 min, acceptable for social media (not financial system). Alternative: Use Redis persistence (AOF + RDB) for durability, but adds write latency 10ms → 50ms.

---

### Q4: What happens if a user with 1M followers posts? How do you prevent system overload?

Rate limiting + async processing:

- **Point 1**: Classification - user_id tagged as 'celebrity' if followers_count > 5K, stored in User table,

- **Point 2**: Bypass fanout - celebrity posts skip push fanout entirely, stored only in PostDB,

- **Point 3**: Pull model - when follower loads feed, query celebrity's recent posts (last 100) from PostDB, merge with normal pushed posts,

- **Point 4**: Async notification - instead of 1M push notifications, send to notification aggregator, batch into 'trending' notifications,

- **Rate limiting - limit celebrity to 10 posts/hour to prevent spam. Implementation**:
  Feed Service checks if poster.followers_count > 5K, if yes: write to PostDB only, publish 'celebrity.posted' event to Kafka, skip fanout worker. Followers see post when they load feed (pull), not pushed to their feeds. Notification Service sends to 1000 most engaged followers immediately, rest get aggregated daily digest. Prevents: 1M database writes, 1M Redis writes, 1M push notifications from overwhelming system.

---

### Q5: How do you handle nested comment threads efficiently?

Hierarchical storage with lazy loading:

- **Point 1**: Database schema - Comment table has parent_comment_id (NULL for top-level, UUID for replies), creates tree structure,

- **Point 2**: Loading strategy - fetch top-level comments first (WHERE parent_comment_id IS NULL LIMIT 20), load replies only when user expands thread (WHERE parent_comment_id = {comment_id}),

- **Point 3**: Depth limit - max 3 levels (post → comment → reply → no further nesting) to prevent UI complexity,

- **Point 4**: Denormalization - each comment stores reply_count for 'Show 5 replies' link without counting,

- **Sorting - top-level sorted by likes_count DESC (best first), replies sorted by created_at ASC (chronological). Query optimization**:
  Composite index on (post_id, parent_comment_id, created_at) enables fast queries. Alternative: Nested set model or closure table for complex hierarchies, but adds write complexity. Example: Post has 10K comments → fetch top 20, user expands comment #5 → fetch replies WHERE parent_comment_id = comment_5_id LIMIT 10, lazy load 'Show more' pagination.

---

### Q6: How do you implement real-time notifications without polling?

WebSocket + pub/sub pattern:

- **Point 1**: Connection - client opens WebSocket to Notification Service on app launch, authenticated via JWT,

- **Subscription - server subscribes to Redis pub/sub channel notifications**:
  {user_id},

- **Event flow - when user A likes user B's post, Engagement Service publishes to Redis channel 'notifications**:
  {user_B_id}',

- **Point 4**: Push - Notification Service listening on that channel receives event, sends JSON over WebSocket to user B's client,

- **Fallback - if client offline, store notification in Notification DB, deliver when client reconnects. Scalability**:
  1M concurrent WebSocket connections = 1000 servers (1K connections each), Redis pub/sub handles 100K messages/sec, load balanced using consistent hashing on user_id. Alternative: Server-Sent Events (SSE) for one-way streaming (simpler), or long polling (fallback for old browsers). Battery optimization: Mobile apps use FCM/APNS push notifications instead of persistent WebSocket, WebSocket only for active users in-app.

---

### Q7: How do you handle media upload failures and retries?

Multi-stage upload with resumability:

- **Point 1**: Presigned URL - client requests POST /api/v1/posts/upload/init, server generates presigned S3 PUT URL (valid 15 min), returns {upload_id, presigned_url},

- **Point 2**: Chunked upload - for large files (>100MB), client uses multipart upload, splits into 5MB chunks, uploads in parallel (10 concurrent),

- **Progress tracking - each chunk upload stores progress in Redis upload**:
  {upload_id} = {chunks_uploaded: [1,2,3], total_chunks: 20},

- **Point 4**: Retry - on failure, client queries GET /api/v1/posts/upload/{upload_id}/status, server returns missing chunks, client uploads only missing parts,

- **Finalize - after all chunks uploaded, client calls POST /api/v1/posts/upload/{upload_id}/complete, server assembles chunks in S3, creates post. Timeouts**:
  Presigned URL expires in 15 min, upload_id expires in 24 hours (Redis TTL). Network resilience: Exponential backoff (1s, 2s, 4s), circuit breaker after 3 consecutive failures. Alternative: Direct S3 multipart upload using AWS SDK, but loses control over progress tracking.

---

### Q8: What's your database partitioning strategy for Posts at scale?

Multi-dimensional sharding:

- **Point 1**: Primary partitioning - Cassandra partitions by user_id (partition key), distributes user's posts across cluster based on hash(user_id), each node handles 1000 users,

- **Point 2**: Clustering - within partition, sort by (created_at DESC, post_id) for efficient time-range queries,

- **Point 3**: Global access - separate PostDB_indexed table with post_id as primary key for direct post lookup (GET /posts/{post_id}),

- **Replication - RF=3 for high availability, QUORUM consistency for reads/writes. Query patterns**:
  User timeline (SELECT * FROM posts WHERE user_id = {id} LIMIT 20) hits single partition = fast. Feed generation (SELECT * FROM posts WHERE user_id IN (100 followees)) fans out to 100 partitions = slow, mitigated by fanout on write. Hotspots: Celebrity users have high partition load, mitigated by pull model (followers query celebrity partition, not written to follower feeds). Scaling: Add nodes increases partition count from 1000 to 2000, Cassandra rebalances automatically using consistent hashing.

---

### Q9: How do you prevent duplicate likes when user clicks multiple times?

Multi-layer idempotency:

- **Point 1**: Client-side - debounce like button 300ms, disable button immediately on click (optimistic UI),

- **API level - check Redis cache like**:
  {user_id}:{post_id}, if exists return 200 OK (already liked),

- **Point 3**: Database level - Cassandra composite primary key (post_id, user_id) enforces uniqueness, duplicate insert fails silently,

- **Transaction - atomic check-and-set in Redis**:
  if SETNX like:{user_id}:{post_id} success → write to DB + INCR counter, if fails → already liked,

- **Race condition - if 2 parallel requests both pass Redis check (race), Cassandra constraint prevents duplicate, one request succeeds, other gets 409 Conflict. Implementation**:
  POST /api/v1/posts/{post_id}/like checks Redis, if not exists: start transaction, write to Like DB, set Redis key (TTL=24h), increment counter, commit. Counters: Use INCR for increment, DECR for unlike, background job syncs actual count from DB every 5 min to fix any drift. Edge case: Redis eviction loses idempotency cache → user can re-like → DB constraint prevents duplicate row, counter incremented correctly.

---

### Q10: How do you implement privacy controls (public vs friends-only posts)?

Multi-layer visibility filtering:

- **Point 1**: Storage - Post table has visibility enum (public, friends, private),

- **Feed generation - during fanout on write, check relationship between poster and follower, only push to feeds if**:
  visibility=public OR (visibility=friends AND is_mutual_follower),

- **Direct access - GET /posts/{post_id} checks**:
  if public → allow, if friends → verify requester.user_id in poster's friends list, if private → only author can view,

- **Feed filtering - pull model queries**:
  SELECT * FROM posts WHERE user_id IN (followees) AND (visibility = 'public' OR (visibility = 'friends' AND mutual = true)),

- **Caching - privacy checks cached in Redis friend**:
  {user_A}:{user_B} = boolean (TTL=1 hour). Friend determination: Mutual follow check - query Followers table for bidirectional edge (A follows B AND B follows A). Performance: Privacy checks add 10ms latency, mitigated by batch checking (verify 100 posts in single Redis MGET). Alternative: Facebook-style fine-grained lists (close friends, acquaintances, custom lists), stored as relationship tags in graph database.

---

## 7. Architectural Trade-Off Analysis Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Feed Generation Model** | **Hybrid Fan-Out (Push for regular, Pull for celebrities)** | Pure Fan-Out on Write (Push) | Pure push causes severe write amplification stampedes when celebrities post (60M writes per post). Hybrid prevents hot-partition crashes. |
| **Feed Storage Tier** | **In-Memory Redis Sorted Sets (ZSET)** | Dynamic Relational SQL Join (`JOIN followers`) | SQL joins across 1,000 friends collapse under 150k read QPS. Redis serves precomputed feeds in $< 10\text{ ms}$ using time-ranked ZSETs. |
| **Post Storage Database** | **Apache Cassandra / ScyllaDB** | Relational Database (PostgreSQL) | Social posts are immutable, write-heavy timeseries data. Cassandra provides linear horizontal write scaling with clustering keys ordered by `post_id DESC`. |
| **Content Moderation Pipeline** | **Asynchronous Kafka Pipeline (`[raw-posts]` $\to$ Moderator $\to$ `[filtered-posts]`)** | Synchronous Blocking API Call | Deep learning moderation models take 100–300 ms. Synchronous execution blocks mobile clients and degrades upload latency. Async queues decouple validation. |
| **Pagination Strategy** | **Cursor-Based Pagination (`timeuuid` / epoch)** | Offset-Based Pagination (`OFFSET 40 LIMIT 20`) | Offset pagination skips rows on disk $O(N)$ and suffers from page drifting when new posts are inserted mid-scroll. Cursor pagination provides consistent $O(1)$ seeks. |
| **Social Graph DB** | **Relational PostgreSQL (with indexes)** | Graph Database (Neo4J) | Unless the scope requires deep multi-hop traversal (e.g., 4th-degree friend recommendations), PostgreSQL handles 1-hop follower lookups with vastly lower operational complexity. |
