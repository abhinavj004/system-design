# System Design 04: Design OTT Platform (Netflix / Prime Video / Hotstar)

> **Source**: [Interview with Bunny - Episode 4](https://www.youtube.com/watch?v=NH-f955mvGg)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/04](https://www.interviewwithbunny.com/systemdesign/04)  
> **Topic**: Designing a Global Video Streaming Architecture (Netflix, Hotstar, Amazon Prime Video)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **User Onboarding & Subscription Plans**: Users can register, log in, and purchase subscription tiers (Free, Basic, Premium 4K) enforcing device and concurrent stream limits.
2. **Catalog Browsing & Search**: Users can search for movies/shows by title, genre, actor, language, and view personalized recommendation rows (Trending, Continue Watching).
3. **Video Ingestion & Transcoding Pipeline**: Content studios can upload high-resolution master video files. The system automatically slices, transcodes, and encodes them into multiple bitrate tiers.
4. **Adaptive Bitrate Video Streaming**: Users can stream video seamlessly across diverse devices (TV, mobile, desktop) and fluctuating network speeds with minimal buffering.
5. **Playback State & Resume**: The system continuously tracks viewing progress so users can pause on one device (e.g., Mobile) and resume from the exact second on another (e.g., Smart TV).

---

### 1.2 Non-Functional Requirements (NFR)
1. **Ultra-Low Startup Latency & Zero Stalling**: Video playback must start within $< 2\text{ seconds}$, with adaptive transitions between quality tiers without video pauses.
2. **High Availability (99.99%)**: Streaming and catalog browsing must be fault-tolerant across global regions.
3. **Massive Scale & Bandwidth Optimization**:
   - Capable of serving 100M+ concurrent viewers (e.g., live sports on Hotstar or blockbuster season drops on Netflix).
   - Minimizing origin server egress costs by serving $> 95\%$ of video data from edge caching networks.
4. **Content Security & DRM**: Enforce strict Digital Rights Management (Widevine, FairPlay) and signed access tokens to prevent unauthorized streaming or scraping.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)
- **Active User Base**: 200 Million Global Subscribers.
- **Daily Active Viewers**: 50 Million DAU.
- **Average Streaming Time**: 2 hours/day per user.
- **Average Video Bitrate**: 3 Mbps (blended average across 1080p, 720p, 480p).
  $$\text{Total Daily Egress} = 50\text{M} \times 2\text{ hours} \times 3600\text{s} \times 3\text{ Mbps} \approx 1,080 \text{ Petabits} \approx 135 \text{ Petabytes / day}$$
- **Peak Egress Bandwidth**:
  $$\text{Peak Bandwidth} \approx \frac{135 \text{ PB} \times 8}{86400 \times 0.5 \text{ (peak distribution)}} \approx 25 \text{ Tbps (Terabits per second)}$$
- **Architecture Implication**:
  - **No cloud origin datacenter can sustain 25 Tbps directly without collapsing or incurring millions in egress bills.**
  - **Over 95% of video chunks must be served via Edge CDNs and ISP-partnered caching appliances (Netflix Open Connect model).**

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+       +--------------------+       +--------------------+
|        User        |       |    Subscription    |       |       Video        |
+--------------------+       +--------------------+       +--------------------+
| id (UUID)          | 1   1 | id (UUID)          | 1   * | id (UUID)          |
| email, name, pwd   |<----->| user_id (FK)       |<----->| title, description |
| active_plan_id (FK)|       | plan_type (4K/HD)  |       | duration_secs(INT) |
+--------------------+       | max_streams (INT)  |       | release_year (INT) |
                             | expires_at (TIME)  |       | maturity_rating    |
                             +--------------------+       +--------------------+
                                                                     | 1
                                                                     | *
+--------------------+       +--------------------+       +--------------------+
|    VideoSegment    | *   1 |   VideoManifest    | 1   * |  UserWatchProgress |
+--------------------+<----->+--------------------+<----->+--------------------+
| id (UUID)          |       | id (UUID)          |       | user_id (PK)       |
| manifest_id (FK)   |       | video_id (FK)      |       | video_id (PK)      |
| resolution (1080p) |       | protocol(HLS/DASH) |       | resume_time_secs   |
| segment_index (INT)|       | manifest_url (.mpd)|       | last_watched_at    |
| duration_secs (10s)|       | audio_tracks (JSON)|       | is_completed (BOOL)|
| s3_blob_url (TEXT) |       +--------------------+       +--------------------+
+--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Get Video Details & Streaming Authorization
```http
GET /api/v1/videos/{video_id}/playback
Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "video_id": "vid_stranger_things_s4e1",
  "title": "Chapter One: The Hellfire Club",
  "duration_secs": 4560,
  "manifest": {
    "protocol": "MPEG-DASH",
    "manifest_url": "https://cdn.netflix.com/manifests/st_s4e1.mpd?token=exp_9981_hmac_abc",
    "drm": {
      "type": "widevine",
      "license_server_url": "https://drm.netflix.com/v1/license"
    }
  },
  "resume_position_secs": 1420 // Pulled from Redis watch progress
}
```

---

### 3.2 Update Viewing Progress (Heartbeat)
```http
POST /api/v1/videos/{video_id}/progress
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "timestamp_seconds": 1430,
  "device_id": "apple_tv_living_room"
}

Response: 204 No Content
```
> **Bunny's Optimization**: The video player fires this lightweight heartbeat every 10 seconds. It is written directly to a Redis key-value store, enabling seamless cross-device resumption.

---

### 3.3 Studio Raw Video Upload (Internal Ingestion)
```http
POST /api/v1/internal/videos/ingest
Authorization: Bearer <studio_admin_key>

Request Body:
{
  "title": "Squid Game Season 2 Episode 1",
  "raw_s3_uri": "s3://netflix-raw-masters/sg_s2e1_prores422.mov",
  "source_resolution": "4K_HDR"
}

Response: 202 Accepted
{
  "job_id": "job_transcode_8819",
  "status": "PROCESSING"
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Studios / Creators ]
          │ (1. Upload Master 50GB file)
          ▼
┌─────────────────────────────────┐
│     S3 Ingest Master Bucket     │
└─────────────────────────────────┘
          │ (2. Trigger Transcode)
          ▼
┌─────────────────────────────────┐
│  Distributed Transcoding Engine │ (FFmpeg Workers)
│  (Slices video into 10s chunks; │
│   Transcodes to 4K, 1080p, 720p)│
└─────────────────────────────────┘
          │ (3. Save Transcoded Chunks & Manifests)
          ▼
┌─────────────────────────────────┐
│   Processed S3 Storage Bucket   │
│   (.mpd, .m3u8, .ts, .m4s chunks)│
└─────────────────────────────────┘
          │ (4. Pre-seed & Cache hot segments)
          ▼
┌─────────────────────────────────────────────────────────────┐
│             Global CDN & Open Connect ISP Nodes             │
│            (Edge Caching of 2s-10s Video Chunks)            │
└─────────────────────────────────────────────────────────────┘
          ▲
          │ (5. Stream Chunks over HTTP via Adaptive Bitrate)
          │
[ Client Video Players (TV, Mobile, Web) ]
          │
          │ (6. Browse, Auth, Heartbeat Progress)
          ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway / Load Balancer                 │
└─────────────────────────────────────────────────────────────┘
          │                            │              │
          ▼                            ▼              ▼
┌──────────────────┐          ┌────────────────┐ ┌────────────────┐
│  User & Billing  │          │ Video Metadata │ │ Playback State │
│     Service      │          │    Service     │ │    Service     │
└──────────────────┘          └────────────────┘ └────────────────┘
          │                            │              │
          ▼                            ▼              ▼
┌──────────────────┐          ┌────────────────┐ ┌────────────────┐
│  PostgreSQL DB   │          │  Cassandra DB  │ │  Redis Cluster │
│ (Subscriptions)  │          │(Catalog & Tags)│ │(Active Progress│
└──────────────────┘          └────────────────┘ └────────────────┘
                                                      │ (Async Flush)
                                                      ▼
                                                 ┌────────────────┐
                                                 │ Cassandra DB   │
                                                 │(Watch History) │
                                                 └────────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 The Video Ingestion & Transcoding Pipeline
Raw studio videos are uncompressed files (50–100 GB). Serving them directly is impossible.

```
[ Raw Video (50GB) ] ──▶ [ Chunker ] ──▶ Split into 10-second segments
                                               │
               ┌───────────────────────────────┴───────────────────────────────┐
               ▼                               ▼                               ▼
       [ Worker Node 1 ]               [ Worker Node 2 ]               [ Worker Node 3 ]
       Transcode: 4K (20 Mbps)         Transcode: 1080p (5 Mbps)       Transcode: 720p (2.5 Mbps)
       Codec: AV1 / HEVC               Codec: H.264                    Codec: H.264
               │                               │                               │
               └───────────────────────────────┬───────────────────────────────┘
                                               ▼
                         [ Manifest Generator (.mpd / .m3u8) ]
```

1. **Chunking**: The video is sliced into **uniform time chunks** (typically 6 to 10 seconds each).
2. **Parallel Transcoding**: Independent worker pods transcode segments into multiple profiles:
   - **4K UHD** (3840x2160) @ 15–25 Mbps (HEVC/AV1)
   - **1080p Full HD** (1920x1080) @ 4–6 Mbps (H.264)
   - **720p HD** (1280x720) @ 2–3 Mbps (H.264)
   - **480p SD** (854x480) @ 1 Mbps (H.264)
   - **360p Low** (640x360) @ 400 Kbps (H.264 for poor mobile connectivity)
3. **Manifest Creation**: Creates an index mapping time segments to specific URLs:
   - **MPEG-DASH (`.mpd`)**: Modern XML-based standard favored by Netflix, Prime, YouTube.
   - **HLS (`.m3u8`)**: Apple's standard HTTP Live Streaming protocol.

---

### 5.2 Adaptive Bitrate Streaming (ABR) in Action
How does playback remain smooth when a user travels into a low-signal area?

```
Timeline:
[ 0s - 10s ]  ──▶ Network Speed = 25 Mbps ──▶ Player requests Chunk 1 in 1080p
[ 10s - 20s ] ──▶ Network Speed = 20 Mbps ──▶ Player requests Chunk 2 in 1080p
[ 20s - 30s ] ──▶ Network Drops to 2 Mbps ──▶ Player requests Chunk 3 in 480p (Seamless drop)
[ 30s - 40s ] ──▶ Network Recovers 15 Mbps──▶ Player requests Chunk 4 in 720p/1080p
```
- The client-side video player (ExoPlayer, AVPlayer, Shaka Player) continuously measures:
  1. **Download time of the last chunk**.
  2. **Current video buffer health** (e.g., "Do I have 30 seconds of pre-buffered video in RAM?").
- If buffer level drops below a safety threshold, the player requests the **next 10-second chunk in a lower bitrate tier**. The user experiences a momentary drop in clarity, but **playback never halts or buffers**.

---

### 5.3 CDN & The Netflix Open Connect Model
Why generic commercial CDNs (Akamai, Cloudflare) are insufficient for massive streaming:
1. **Open Connect Appliances (OCAs)**:
   - Netflix builds custom rack servers loaded with petabytes of SSD/HDD storage.
   - They deploy these OCAs directly inside thousands of **Internet Service Provider (ISP)** networks across the globe for free.
2. **Proactive Pre-Positioning**:
   - During off-peak hours (e.g., 2:00 AM – 5:00 AM), Netflix pre-seeds upcoming shows (e.g., *Stranger Things*) directly into local ISP caches.
   - When users press "Play" in the evening, $> 95\%$ of video traffic travels only from their local ISP router down the street, avoiding trans-continental internet backbones.

---

### 5.4 Playback Progress & "Continue Watching" Engine
1. **Hot In-Memory State (Redis)**:
   - Client sends heartbeat every 10 seconds:
     ```redis
     HSET user:u_101:watch_progress vid_stranger_things_s4e1 1430
     ```
   - Returning to the homepage reads this Redis hash in $< 2\text{ms}$ to populate the "Continue Watching" row.
2. **Cold Durable Storage (Apache Cassandra)**:
   - An asynchronous worker flushes Redis progress records to Cassandra in batches every 5 minutes:
     ```cql
     CREATE TABLE user_watch_history (
         user_id uuid,
         video_id uuid,
         last_position_seconds int,
         updated_at timestamp,
         completed boolean,
         PRIMARY KEY (user_id, video_id)
     );
     ```

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How does adaptive bitrate streaming work in Netflix/YouTube?

Adaptive bitrate streaming (ABR) dynamically adjusts video quality based on network conditions:

- **Video preparation**:
  Original video transcoded into multiple resolutions (4K/2160p at 20 Mbps, 1080p at 8 Mbps, 720p at 5 Mbps, 480p at 2.5 Mbps, 360p at 1 Mbps) using FFmpeg/MediaConvert,

- **Segmentation**:
  Each resolution split into 10-second chunks (segments),

- **Manifest file**:
  HLS master.m3u8 or DASH master.mpd lists all available resolutions with bitrates,

- **Player logic**:
  (a) Client requests manifest, (b) Measures current bandwidth by timing segment downloads: bandwidth = segment_size / download_time, (c) Selects appropriate resolution: if bandwidth > 8 Mbps → request 1080p segments, 5-8 Mbps → 720p, 2.5-5 Mbps → 480p, <2.5 Mbps → 360p, (d) Downloads segments sequentially, (e) Monitors buffer level (20-40 sec ahead), (f) If bandwidth drops or buffer depletes → switch to lower resolution for next segment,

- **Seamless switching**:
  Segments aligned on keyframes (I-frames at 0th second), player switches between playlists mid-playback without rebuffering,

- **Quality metrics**:
  Track initial buffering time, rebuffering events, bitrate switches (too frequent = poor UX). Example: User on WiFi (8 Mbps) → Player starts with 1080p → WiFi degrades to 3 Mbps → Player detects slow download after 2-3 segments → Switches to 480p → User continues watching without stall. Key insight from image: 'Switching happens only if keyframe is set at 0th second, at this keyframe point depending on client bandwidth, player either can resume from old video format

- **or some or different resolution.' HLS vs DASH**:
  HLS uses .m3u8 + .ts segments (Apple standard), DASH uses .mpd + .m4s segments (MPEG standard, codec agnostic). Production: Serve both HLS (for Apple devices) and DASH (for others).

---

### Q2: Explain the video upload and transcoding pipeline end-to-end.

End-to-end video processing flow:

- **Upload**:
  Content creator uploads raw video (e.g., movie.mp4, 50GB 4K) + metadata {title, description, genre, actors, duration},

- **Uploader Service**:
  (a) Generates video_id (UUID), (b) Multipart upload to S3: s3://videos-raw/{video_id}/original.mp4 (parallel chunks for faster upload), (c) Stores metadata: INSERT INTO video_metadata (video_id, title, processing_status='UPLOADING'), (d) On upload completion → Publish Kafka event 'video.uploaded' with {video_id, s3_path},

- **Transcoding Pipeline**:
  (a) Lambda/Worker consumes Kafka event, (b) Downloads original from S3, (c) FFmpeg/AWS MediaConvert transcodes to multiple resolutions: 4K (3840×2160, H.265, 20 Mbps), 1080p (1920×1080, H.264, 8 Mbps), 720p (1280×720, 5 Mbps), 480p (2.5 Mbps), 360p (1 Mbps), (d) For EACH resolution: Encode video, Split into 10-sec segments (.ts for HLS or .m4s for DASH), Upload to S3: s3://videos-processed/{video_id}/{resolution}/segment_001.ts, ..., (e) Generate manifest files: Master.m3u8 (points to all resolution playlists), 1080p/playlist.m3u8 (lists segment URLs for 1080p), (f) Upload manifests to S3,

- **Metadata Update**:
  UPDATE video_metadata SET processing_status='COMPLETED', manifest_url='s3://.../master.m3u8', resolutions=['4K','1080p','720p','480p','360p'],

- **Search Indexing**:
  Kafka CDC event → Elasticsearch indexes video for search,

- **CDN Distribution**:
  CloudFront pulls segments from S3, caches at edge locations globally. Transcoding optimizations: (a) Parallel processing: Each resolution transcoded on separate GPU instance, total time ~30 min for 2-hour movie, (b) Spot instances for cost savings (transcoding is batch, interruptible), (c) Two-pass encoding for better quality, (d) Auto-scale workers based on SQS queue depth (10-100 instances). Storage: Original 4K (24GB) + Transcoded (4K+1080p+720p+480p+360p ≈ 16GB) = 40GB per 1-hour video.

---

### Q3: How do you minimize video buffering and ensure smooth playback?

Multi-layer buffering prevention strategy:

- **Adaptive bitrate**:
  Automatically reduce quality when bandwidth drops (8 Mbps → 1080p, 3 Mbps → 480p), prevents stalling during network fluctuations,

- **Buffering strategy**:
  Player maintains 20-40 sec buffer (downloads 3-5 segments ahead), if buffer <10 sec → aggressive switch to lower resolution + increase segment prefetch,

- **CDN edge caching**:
  Segments cached at 200+ CloudFront PoPs globally, 95% cache hit rate, <50ms latency vs 200ms from origin,

- **Segment size**:
  10-sec segments (vs 30-sec) enable faster quality switching and seeking, smaller download chunks reduce impact of network interruptions,

- **Preloading**:
  On video start, download first 2-3 segments at lower quality for instant playback, then switch to optimal quality,

- **Connection optimization**:
  HTTP/2 multiplexing for parallel segment downloads, connection keep-alive reduces handshake overhead,

- **Predictive prefetching**:
  If user watching series, preload first 2 segments of next episode,

- **Network condition monitoring**:
  Player tracks: segment download time, throughput (Mbps), buffer level, packet loss → adjusts quality proactively,

- **Fallback quality**:
  If multiple segments fail to download at current quality, immediately drop 2 levels (1080p → 480p instead of gradual 1080p → 720p → 480p),

- **CDN failover**:
  If edge location errors (5xx), automatically retry from different PoP or origin shield,

- **Client-side optimization**:
  Hardware decoding (GPU) for H.264/H.265 reduces CPU load, video.js/hls.js optimized players with smart buffering logic. Quality metrics tracked: Initial buffering time (time to first frame, target <2 sec), Rebuffering ratio (time spent buffering / total watch time, target <0.5%), Bitrate switches (too frequent = annoying, target <5 switches per hour). Example scenario: User starts watching → Player requests 1080p → First 2 segments download fast (WiFi, buffer at 20 sec) → User moves, WiFi weakens → Segment 3 download slow (detected) → Player switches to 720p for segment 4 → Buffering continues uninterrupted → User never sees spinner.

---

### Q4: How do you design the CDN architecture for global video delivery?

Multi-tier CDN architecture for global scale:

- **Origin**:
  S3 buckets in multiple regions (us-east-1, eu-west-1, ap-south-1) with cross-region replication, stores transcoded video segments,

- **Origin Shield**:
  CloudFront feature, centralized caching layer between edge and origin, acts as 'shield' to collapse requests (100 edge requests → 1 shield request to origin), reduces origin load by 90%, improves cache hit rate,

- **Edge Locations**:
  200+ CloudFront PoPs globally (NYC, London, Mumbai, Tokyo, etc.), cache segments with TTL=24 hours, serve 95% of requests without origin hit,

- **Request flow**:
  User in Mumbai requests segment → DNS routes to nearest edge (Mumbai PoP) → Edge checks cache (cache key: video_id/resolution/segment_number) → If HIT: Return cached segment (10-50ms) → If MISS: Request from Origin Shield (us-east-1) → Shield checks cache → If miss: Fetch from S3 → Shield caches + returns → Edge caches + returns (200ms first time, <50ms subsequent),

- **Cache optimization**:
  (a) Popular content (trending movies): Pre-warmed to edges before release (push invalidation), (b) Long-tail content: Cached on-demand (first request slower, subsequent fast), (c) Purge strategy: Invalidate segments only when video updated (rare), otherwise rely on TTL,

- **Geo-routing**:
  S3 buckets per region (us-west-2, eu-west-1, ap-south-1), CloudFront fetches from nearest origin based on edge location, reduces latency by 60% (500ms → 200ms for origin fetches),

- **Signed URLs**:
  CloudFront signed URLs with expiry=4 hours prevent hotlinking and unauthorized access, signature validates user has active subscription,

- **Compression**:
  Gzip/Brotli for manifest files (.m3u8, .mpd) saves 70% bandwidth, not for video segments (already compressed),

- **HTTP/2**:
  Multiplexing allows parallel segment downloads over single connection, reduces handshake overhead,

- **Monitoring**:
  CloudWatch tracks: cache hit rate (target >95%), origin requests (target <5%), error rate (4xx, 5xx <0.1%), bandwidth usage. Cost calculation: 20M DAU × 2 hours × 5 Mbps / 8 = 20M × 2 × 5/8 GB = 25M GB/day = 750M GB/month → 750M GB × $0.085/GB = $63M/month in CDN bandwidth. Optimization: Cache hit rate 95% → only 5% hits origin → $63M × 0.05 = $3.15M origin bandwidth cost.

---

### Q5: How do you implement video search with filters (genre, actor, language)?

Elasticsearch-based video search with filters:

- **Index mapping**:
  PUT /videos { 'mappings': { 'properties': { 'video_id': {'type': 'keyword'}, 'title': {'type': 'text', 'analyzer': 'standard', 'fields': {'keyword': {'type': 'keyword'}}}, 'description': {'type': 'text'}, 'genre': {'type': 'keyword'}, 'actors': {'type': 'text', 'fields': {'keyword': {'type': 'keyword'}}}, 'director': {'type': 'text'}, 'language': {'type': 'keyword'}, 'content_rating': {'type': 'keyword'}, 'release_date': {'type': 'date'}, 'popularity_score': {'type': 'float'} } } },

- **Indexing**:
  Video metadata from PostgreSQL → Kafka CDC → Elasticsearch indexer, each video document indexed with all searchable fields,

- **Search query**:
  GET /v1/videos/search?q=Inception&genre=Sci-Fi&actor=Leonardo DiCaprio&language=English&sort=popularity → Backend builds ES query: POST /videos/_search { 'query': { 'bool': { 'must': [ { 'multi_match': { 'query': 'Inception', 'fields': ['title^3', 'description', 'actors', 'director'], 'type': 'best_fields' } } ], 'filter': [ { 'term': { 'genre': 'Sci-Fi' } }, { 'match': { 'actors': 'Leonardo DiCaprio' } }, { 'term': { 'language': 'English' } } ] } }, 'sort': [ { 'popularity_score': 'desc' } ], 'from': 0, 'size': 20 },

- **Scoring**:
  (a) Text relevance: 'title^3' boosts title matches 3x over description, (b) Popularity: popularity_score = log(view_count) + 0.5 × avg_rating + recency_boost, calculated offline daily via batch job, (c) Personalization (advanced): Boost genres user previously watched, use ML model (collaborative filtering) to rerank results,

- **Faceted search**:
  Add aggregations: 'aggs': { 'genres': { 'terms': { 'field': 'genre', 'size': 20 } }, 'languages': { 'terms': { 'field': 'language' } }, 'rating_buckets': { 'histogram': { 'field': 'popularity_score', 'interval': 1 } } } → Returns counts: {Sci-Fi: 150, Action: 200, ...} for UI filters,

- **Autocomplete**:
  Separate index with edge_ngram tokenizer for type-ahead suggestions, queries update every keystroke with debouncing (300ms),

- **Caching**:
  Redis caches popular queries: Key = hash(query_text + filters + sort), Value = JSON result, TTL = 10 min, Cache hit rate 60-70%,

- **Performance**:
  ES cluster: 10 data nodes (5 primary shards, 1 replica each), query latency p95 <100ms for 10M videos,

- **Index updates**:
  Near real-time via Kafka, new videos searchable within 1-2 seconds of metadata creation. Example: User searches 'leonardo' → Autocomplete suggests 'Leonardo DiCaprio', 'The Revenant', 'Inception' → User selects 'Inception' → Full search returns Inception + similar movies → User filters by 'Sci-Fi' → Results filtered client-side or re-query with filter.

---

### Q6: How do you handle subscription management and payment processing?

Subscription lifecycle with payment integration:

- **Subscription plans**:
  Table subscription_plans {plan_id: 'basic', name: 'Basic', price: 9.99, currency: 'USD', max_resolution: '720p', max_screens: 1, billing_period: 'monthly'}, Plans: Basic ($9/mo, 720p, 1 screen), Standard ($13/mo, 1080p, 2 screens), Premium ($17/mo, 4K, 4 screens),

- **User subscribes**:
  POST /v1/subscriptions with {user_id, plan_id: 'premium', payment_method_id: 'pm_abc123' (from Stripe)},

- **Payment flow**:
  (a) Subscription Service calls Stripe API: stripe.subscriptions.create({ customer: user.stripe_customer_id, items: [{price: plan.stripe_price_id}], payment_behavior: 'default_incomplete', expand: ['latest_invoice.payment_intent'] }), (b) Returns client_secret for confirming payment on frontend, (c) User confirms payment in browser (3D Secure if required), (d) Stripe sends webhook POST /webhooks/stripe with {type: 'invoice.paid', subscription_id, payment_intent},

- **Webhook handling**:
  (a) Validate signature (security), (b) Idempotency check: SELECT * FROM subscriptions WHERE stripe_subscription_id={subscription_id}, if exists return 200 (already processed), (c) BEGIN TRANSACTION; INSERT INTO subscriptions (subscription_id, user_id, plan_id, status='ACTIVE', start_date=now(), end_date=now()+30days, stripe_subscription_id, stripe_payment_intent_id); UPDATE users SET subscription_status='ACTIVE', subscription_id={subscription_id}; COMMIT;, (d) Invalidate Redis cache: DEL user:{user_id}:subscription, (e) Publish Kafka 'subscription.created' → Notification Service sends welcome email,

- **Access control**:
  On video playback request: (a) GET /v1/videos/{video_id}/play → Validate subscription: SELECT status, end_date, plan_id FROM subscriptions WHERE user_id={user_id} AND status='ACTIVE', (b) Check plan allows video quality: if requesting 4K but plan.max_resolution='1080p' → return 403 with upgrade prompt, (c) Check concurrent streams: SELECT COUNT(*) FROM active_sessions WHERE user_id={user_id} → if count >= plan.max_screens → return 403 'Too many devices', (d) Generate signed CDN URL with expiry,

- **Auto-renewal**:
  (a) Stripe automatically charges on renewal date (30 days from start), (b) If payment succeeds → webhook 'invoice.paid' → UPDATE subscriptions SET end_date=end_date+30days, (c) If payment fails → webhook 'invoice.payment_failed' → Notification Service sends email, grace period 7 days before downgrade, retry payment 3 times, (d) After grace period: UPDATE subscriptions SET status='EXPIRED' → user downgraded to free tier (if exists) or blocked,

- **Cancellation**:
  PUT /v1/subscriptions/{subscription_id}/cancel → stripe.subscriptions.update({subscription_id, cancel_at_period_end: true}) → User keeps access until end_date, then status='CANCELLED',

- **Billing history**:
  GET /v1/users/{user_id}/billing → fetch from Stripe API: stripe.invoices.list({customer: stripe_customer_id}) → display in user dashboard. Edge cases:

- **Dunning**:
  Failed payment → retry with exponential backoff, send notification,

- **Refunds**:
  if user cancels within 7 days, pro-rated refund via stripe.refunds.create(),

- **Upgrades**:
  Mid-cycle upgrade from Basic to Premium → calculate pro-rated charge for remaining days,

- **Regional pricing**:
  Different prices per country stored in subscription_plans_regional table.

---

### Q7: How do you track and store watch history for 200M users?

Scalable watch history with Cassandra + Redis:

- **Real-time progress tracking**:
  Client sends heartbeat every 10 seconds: POST /v1/videos/{video_id}/progress with {user_id, current_time_seconds: 1245, timestamp},

- **Redis for hot data**:
  (a) Write to Redis: HSET watch_progress:{user_id}:{video_id} 'current_time' {current_time} 'last_updated' {timestamp} 'device_id' {device_id}, (b) TTL: EXPIRE watch_progress:{user_id}:{video_id} 604800 (7 days), auto-cleanup old sessions, (c) Read on resume: GET /v1/videos/{video_id}/play → HGET watch_progress:{user_id}:{video_id} 'current_time' → return {resume_from: 1245}, client seeks to position,

- **Cassandra for persistence**:
  (a) Background job (every 60s): Batch write from Redis to Cassandra, (b) Table schema: CREATE TABLE watch_history (user_id uuid, video_id uuid, watched_at timestamp, current_time int, completion_percentage int, device_type text, PRIMARY KEY (user_id, watched_at)) WITH CLUSTERING ORDER BY (watched_at DESC), partition by user_id for fast user-specific queries, (c) INSERT: INSERT INTO watch_history (user_id, video_id, watched_at, current_time, completion_percentage, device_type) VALUES ({user_id}, {video_id}, now(), {current_time}, {percentage}, {device_type}),

- **Analytics pipeline**:
  (a) Kafka event stream: On every progress update, publish 'video.watched' event: {user_id, video_id, session_id, current_time, duration_watched, timestamp, geo_location, device_type}, (b) Stream processing: Apache Flink/Spark Streaming consumes events, calculates aggregations: total_watch_time, unique_viewers, completion_rate, drop-off_points, (c) Results stored in analytics DB (BigQuery/Redshift) for dashboards,

- **Recommendations**:
  (a) Collaborative filtering: Users who watched X also watched Y, (b) Content-based: Similar genre, actors, (c) ML model (TensorFlow) trained on: watch_history (implicit feedback: user watched = positive signal), explicit ratings if available, search queries, (d) Inference: Batch job runs nightly, generates top 20 recommendations per user, stored in Redis: SET recommendations:{user_id} '[video_ids]' EX 86400, (e) Real-time updates: As user watches, update recommendations incrementally,

- **Scalability**:
  (a) Cassandra: Partitioned by user_id, 200M users × 100 entries avg × 500 bytes = 10 TB, replicated 3x = 30 TB, (b) Redis: Only active sessions (10M concurrent users × 1 KB = 10 GB), (c) Cassandra write throughput: 20M DAU × 1 update/10s = 2M writes/sec, Cassandra handles 1M writes/sec per node, 3 nodes sufficient with replication,

- **Data retention**:
  Cassandra TTL on rows: WITH default_time_to_live = 31536000 (1 year), old data auto-deleted. Example: User pauses at 20:45 → Client sends progress → Redis updated → Background job syncs to Cassandra → User opens app next day → Resume API queries Redis → Returns last_position=1245s → Player seeks to 20:45 → Seamless resume.

---

### Q8: How do you optimize video delivery cost at scale (CDN, storage, transcoding)?

Multi-layer cost optimization:

- **CDN caching**:
  95% cache hit rate → only 5% requests hit origin, saves 95% of origin bandwidth cost, Cost: 20M DAU × 2h × 5Mbps / 8 = 25M GB/day → 750M GB/month, CDN $0.085/GB → $63M/month, Origin (5%) $0.023/GB → $63M × 0.05 × 0.023/0.085 = $0.86M/month, Total CDN: $63.86M/month → Optimization: Negotiate volume discounts (50% off at petabyte scale) → ~$32M/month,

- **Storage tiering**:
  (a) Hot data (recent releases, popular content): S3 Standard ($0.023/GB) → 20% of catalog = 80 TB × $0.023 = $1,840/month, (b) Warm data (1-year old): S3 Infrequent Access ($0.0125/GB) → 30% = 120 TB × $0.0125 = $1,500/month, (c) Cold data (>2 years old, rarely watched): S3 Glacier ($0.004/GB) → 50% = 200 TB × $0.004 = $800/month, Total storage: $4,140/month vs $9,200 all-standard (55% savings),

- **Transcoding optimization**:
  (a) Spot instances: Use EC2 spot for 70% of transcoding workers → 70% discount vs on-demand ($2.50/hr → $0.75/hr), Risk: Interruption → batch job idempotent, restarts from checkpoint, (b) Auto-scaling: Scale workers 10-100 based on queue depth (SQS), average 30 instances × 24h × 30 days × $0.75 = $16,200/month, (c) Smart resolution selection: Don't transcode 4K for all content (only blockbusters), save 30% transcoding cost,

- **Compression**:
  (a) H.265/HEVC: 50% better compression vs H.264, reduces storage and bandwidth by 40%, 10k videos × 40GB → with HEVC 10k × 24GB = 240 TB vs 400 TB, savings: 160 TB × $0.023 = $3,680/month storage + proportional CDN savings, (b) AV1 (future): 30% better than HEVC, but slower encoding (2x time), not yet widely supported,

- **Lazy transcoding**:
  Don't transcode all resolutions immediately, transcode on-demand when requested (e.g., only 1% watch 4K), saves 40% transcoding cost,

- **Content pruning**:
  Delete unpopular content after 5 years (watched <100 times), frees storage, saves $500/month,

- **Regional optimization**:
  Store content in S3 bucket closest to main audience region (US content in us-east-1), reduces cross-region transfer costs,

- **Manifest file compression**:
  Gzip .m3u8 files saves 70% → 10k videos × 10 KB × 0.7 = negligible but good practice,

- **Reserved capacity**:
  Reserve 50% baseline CDN bandwidth for 1-year term → 30% discount vs on-demand,

- **Monitoring and alerting**:
  Track cost per video, cost per user, shutdown zombie resources (unused transcoding workers, orphaned S3 objects). Total monthly cost (before optimizations): CDN $63M + Storage $9K + Transcoding $50K + Misc $100K ≈ $63.2M → After optimizations: CDN $32M + Storage $4K + Transcoding $16K + Misc $50K ≈ $32.1M (49% savings).

---

### Q9: How do you implement DRM (Digital Rights Management) to prevent piracy?

DRM implementation for content protection:

- **DRM standards**:
  (a) Widevine: Google standard, used by Chrome, Android, (b) FairPlay: Apple standard, used by Safari, iOS, (c) PlayReady: Microsoft standard, used by Edge, Xbox,

- **Encryption**:
  (a) Video segments encrypted during transcoding using AES-128 or AES-256, (b) Each segment encrypted with unique encryption key, (c) Encryption keys stored in License Server (AWS Elemental MediaPackage, BuyDRM, etc.),

- **Key delivery**:
  (a) Player requests manifest → receives encrypted segment URLs + license_url, (b) Player requests decryption key: GET {license_url} with user_token and device_id, (c) License Server validates: User has active subscription, Device is registered (max 5 devices per user), DRM client installed (not emulator/rooted device), (d) If valid: Returns decryption key encrypted with device's public key (only that device can decrypt), (e) Player decrypts segments in memory, renders video to screen (encrypted buffer → decryption → display pipeline),

- **Content protection**:
  (a) HDCP (High-bandwidth Digital Content Protection): Prevents screen recording via HDMI, enforced by hardware, (b) Watermarking: Invisible forensic watermark embedded with user_id, if leaked can trace source, (c) Screen capture blocking: DRM API blocks screen recording apps, screenshots show black screen,

- **Offline downloads**:
  (a) Download encrypted segments to local storage (mobile app), (b) License downloaded with expiry (e.g., 30 days), (c) On playback: Validate license not expired, decrypt and play, (d) On subscription end: License invalidated, downloaded content unplayable,

- **Implementation**:
  (a) Transcoding: FFmpeg with DRM packaging: ffmpeg -i input.mp4 -c:v libx264 -encryption_scheme cenc -encryption_key {key} -encryption_kid {key_id} output.mp4, (b) Manifest: HLS includes #EXT-X-KEY:METHOD=SAMPLE-AES,URI={license_server_url},KEYFORMAT='com.apple.streamingkeydelivery', (c) Player: Use ExoPlayer (Android), AVPlayer (iOS), Shaka Player (web) with DRM plugin,

- **Multi-DRM**:
  Support all 3 standards for cross-platform: if Safari → FairPlay, if Chrome → Widevine, if Edge → PlayReady, use unified license server (BuyDRM, Irdeto) that handles all,

- **Challenges**:
  (a) Performance: Decryption overhead ~5-10% CPU, use hardware-accelerated decryption (GPU), (b) Compatibility: Older devices may not support, fallback to lower DRM level or no DRM, (c) Cost: License server $0.01-0.05 per view, 20M daily views = $200K-$1M/month,

- **Anti-piracy**:
  (a) Geofencing: Block VPNs, restrict content by region, (b) Concurrent stream limits: Max 2-4 devices simultaneously, (c) Token expiry: Signed URLs valid for 4 hours, forces re-authentication. Example: User clicks play → Player requests manifest → Manifest includes encrypted segment URLs + license_url → Player requests license from server with user_token → Server validates subscription + device → Returns decryption key (encrypted) → Player decrypts using device key → Segments decrypted in memory and rendered → User cannot save or share video file.

---

### Q10: What's your disaster recovery and fault tolerance strategy for a streaming platform?

Multi-region DR with automated failover:

- **Architecture**:
  Primary region us-east-1, Secondary us-west-2, Tertiary eu-west-1,

- **Data replication**:
  (a) PostgreSQL: Primary in us-east-1, streaming replication to us-west-2 (warm standby, lag <5s) and eu-west-1 (read replica, lag <10s), (b) S3: Cross-region replication (CRR) enabled, video segments replicated to all 3 regions automatically within minutes, (c) Redis: Redis Sentinel for automatic failover, master-slave replication (lag <1s), AOF persistence for durability, (d) Cassandra: Multi-DC deployment with replication_factor=3 across all regions, LOCAL_QUORUM for reads/writes,

- **CDN resilience**:
  CloudFront automatically routes to healthy origins, if us-east-1 S3 fails → CloudFront fetches from us-west-2 bucket transparently, no user impact,

- **Failover procedure**:
  (a) Health checks: Route53 health checks on primary region every 30s (TCP, HTTP, latency), (b) Trigger: If 3 consecutive checks fail (90s) → Route53 updates DNS to point to us-west-2, (c) Database promotion: us-west-2 PostgreSQL promoted to primary (pg_ctl promote), takes ~2-3 min, (d) Application servers: Auto-scaling groups in us-west-2 pre-warmed (min 10% capacity always running), scale up to 100% within 5 min, (e) DNS propagation: 5-10 min for DNS TTL, progressive migration as clients refresh,

- **RPO/RTO targets**:
  (a) RPO (Recovery Point Objective): <5 seconds data loss (streaming replication lag), (b) RTO (Recovery Time Objective): <10 minutes service restoration,

- **Service degradation**:
  (a) Read-only mode: If primary DB fails but replicas healthy → serve read-only, users can watch but not subscribe/update profiles, (b) CDN-only mode: If all origins down → serve cached content only (95% of requests), new uploads fail, (c) Graceful degradation: Disable non-critical features (recommendations, continue watching) to reduce load,

- **Testing**:
  (a) Monthly DR drills: Simulate primary region failure, execute failover, verify RTO met, (b) Chaos engineering: Randomly terminate instances in production (during low-traffic hours), verify auto-recovery, (c) Game days: Simulate various failure scenarios (DB corruption, CDN outage, DDoS attack),

- **Monitoring**:
  (a) Datadog/New Relic: Real-time metrics (latency, error rate, throughput), (b) Alerts: PagerDuty for critical issues (DB lag >10s, API error rate >1%, CDN cache hit rate <90%), (c) Dashboards: Grafana showing: concurrent viewers, video playback errors, buffering ratio, DB replication lag,

- **Backup and restore**:
  (a) S3: Versioning enabled, accidental deletes recoverable, (b) Database: Daily full backups + hourly incremental to S3 Glacier, retention 30 days, (c) Point-in-time recovery: Can restore DB to any point within last 7 days, (d) Backup restore testing: Quarterly restore from backup to staging environment, verify data integrity,

- **DDoS protection**:
  (a) AWS Shield Advanced: Protects against L3/L4 DDoS, (b) CloudFront rate limiting: Geo-blocking suspicious traffic, WAF rules block malicious patterns, (c) Auto-scaling: Absorb traffic spikes up to 10x normal load. Example failure: Primary region (us-east-1) suffers power outage at 2:00 PM → Route53 health checks fail at 2:01:30 → DNS updated to us-west-2 at 2:02 → PostgreSQL promoted at 2:04 → Users experience brief buffering (current segment completes from cache, next segment from us-west-2) → Service fully restored at 2:10 PM → Data loss: <5s of watch progress for in-flight requests. Total cost: Multi-region setup adds 30-40% infrastructure cost, insurance against revenue loss during outages (1 hour outage = $500K lost revenue + reputation damage).

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Streaming Protocol** | MPEG-DASH (.mpd) / HLS (.m3u8) | Monolithic MP4 File Streaming | Slices video into 10s chunks, enabling dynamic Adaptive Bitrate quality switching on the fly. |
| **Edge Distribution** | ISP-Partnered Open Connect / CDN | Direct AWS S3 Video Ingress | Eliminates multi-million dollar internet backbone egress fees; reduces buffering latency to $< 2\text{s}$. |
| **Catalog Storage** | Apache Cassandra | Relational SQL (MySQL) | Linear horizontal scalability for millions of concurrent read queries across global regions. |
| **Watch Progress State**| Dual Store: Redis (Hot) + Cassandra (Cold) | Direct Relational DB Writes | Absorbs millions of 10-second heartbeat writes in memory without causing disk I/O bottlenecks. |
| **Transcoding Engine** | Asynchronous Distributed FFmpeg Workers | Monolithic Upload Server | Parallelizes compute-heavy encoding across dozens of worker pods; scales dynamically with upload volume. |
