# FAANG+ System Design Master Handbook & Architectural Blueprints

> A comprehensive, production-grade collection of **30 end-to-end System Design blueprints**, architectural trade-off analyses, and **150+ staff-level interview follow-up cross-questions**.  
> Every design follows the standardized **5-Step FAANG Delivery Framework** and incorporates real-world capacity math, database schemas, API contracts, ASCII architecture diagrams, and low-level concurrency controls.

---

## 📚 Table of Contents
1. [The 5-Step Delivery Framework (Master Blueprint)](#the-5-step-delivery-framework)
2. [Architectural Pattern Lineage & Transferred Learnings](#architectural-pattern-lineage)
3. [The Complete System Design Catalog (30 Problems)](#system-design-catalog)
   - [Part I: Interview with Bunny Series (01–21)](#part-i-interview-with-bunny-series)
   - [Part II: Advanced Distributed & High-Scale Systems (22–29)](#part-ii-advanced-distributed-systems)
4. [Last-Minute 15-Minute Revision Matrix (Advanced Systems 23–29)](#last-minute-revision-matrix)
5. [Master Interview Question Bank & Topic Reference (171 Questions)](#master-interview-question-bank)
6. [Latency Numbers & Capacity Math Cheatsheet](#latency-numbers-cheatsheet)
7. [Technology Selection Rules of Thumb](#technology-selection-rules-of-thumb)

---

## 🎯 The 5-Step Delivery Framework

All blueprints in this repository strictly adhere to the universal delivery framework codified in [`00-system-design-blueprint-pattern.md`](./00-system-design-blueprint-pattern.md):

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        THE 45-MINUTE SYSTEM DESIGN INTERVIEW BLUEPRINT                 │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 1: Requirements & Estimations (5–8 Mins)                                          │
│   ├── Functional Requirements (FR) & Non-Functional Requirements (NFR)                 │
│   └── Back-of-the-Envelope Math (QPS, 5-Yr Storage, Memory / 80-20 Cache RAM, Bandwidth)│
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 2: Core Entities & Data Modeling (3–5 Mins)                                       │
│   └── Entity Relationships, Relational vs NoSQL, Primary/Partition/Sharding Keys        │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 3: API & Protocol Interface Design (4–6 Mins)                                     │
│   └── REST Endpoints, WebSocket Frames, Protobuf Schemas, Idempotency Headers          │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 4: High-Level Architecture (HLD) (8–10 Mins)                                      │
│   └── Component Topologies, Load Balancers, Gateways, Queues, Caches, Read Replicas    │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 5: Low-Level Design (LLD) & Deep Dives (12–15 Mins)                               │
│   └── Concurrency Controls (Locks/Sagas), Algorithms, State Machines, Failure Failovers│
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 6: Common Interview Follow-Ups & Cross-Questions (Deep Dive)                      │
│   └── 5–10 staff-level edge cases, scale bottlenecks, consistency anomalies            │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Step 7: Architectural Trade-Off Analysis Matrix & Key Takeaways                        │
│   └── Structured comparative matrix defending technology selections                    │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Architectural Pattern Lineage

Mastering system design is about **pattern recognition and cross-pollination**. The table below highlights how architectural building blocks transfer across different domains in this repository:

| Subsystem Challenge | Transferred Pattern Archetype | Foundational Lineage File |
| :--- | :--- | :--- |
| **Concurrency & Double-Booking** | Redis `SETNX` Lock + Postgres `SELECT FOR UPDATE` | [`02-design-ticket-booking-system.md`](./02-design-ticket-booking-system.md) |
| **High-Throughput Unique IDs** | Twitter Snowflake (Timestamp + Worker ID + Seq) | [`01-design-tinyurl.md`](./01-design-tinyurl.md) |
| **Geospatial Proximity Search** | Dynamic QuadTrees, Geohashing, Uber H3 Hexagons | [`07-design-proximity-search.md`](./07-design-proximity-search.md) & [`11-design-ride-sharing-uber.md`](./11-design-ride-sharing-uber.md) |
| **Real-Time Bidirectional Sync** | Stateful WebSocket Gateways & Epoll Socket Tuning | [`08-design-chat-app-whatsapp.md`](./08-design-chat-app-whatsapp.md) & [`27-design-real-time-presence-system.md`](./27-design-real-time-presence-system.md) |
| **Change Data Capture (CDC)** | DB Commitlogs $\to$ Debezium $\to$ Kafka $\to$ Elasticsearch | [`02-design-ticket-booking-system.md`](./02-design-ticket-booking-system.md) & [`03-design-ecommerce-platform.md`](./03-design-ecommerce-platform.md) |
| **Multi-Stage Recommendation** | Retrieval (HNSW) $\to$ ML Scoring (DLRM) $\to$ Re-ranking | [`04-design-ott-platform-netflix.md`](./04-design-ott-platform-netflix.md) & [`28-design-real-time-ecommerce-recommendation-system.md`](./28-design-real-time-ecommerce-recommendation-system.md) |
| **Financial Ledger Integrity** | Double-Entry Bookkeeping & Idempotent Keys | [`17-design-payment-gateway-stripe.md`](./17-design-payment-gateway-stripe.md) |
| **Heavy Binary Chunking** | Content-Defined Chunking (FastCDC / Rabin Fingerprints) | [`18-design-google-drive-dropbox.md`](./18-design-google-drive-dropbox.md) |
| **Surge Traffic Mitigation** | Virtual Waiting Room (Token Bucket Priority Queue) | [`02-design-ticket-booking-system.md`](./02-design-ticket-booking-system.md) & [`21-design-distributed-rate-limiter.md`](./21-design-distributed-rate-limiter.md) |
| **Collaborative Conflict Res.**| Operational Transformation (OT) & CRDTs | [`10-design-collaborative-editor-google-docs.md`](./10-design-collaborative-editor-google-docs.md) |
| **High-Frequency Ingestion** | UDP Multicast + LMAX Disruptor Ring Buffer + NACK | [`29-design-real-time-market-data-ingestion-system.md`](./29-design-real-time-market-data-ingestion-system.md) |

---

## 🗂️ System Design Catalog

### Part I: Interview with Bunny Series (Complete 21 Lessons)

Each problem below contains the complete architectural breakdown, capacity estimations, database schemas, and Section 6 with dedicated **Common Interview Follow-Ups & Cross-Questions**:

| # | Document | Primary Architectural Themes | Core Trade-offs & Focus Areas | Cross Qs |
| :---: | :--- | :--- | :--- | :---: |
| **00** | [`00-system-design-blueprint-pattern.md`](./00-system-design-blueprint-pattern.md) | Master 5-Step Framework | Universal template, estimation constants, comparative matrices. | — |
| **01** | [`01-design-tinyurl.md`](./01-design-tinyurl.md) | URL Shortener (TinyURL / Bitly) | Snowflake + ZK Worker ID, Base62 encoding, 302 vs 301, Bloom filters, abuse prevention. | 3 Qs |
| **02** | [`02-design-ticket-booking-system.md`](./02-design-ticket-booking-system.md) | Ticket Booking (BookMyShow / District) | `SETNX` distributed locks, Postgres `SELECT FOR UPDATE`, CDC sync to ES, virtual waiting room. | 9 Qs |
| **03** | [`03-design-ecommerce-platform.md`](./03-design-ecommerce-platform.md) | E-Commerce Platform (Amazon / Flipkart) | Distributed inventory reservation Saga, transactional outbox, cart sync, flash sales. | 10 Qs |
| **04** | [`04-design-ott-platform-netflix.md`](./04-design-ott-platform-netflix.md) | OTT Video Platform (Netflix / Prime) | Chunking & Transcoding DAGs, CDN Open Connect, Dynamic ABR (HLS/DASH), DRM tokens. | 10 Qs |
| **05** | [`05-design-hotel-booking-airbnb.md`](./05-design-hotel-booking-airbnb.md) | Hotel Booking (Airbnb / Booking.com) | Date-range calendar indices (`GiST`), optimistic locking, multi-currency escrow. | 10 Qs |
| **06** | [`06-design-food-delivery-zomato-swiggy.md`](./06-design-food-delivery-zomato-swiggy.md) | Food Delivery (Zomato / Swiggy) | Dynamic routing ETA, driver batching, live GPS ingestion via Redis Geo, order state machines. | 10 Qs |
| **07** | [`07-design-proximity-search.md`](./07-design-proximity-search.md) | Proximity Search (Yelp / Google Maps) | Dynamic QuadTree vs Geohash vs Google S2, memory sharding, bounding box pre-filtering. | 10 Qs |
| **08** | [`08-design-chat-app-whatsapp.md`](./08-design-chat-app-whatsapp.md) | Chat Application (WhatsApp / Telegram) | Offline message queues, WebSocket session managers, Cassandra message schema, tick receipts. | 10 Qs |
| **09** | [`09-design-distributed-job-scheduler.md`](./09-design-distributed-job-scheduler.md) | Distributed Job Scheduler (Airflow) | Leader election, DAG dependency resolution, heartbeat crash detection, exponential backoff. | 10 Qs |
| **10** | [`10-design-collaborative-editor-google-docs.md`](./10-design-collaborative-editor-google-docs.md) | Collaborative Editor (Google Docs) | Operational Transformation (OT) vs CRDTs, central sequencer, cursor broadcasting, snapshotting. | 8 Qs |
| **11** | [`11-design-ride-sharing-uber.md`](./11-design-ride-sharing-uber.md) | Ride Sharing (Uber / Lyft) | Dynamic surge pricing via Uber H3 hex cells, driver dispatch lock, WebSocket GPS tracking. | 5 Qs |
| **12** | [`12-design-stock-broker-zerodha.md`](./12-design-stock-broker-zerodha.md) | Stock Broker Platform (Zerodha) | Sub-50ms tick streaming, atomic ledger balance checks, order state machine. | 3 Qs |
| **13** | [`13-design-social-media-feed.md`](./13-design-social-media-feed.md) | Social Media Feed (Instagram / Facebook) | Hybrid Fan-out (write for normal users, read for celebrities), like counts in Redis, pagination. | 10 Qs |
| **14** | [`14-design-top-k-leaderboard.md`](./14-design-top-k-leaderboard.md) | Real-time Leaderboard & Ranking | Redis Sorted Sets (`ZSET`), Count-Min Sketch, YouTube trending velocity decay scoring. | 3 Qs |
| **15** | [`15-design-notification-system.md`](./15-design-notification-system.md) | Scalable Notification Service | At-least-once delivery, APNs/FCM provider circuit breakers, priority Kafka queues. | 3 Qs |
| **16** | [`16-design-distributed-logging-splunk.md`](./16-design-distributed-logging-splunk.md) | Distributed Logging (Splunk / ELK) | Kafka ingestion buffer, out-of-order log watermarks, Flink stream deduplication, index tiering. | 10 Qs |
| **17** | [`17-design-payment-gateway-stripe.md`](./17-design-payment-gateway-stripe.md) | Payment Gateway (Stripe) | PCI-DSS tokenization, idempotency keys, double-entry ledger, bank reconciliation. | 10 Qs |
| **18** | [`18-design-google-drive-dropbox.md`](./18-design-google-drive-dropbox.md) | Cloud Storage (Google Drive / Dropbox) | FastCDC / Rabin Fingerprints rolling hash deduplication, S3 multipart resumable uploads. | 4 Qs |
| **19** | [`19-design-online-learning-udemy.md`](./19-design-online-learning-udemy.md) | Online Learning (Udemy / Coursera) | Adaptive Bitrate Video (HLS), video resume checkpoints, idempotent course enrollment. | 3 Qs |
| **20** | [`20-design-email-delivery-gmail.md`](./20-design-email-delivery-gmail.md) | Email Delivery Engine (Gmail / SendGrid) | Outbox orchestrator, SLA priority routing, SPF / DKIM / DMARC cryptographic validation. | 3 Qs |
| **21** | [`21-design-distributed-rate-limiter.md`](./21-design-distributed-rate-limiter.md) | Distributed Rate Limiter | Token Bucket, Leaky Bucket, Sliding Window Counter, Redis Lua atomic synchronization. | 3 Qs |

---

### Part II: Advanced Distributed & High-Scale Systems (08 Core Additions)

| # | Document | Primary Architectural Themes | Core Trade-offs & Focus Areas | Cross Qs |
| :---: | :--- | :--- | :--- | :---: |
| **22** | [`22-design-yelp-proximity.md`](./22-design-yelp-proximity.md) | Yelp Proximity & Search | BKD-Trees vs QuadTrees, spatial sharding, location cache invalidation. | 5 Qs |
| **23** | [`23-design-distributed-cache.md`](./23-design-distributed-cache.md) | Distributed Cache (Memcached/Redis) | Consistent hashing with virtual nodes, LRU eviction, thundering herd / cache stampede. | 5 Qs |
| **24** | [`24-design-online-auction.md`](./24-design-online-auction.md) | Real-Time Online Auction (eBay) | Kafka partition serialization by auction_id, SSE + Redis Pub/Sub live streaming, DB sharding. | 4 Qs |
| **25** | [`25-design-news-aggregator.md`](./25-design-news-aggregator.md) | News Aggregator (Google News) | Distributed web scrapers, SimHash near-duplicate article deduplication, inverted search indexing. | 6 Qs |
| **26** | [`26-design-leetcode-judge.md`](./26-design-leetcode-judge.md) | Online Code Judge (LeetCode) | Secure untrusted code execution sandbox (Docker/gVisor), cgroups limits, worker queue scheduling. | 6 Qs |
| **27** | [`27-design-real-time-presence-system.md`](./27-design-real-time-presence-system.md) | Real-Time Presence (WhatsApp/Slack) | 50M WebSockets, 1.67M QPS heartbeats, debounce state machine, dynamic viewport subscriptions. | 8 Qs |
| **28** | [`28-design-real-time-ecommerce-recommendation-system.md`](./28-design-real-time-ecommerce-recommendation-system.md) | E-Commerce Recommendation (Amazon) | Asynchronous clickstream ingestion, Flink nearline feature vectors, 3-stage funnel (HNSW + DLRM). | 8 Qs |
| **29** | [`29-design-real-time-market-data-ingestion-system.md`](./29-design-real-time-market-data-ingestion-system.md) | Financial Market Data Ingestion | 20M ticks/sec UDP multicast, MoldUDP64/ITCH, NACK TCP retransmission, LMAX Disruptor ring buffer. | 8 Qs |

---

<span id="last-minute-revision-matrix"></span>
## ⚡ Last-Minute 15-Minute Revision Matrix (Advanced Systems 23–29)

Skim this quick-reference matrix when you have **15 minutes before an interview** to recall core mental models, throughput metrics, and primary concurrency defenses:

| # | System Design Domain | 30-Second Mental Model Flow | Key Metric / SLA | Primary Concurrency & Failure Defense | Document Link |
| :---: | :--- | :--- | :--- | :--- | :---: |
| **23** | **Distributed Cache** (Redis/Memcached) | Client Smart Routing (MurmurHash3 on Consistent Hash Ring with 256 v-nodes) $	o$ Shard Primary ($O(1)$ Hash Map + Doubly-Linked List LRU) $	o$ Async Replica Sync. | $< 1\text{ms}$ read; $1\text{M}$ QPS; $1\text{TB}$ RAM. | Consistent Hashing Ring avoids 95% rehash stampede; Distributed Mutex (`SETNX`) on cache miss. | [`23-design-distributed-cache.md`](./23-design-distributed-cache.md) |
| **24** | **Online Auction System** (eBay) | Bid $	o$ Envoy $	o$ Go Bidding Service $	o$ Redis Atomic Lua CAS (checks end-time, valid increment, updates state) $	o$ Redis Pub/Sub WebSocket fanout + Kafka DB write-behind. | $50\text{k}$ bids/sec on closing item; $< 50\text{ms}$ fanout. | Redis Lua atomic compare-and-swap (CAS); Anti-sniping auto-extends `end_time` by 2 mins if bid within 120s. | [`24-design-online-auction.md`](./24-design-online-auction.md) |
| **25** | **News Aggregator** (Google News) | Distributed Crawler Fleet (Politeness Token Bucket + HTTP ETag) $	o$ Kafka $	o$ 64-bit SimHash Deduplicator (Hamming dist $\le 3$) $	o$ NLP Tagger $	o$ Elasticsearch + Redis `ZSET`s. | 10k sources; 500k arts/day; 10k QPS read at $< 50\text{ms}$. | SimHash deduplication for syndicated press; Per-domain politeness throttling to prevent crawler IP bans. | [`25-design-news-aggregator.md`](./25-design-news-aggregator.md) |
| **26** | **Online Code Judge** (LeetCode) | Code $	o$ Gateway $	o$ PostgreSQL `PENDING` $	o$ Kafka (lang partition) $	o$ Judge Worker $	o$ MicroVM Sandbox (gVisor/Firecracker, cgroups v2, `--net=none`) $	o$ WebSockets. | 10M runs/day; $1\text{k}$ QPS; 1.0s CPU / 256MB RAM cap. | User-space virtualization (gVisor) + cgroups v2 (`pids.max=32` anti-forkbomb, `--net=none` airgap). | [`26-design-leetcode-judge.md`](./26-design-leetcode-judge.md) |
| **27** | **Real-Time Presence** (WhatsApp/Slack) | 50M WebSockets (Linux epoll) $	o$ 30s pings batched into Redis Cluster (`SETEX` 60s TTL) $	o$ 30s Debounce State Machine $	o$ Redis Pub/Sub Viewport Fanout $	o$ ScyllaDB last seen. | 50M conns; 1.67M QPS heartbeats; 3.2GB Redis RAM. | 30-second disconnect debounce grace period prevents mobile flapping; Dynamic viewport limits fanout. | [`27-design-real-time-presence-system.md`](./27-design-real-time-presence-system.md) |
| **28** | **E-Commerce Recs** (Amazon) | Clicks $	o$ Kafka $	o$ Flink 15-min Session Window $	o$ Redis Intent Vector $	o$ 3-Stage Funnel: HNSW Retrieval (1000) $	o$ DLRM TensorRT Ranking (100) $	o$ Suppression (20). | 100M items; 200k clicks/s; $< 50\text{ms}$ p99 response. | 3-stage funnel prevents scoring 100M items with deep net; Redis suppression set solves washing machine trap. | [`28-design-real-time-ecommerce-recommendation-system.md`](./28-design-real-time-ecommerce-recommendation-system.md) |
| **29** | **Market Data Ingestion** (Exchanges) | Global Feeds $	o$ Loaders $	o$ UDP Multicast (MoldUDP64) $	o$ Kernel-Bypass NICs $	o$ LMAX Disruptor Ring Buffer $	o$ TCP NACK Retransmission $	o$ Aeron IPC bots + ClickHouse. | 20M ticks/sec; $< 500\mu\text{s}$ internal; 57B daily ticks. | UDP Multicast avoids TCP head-of-line blocking; LMAX Disruptor lock-free ring buffer avoids Java GC pauses. | [`29-design-real-time-market-data-ingestion-system.md`](./29-design-real-time-market-data-ingestion-system.md) |

---

<span id="master-interview-question-bank"></span>
## 🔍 Master Interview Question Bank & Topic Reference (171 Questions)

Below is the complete, searchable question-to-document index mapping **every single interview follow-up cross-question** across all 29 system design problems directly to its exact file and section anchor. Use this index to jump straight to specific technical deep dives during interview preparation:

| # | System Design Domain / Topic | Specific Follow-Up / Cross-Question | Direct File Reference |
| :---: | :--- | :--- | :--- |
| 1 | **Design TinyURL / Bitly** | How do you generate unique short URLs? Compare the three main approaches (ZooKeeper Counter, Hash-Based MD5/SHA-1, and Twitter Snowflake). | [01-design-tinyurl.md](01-design-tinyurl.md#q1-how-do-you-generate-unique-short-urls-compare-the-three-main-approaches-zookeeper-counter-hash-based-md5sha-1-and-twitter-snowflake) |
| 2 | **Design TinyURL / Bitly** | How do you handle 1 Billion redirects per day with low latency (<100ms SLA, p99)? | [01-design-tinyurl.md](01-design-tinyurl.md#q2-how-do-you-handle-1-billion-redirects-per-day-with-low-latency-100ms-sla-p99) |
| 3 | **Design TinyURL / Bitly** | How do you prevent abuse (spam URLs, malicious links) and implement rate limiting? | [01-design-tinyurl.md](01-design-tinyurl.md#q3-how-do-you-prevent-abuse-spam-urls-malicious-links-and-implement-rate-limiting) |
| 4 | **Design Ticket Booking System** | How do you prevent double booking when 100 users try to book the same last seat? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q1-how-do-you-prevent-double-booking-when-100-users-try-to-book-the-same-last-seat) |
| 5 | **Design Ticket Booking System** | Why use Redis holds with TTL instead of just database expiry? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q2-why-use-redis-holds-with-ttl-instead-of-just-database-expiry) |
| 6 | **Design Ticket Booking System** | How does the CDC pipeline work for syncing Event DB to Elasticsearch? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q3-how-does-the-cdc-pipeline-work-for-syncing-event-db-to-elasticsearch) |
| 7 | **Design Ticket Booking System** | How do you handle payment failures and refunds? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q4-how-do-you-handle-payment-failures-and-refunds) |
| 8 | **Design Ticket Booking System** | How do you handle popular event launches (e.g., Marvel movie premiere)? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q5-how-do-you-handle-popular-event-launches-eg-marvel-movie-premiere) |
| 9 | **Design Ticket Booking System** | How does seat map rendering work for large venues (1000+ seats)? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q6-how-does-seat-map-rendering-work-for-large-venues-1000-seats) |
| 10 | **Design Ticket Booking System** | How do you prevent bots from hoarding seats during high-demand shows? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q7-how-do-you-prevent-bots-from-hoarding-seats-during-high-demand-shows) |
| 11 | **Design Ticket Booking System** | What happens if the database goes down during a booking transaction? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q8-what-happens-if-the-database-goes-down-during-a-booking-transaction) |
| 12 | **Design Ticket Booking System** | How do you implement QR code generation for ticket verification at venue entry? | [02-design-ticket-booking-system.md](02-design-ticket-booking-system.md#q9-how-do-you-implement-qr-code-generation-for-ticket-verification-at-venue-entry) |
| 13 | **Design E-Commerce Platform** | How do you prevent overselling when multiple users try to buy the last item? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q1-how-do-you-prevent-overselling-when-multiple-users-try-to-buy-the-last-item) |
| 14 | **Design E-Commerce Platform** | How do you implement product search with filters (category, price, rating)? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q2-how-do-you-implement-product-search-with-filters-category-price-rating) |
| 15 | **Design E-Commerce Platform** | How do you manage shopping cart state across sessions and devices? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q3-how-do-you-manage-shopping-cart-state-across-sessions-and-devices) |
| 16 | **Design E-Commerce Platform** | How do you handle the checkout flow from cart to order confirmation? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q4-how-do-you-handle-the-checkout-flow-from-cart-to-order-confirmation) |
| 17 | **Design E-Commerce Platform** | How do you implement CDC (Change Data Capture) for syncing Product DB to Elasticsearch? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q5-how-do-you-implement-cdc-change-data-capture-for-syncing-product-db-to-elasticsearch) |
| 18 | **Design E-Commerce Platform** | How do you design the payment integration with idempotency and webhook handling? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q6-how-do-you-design-the-payment-integration-with-idempotency-and-webhook-handling) |
| 19 | **Design E-Commerce Platform** | How do you track order status and implement real-time updates for users? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q7-how-do-you-track-order-status-and-implement-real-time-updates-for-users) |
| 20 | **Design E-Commerce Platform** | How do you handle high-volume flash sales (100K orders in 1 minute)? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q8-how-do-you-handle-high-volume-flash-sales-100k-orders-in-1-minute) |
| 21 | **Design E-Commerce Platform** | How do you implement product recommendations and personalization? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q9-how-do-you-implement-product-recommendations-and-personalization) |
| 22 | **Design E-Commerce Platform** | What's your disaster recovery and data consistency strategy for the e-commerce platform? | [03-design-ecommerce-platform.md](03-design-ecommerce-platform.md#q10-whats-your-disaster-recovery-and-data-consistency-strategy-for-the-e-commerce-platform) |
| 23 | **Design OTT Platform** | How does adaptive bitrate streaming work in Netflix/YouTube? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q1-how-does-adaptive-bitrate-streaming-work-in-netflixyoutube) |
| 24 | **Design OTT Platform** | Explain the video upload and transcoding pipeline end-to-end. | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q2-explain-the-video-upload-and-transcoding-pipeline-end-to-end) |
| 25 | **Design OTT Platform** | How do you minimize video buffering and ensure smooth playback? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q3-how-do-you-minimize-video-buffering-and-ensure-smooth-playback) |
| 26 | **Design OTT Platform** | How do you design the CDN architecture for global video delivery? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q4-how-do-you-design-the-cdn-architecture-for-global-video-delivery) |
| 27 | **Design OTT Platform** | How do you implement video search with filters (genre, actor, language)? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q5-how-do-you-implement-video-search-with-filters-genre-actor-language) |
| 28 | **Design OTT Platform** | How do you handle subscription management and payment processing? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q6-how-do-you-handle-subscription-management-and-payment-processing) |
| 29 | **Design OTT Platform** | How do you track and store watch history for 200M users? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q7-how-do-you-track-and-store-watch-history-for-200m-users) |
| 30 | **Design OTT Platform** | How do you optimize video delivery cost at scale (CDN, storage, transcoding)? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q8-how-do-you-optimize-video-delivery-cost-at-scale-cdn-storage-transcoding) |
| 31 | **Design OTT Platform** | How do you implement DRM (Digital Rights Management) to prevent piracy? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q9-how-do-you-implement-drm-digital-rights-management-to-prevent-piracy) |
| 32 | **Design OTT Platform** | What's your disaster recovery and fault tolerance strategy for a streaming platform? | [04-design-ott-platform-netflix.md](04-design-ott-platform-netflix.md#q10-whats-your-disaster-recovery-and-fault-tolerance-strategy-for-a-streaming-platform) |
| 33 | **Design Hotel Booking App** | How do you prevent double booking (two users booking same room for overlapping dates)? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q1-how-do-you-prevent-double-booking-two-users-booking-same-room-for-overlapping-dates) |
| 34 | **Design Hotel Booking App** | How do you implement hotel search with location, dates, and price filters efficiently? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q2-how-do-you-implement-hotel-search-with-location-dates-and-price-filters-efficiently) |
| 35 | **Design Hotel Booking App** | How do you ensure payment idempotency (prevent double charging if user clicks pay twice)? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q3-how-do-you-ensure-payment-idempotency-prevent-double-charging-if-user-clicks-pay-twice) |
| 36 | **Design Hotel Booking App** | How do you handle room availability updates and booking modifications? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q4-how-do-you-handle-room-availability-updates-and-booking-modifications) |
| 37 | **Design Hotel Booking App** | How do you implement the review and rating system with aggregation? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q5-how-do-you-implement-the-review-and-rating-system-with-aggregation) |
| 38 | **Design Hotel Booking App** | How do you handle time zones and date calculations for global hotels? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q6-how-do-you-handle-time-zones-and-date-calculations-for-global-hotels) |
| 39 | **Design Hotel Booking App** | How do you implement cancellation policies and refund processing? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q7-how-do-you-implement-cancellation-policies-and-refund-processing) |
| 40 | **Design Hotel Booking App** | How do you scale the search service to handle 100M searches per day? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q8-how-do-you-scale-the-search-service-to-handle-100m-searches-per-day) |
| 41 | **Design Hotel Booking App** | How do you handle overbooking scenarios and inventory management? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q9-how-do-you-handle-overbooking-scenarios-and-inventory-management) |
| 42 | **Design Hotel Booking App** | What's your strategy for disaster recovery and data consistency? | [05-design-hotel-booking-airbnb.md](05-design-hotel-booking-airbnb.md#q10-whats-your-strategy-for-disaster-recovery-and-data-consistency) |
| 43 | **Design Food Delivery App** | How do you search for nearby restaurants efficiently? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q1-how-do-you-search-for-nearby-restaurants-efficiently) |
| 44 | **Design Food Delivery App** | How do you implement cart functionality with fast add/remove operations? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q2-how-do-you-implement-cart-functionality-with-fast-addremove-operations) |
| 45 | **Design Food Delivery App** | How do you ensure exactly-once order processing and prevent double charging? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q3-how-do-you-ensure-exactly-once-order-processing-and-prevent-double-charging) |
| 46 | **Design Food Delivery App** | How do you match drivers to orders efficiently and prevent double assignment? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q4-how-do-you-match-drivers-to-orders-efficiently-and-prevent-double-assignment) |
| 47 | **Design Food Delivery App** | How do you implement real-time order tracking with driver location updates? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q5-how-do-you-implement-real-time-order-tracking-with-driver-location-updates) |
| 48 | **Design Food Delivery App** | How do you handle order cancellations and refunds? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q6-how-do-you-handle-order-cancellations-and-refunds) |
| 49 | **Design Food Delivery App** | What's your strategy for handling peak traffic during lunch/dinner hours? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q7-whats-your-strategy-for-handling-peak-traffic-during-lunchdinner-hours) |
| 50 | **Design Food Delivery App** | How do you ensure data consistency across multiple microservices? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q8-how-do-you-ensure-data-consistency-across-multiple-microservices) |
| 51 | **Design Food Delivery App** | How do you implement search with multiple filters (cuisine, rating, delivery time)? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q9-how-do-you-implement-search-with-multiple-filters-cuisine-rating-delivery-time) |
| 52 | **Design Food Delivery App** | How do you calculate delivery ETA accurately? | [06-design-food-delivery-zomato-swiggy.md](06-design-food-delivery-zomato-swiggy.md#q10-how-do-you-calculate-delivery-eta-accurately) |
| 53 | **Design Proximity Search** | Explain geohash and how to handle edge cases at cell boundaries. | [07-design-proximity-search.md](07-design-proximity-search.md#q1-explain-geohash-and-how-to-handle-edge-cases-at-cell-boundaries) |
| 54 | **Design Proximity Search** | Compare QuadTree vs Geohash vs PostGIS - when to use each? | [07-design-proximity-search.md](07-design-proximity-search.md#q2-compare-quadtree-vs-geohash-vs-postgis---when-to-use-each) |
| 55 | **Design Proximity Search** | How do you optimize proximity search for millions of locations? | [07-design-proximity-search.md](07-design-proximity-search.md#q3-how-do-you-optimize-proximity-search-for-millions-of-locations) |
| 56 | **Design Proximity Search** | Implement proximity search with PostgreSQL PostGIS - show schema and query. | [07-design-proximity-search.md](07-design-proximity-search.md#q4-implement-proximity-search-with-postgresql-postgis---show-schema-and-query) |
| 57 | **Design Proximity Search** | How does Redis GEORADIUS work internally and when should you use it? | [07-design-proximity-search.md](07-design-proximity-search.md#q5-how-does-redis-georadius-work-internally-and-when-should-you-use-it) |
| 58 | **Design Proximity Search** | How do you calculate Haversine distance and why use bounding box pre-filter? | [07-design-proximity-search.md](07-design-proximity-search.md#q6-how-do-you-calculate-haversine-distance-and-why-use-bounding-box-pre-filter) |
| 59 | **Design Proximity Search** | Design Elasticsearch mapping and query for 'pizza restaurants within 5km, rating ≥4'. | [07-design-proximity-search.md](07-design-proximity-search.md#q7-design-elasticsearch-mapping-and-query-for-pizza-restaurants-within-5km-rating-4) |
| 60 | **Design Proximity Search** | How do you shard geospatial data for horizontal scaling? | [07-design-proximity-search.md](07-design-proximity-search.md#q8-how-do-you-shard-geospatial-data-for-horizontal-scaling) |
| 61 | **Design Proximity Search** | What's your caching strategy for proximity searches? | [07-design-proximity-search.md](07-design-proximity-search.md#q9-whats-your-caching-strategy-for-proximity-searches) |
| 62 | **Design Proximity Search** | How do you handle polygon-based searches (e.g., restaurants in a neighborhood)? | [07-design-proximity-search.md](07-design-proximity-search.md#q10-how-do-you-handle-polygon-based-searches-eg-restaurants-in-a-neighborhood) |
| 63 | **Design Chat Application** | How do you ensure messages are delivered even when the recipient is offline? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q1-how-do-you-ensure-messages-are-delivered-even-when-the-recipient-is-offline) |
| 64 | **Design Chat Application** | How do you implement read receipts and delivery status at scale? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q2-how-do-you-implement-read-receipts-and-delivery-status-at-scale) |
| 65 | **Design Chat Application** | What happens if a WebSocket connection drops in the middle of sending a message? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q3-what-happens-if-a-websocket-connection-drops-in-the-middle-of-sending-a-message) |
| 66 | **Design Chat Application** | How do you handle last seen and online status efficiently? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q4-how-do-you-handle-last-seen-and-online-status-efficiently) |
| 67 | **Design Chat Application** | How do you design the message schema in Cassandra for efficient queries? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q5-how-do-you-design-the-message-schema-in-cassandra-for-efficient-queries) |
| 68 | **Design Chat Application** | How do you implement group messaging efficiently? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q6-how-do-you-implement-group-messaging-efficiently) |
| 69 | **Design Chat Application** | What's your media upload and delivery strategy? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q7-whats-your-media-upload-and-delivery-strategy) |
| 70 | **Design Chat Application** | How do you implement typing indicators in real-time? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q8-how-do-you-implement-typing-indicators-in-real-time) |
| 71 | **Design Chat Application** | How do you handle message ordering and prevent race conditions? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q9-how-do-you-handle-message-ordering-and-prevent-race-conditions) |
| 72 | **Design Chat Application** | What's your disaster recovery strategy for chat data? | [08-design-chat-app-whatsapp.md](08-design-chat-app-whatsapp.md#q10-whats-your-disaster-recovery-strategy-for-chat-data) |
| 73 | **Design Distributed Job Scheduler** | How do you prevent the same job from being scheduled twice by multiple watcher instances? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q1-how-do-you-prevent-the-same-job-from-being-scheduled-twice-by-multiple-watcher-instances) |
| 74 | **Design Distributed Job Scheduler** | How do you handle jobs that are scheduled while the scheduler is down? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q2-how-do-you-handle-jobs-that-are-scheduled-while-the-scheduler-is-down) |
| 75 | **Design Distributed Job Scheduler** | How do you implement retry mechanism with exponential backoff? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q3-how-do-you-implement-retry-mechanism-with-exponential-backoff) |
| 76 | **Design Distributed Job Scheduler** | How do you handle job dependencies in a DAG (Directed Acyclic Graph)? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q4-how-do-you-handle-job-dependencies-in-a-dag-directed-acyclic-graph) |
| 77 | **Design Distributed Job Scheduler** | What happens if an executor crashes while running a job? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q5-what-happens-if-an-executor-crashes-while-running-a-job) |
| 78 | **Design Distributed Job Scheduler** | How do you implement priority-based job scheduling? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q6-how-do-you-implement-priority-based-job-scheduling) |
| 79 | **Design Distributed Job Scheduler** | How do you handle time zone conversions for scheduled jobs? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q7-how-do-you-handle-time-zone-conversions-for-scheduled-jobs) |
| 80 | **Design Distributed Job Scheduler** | How do you implement job cancellation for already running jobs? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q8-how-do-you-implement-job-cancellation-for-already-running-jobs) |
| 81 | **Design Distributed Job Scheduler** | How do you prevent job duplication on executor restart? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q9-how-do-you-prevent-job-duplication-on-executor-restart) |
| 82 | **Design Distributed Job Scheduler** | What's your strategy for monitoring and alerting on job failures? | [09-design-distributed-job-scheduler.md](09-design-distributed-job-scheduler.md#q10-whats-your-strategy-for-monitoring-and-alerting-on-job-failures) |
| 83 | **Design Real-Time Collaborative Text Editor** | How does Operational Transformation (OT) work to resolve conflicts in concurrent editing? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q1-how-does-operational-transformation-ot-work-to-resolve-conflicts-in-concurrent-editing) |
| 84 | **Design Real-Time Collaborative Text Editor** | What are CRDTs and how do they differ from Operational Transformation? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q2-what-are-crdts-and-how-do-they-differ-from-operational-transformation) |
| 85 | **Design Real-Time Collaborative Text Editor** | How do you handle offline editing and sync when the user reconnects? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q3-how-do-you-handle-offline-editing-and-sync-when-the-user-reconnects) |
| 86 | **Design Real-Time Collaborative Text Editor** | How do you implement cursor presence and real-time cursor updates for multiple users? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q4-how-do-you-implement-cursor-presence-and-real-time-cursor-updates-for-multiple-users) |
| 87 | **Design Real-Time Collaborative Text Editor** | How do you implement auto-save and periodic snapshots? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q5-how-do-you-implement-auto-save-and-periodic-snapshots) |
| 88 | **Design Real-Time Collaborative Text Editor** | How do you handle version history and rollback to previous versions? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q6-how-do-you-handle-version-history-and-rollback-to-previous-versions) |
| 89 | **Design Real-Time Collaborative Text Editor** | How do you implement the Reconciliation Service and what does it do? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q7-how-do-you-implement-the-reconciliation-service-and-what-does-it-do) |
| 90 | **Design Real-Time Collaborative Text Editor** | How would you scale this system to handle 1 million concurrent editors? | [10-design-collaborative-editor-google-docs.md](10-design-collaborative-editor-google-docs.md#q8-how-would-you-scale-this-system-to-handle-1-million-concurrent-editors) |
| 91 | **Design Ride Sharing Application** | How do you prevent double assignment of a driver to multiple ride requests? | [11-design-ride-sharing-uber.md](11-design-ride-sharing-uber.md#q1-how-do-you-prevent-double-assignment-of-a-driver-to-multiple-ride-requests) |
| 92 | **Design Ride Sharing Application** | How does real-time location tracking work with WebSockets? | [11-design-ride-sharing-uber.md](11-design-ride-sharing-uber.md#q2-how-does-real-time-location-tracking-work-with-websockets) |
| 93 | **Design Ride Sharing Application** | How do you calculate surge pricing dynamically? | [11-design-ride-sharing-uber.md](11-design-ride-sharing-uber.md#q3-how-do-you-calculate-surge-pricing-dynamically) |
| 94 | **Design Ride Sharing Application** | How do you handle geospatial queries efficiently for finding nearby drivers? | [11-design-ride-sharing-uber.md](11-design-ride-sharing-uber.md#q4-how-do-you-handle-geospatial-queries-efficiently-for-finding-nearby-drivers) |
| 95 | **Design Ride Sharing Application** | How do you design the notification system for real-time updates to riders and drivers? | [11-design-ride-sharing-uber.md](11-design-ride-sharing-uber.md#q5-how-do-you-design-the-notification-system-for-real-time-updates-to-riders-and-drivers) |
| 96 | **Design Stock Broker Trading Platform** | How do you ensure real-time price updates reach millions of users with <50ms latency? | [12-design-stock-broker-zerodha.md](12-design-stock-broker-zerodha.md#q1-how-do-you-ensure-real-time-price-updates-reach-millions-of-users-with-50ms-latency) |
| 97 | **Design Stock Broker Trading Platform** | How do you prevent overdraft when multiple orders are placed simultaneously? | [12-design-stock-broker-zerodha.md](12-design-stock-broker-zerodha.md#q2-how-do-you-prevent-overdraft-when-multiple-orders-are-placed-simultaneously) |
| 98 | **Design Stock Broker Trading Platform** | Walk through complete order flow from placement to execution with all validations and state transitions. | [12-design-stock-broker-zerodha.md](12-design-stock-broker-zerodha.md#q3-walk-through-complete-order-flow-from-placement-to-execution-with-all-validations-and-state-transitions) |
| 99 | **Design Social Media Feed Platform** | Why use fanout on write instead of fanout on read for normal users? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q1-why-use-fanout-on-write-instead-of-fanout-on-read-for-normal-users) |
| 100 | **Design Social Media Feed Platform** | How do you handle the celebrity/hotspot problem in feed generation? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q2-how-do-you-handle-the-celebrityhotspot-problem-in-feed-generation) |
| 101 | **Design Social Media Feed Platform** | How do you ensure consistency between like counts in cache vs database? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q3-how-do-you-ensure-consistency-between-like-counts-in-cache-vs-database) |
| 102 | **Design Social Media Feed Platform** | What happens if a user with 1M followers posts? How do you prevent system overload? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q4-what-happens-if-a-user-with-1m-followers-posts-how-do-you-prevent-system-overload) |
| 103 | **Design Social Media Feed Platform** | How do you handle nested comment threads efficiently? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q5-how-do-you-handle-nested-comment-threads-efficiently) |
| 104 | **Design Social Media Feed Platform** | How do you implement real-time notifications without polling? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q6-how-do-you-implement-real-time-notifications-without-polling) |
| 105 | **Design Social Media Feed Platform** | How do you handle media upload failures and retries? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q7-how-do-you-handle-media-upload-failures-and-retries) |
| 106 | **Design Social Media Feed Platform** | What's your database partitioning strategy for Posts at scale? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q8-whats-your-database-partitioning-strategy-for-posts-at-scale) |
| 107 | **Design Social Media Feed Platform** | How do you prevent duplicate likes when user clicks multiple times? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q9-how-do-you-prevent-duplicate-likes-when-user-clicks-multiple-times) |
| 108 | **Design Social Media Feed Platform** | How do you implement privacy controls (public vs friends-only posts)? | [13-design-social-media-feed.md](13-design-social-media-feed.md#q10-how-do-you-implement-privacy-controls-public-vs-friends-only-posts) |
| 109 | **Design Top K Leaderboard / Real-Time Ranking System** | Design a real-time gaming leaderboard that shows top 100 players globally and by region. How do you handle millions of score updates per second? | [14-design-top-k-leaderboard.md](14-design-top-k-leaderboard.md#q1-design-a-real-time-gaming-leaderboard-that-shows-top-100-players-globally-and-by-region-how-do-you-handle-millions-of-score-updates-per-second) |
| 110 | **Design Top K Leaderboard / Real-Time Ranking System** | How would you design a YouTube Trending page that shows top 50 trending videos? Explain the time-decay algorithm. | [14-design-top-k-leaderboard.md](14-design-top-k-leaderboard.md#q2-how-would-you-design-a-youtube-trending-page-that-shows-top-50-trending-videos-explain-the-time-decay-algorithm) |
| 111 | **Design Top K Leaderboard / Real-Time Ranking System** | Compare Redis ZSET vs database-based leaderboard approaches. When would you choose one over the other? | [14-design-top-k-leaderboard.md](14-design-top-k-leaderboard.md#q3-compare-redis-zset-vs-database-based-leaderboard-approaches-when-would-you-choose-one-over-the-other) |
| 112 | **Design Scalable Notification System** | How do you ensure no notification is lost and implement at-least-once delivery guarantee? | [15-design-notification-system.md](15-design-notification-system.md#q1-how-do-you-ensure-no-notification-is-lost-and-implement-at-least-once-delivery-guarantee) |
| 113 | **Design Scalable Notification System** | How do you handle rate limiting when sending millions of notifications to prevent overwhelming external providers? | [15-design-notification-system.md](15-design-notification-system.md#q2-how-do-you-handle-rate-limiting-when-sending-millions-of-notifications-to-prevent-overwhelming-external-providers) |
| 114 | **Design Scalable Notification System** | Walk through complete notification flow from creation to delivery with all state transitions and failure handling. | [15-design-notification-system.md](15-design-notification-system.md#q3-walk-through-complete-notification-flow-from-creation-to-delivery-with-all-state-transitions-and-failure-handling) |
| 115 | **Design Distributed Logging System** | Why use Kafka as a buffer instead of writing directly to the database? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q1-why-use-kafka-as-a-buffer-instead-of-writing-directly-to-the-database) |
| 116 | **Design Distributed Logging System** | How do you handle out-of-order logs with different timestamps? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q2-how-do-you-handle-out-of-order-logs-with-different-timestamps) |
| 117 | **Design Distributed Logging System** | What happens if Elasticsearch goes down? Can users still search logs? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q3-what-happens-if-elasticsearch-goes-down-can-users-still-search-logs) |
| 118 | **Design Distributed Logging System** | How do you ensure no log data is lost during ingestion? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q4-how-do-you-ensure-no-log-data-is-lost-during-ingestion) |
| 119 | **Design Distributed Logging System** | How does the deduplication mechanism work in Flink? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q5-how-does-the-deduplication-mechanism-work-in-flink) |
| 120 | **Design Distributed Logging System** | What's your data retention and archival strategy? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q6-whats-your-data-retention-and-archival-strategy) |
| 121 | **Design Distributed Logging System** | How do you handle log search across multiple time ranges efficiently? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q7-how-do-you-handle-log-search-across-multiple-time-ranges-efficiently) |
| 122 | **Design Distributed Logging System** | How do you implement real-time log tailing efficiently? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q8-how-do-you-implement-real-time-log-tailing-efficiently) |
| 123 | **Design Distributed Logging System** | What's your alerting strategy to avoid alert fatigue? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q9-whats-your-alerting-strategy-to-avoid-alert-fatigue) |
| 124 | **Design Distributed Logging System** | How do you handle schema evolution when log formats change? | [16-design-distributed-logging-splunk.md](16-design-distributed-logging-splunk.md#q10-how-do-you-handle-schema-evolution-when-log-formats-change) |
| 125 | **Design Payment Gateway System** | Why use tokenization instead of storing card numbers? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q1-why-use-tokenization-instead-of-storing-card-numbers) |
| 126 | **Design Payment Gateway System** | How do you handle concurrent payment attempts with the same card? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q2-how-do-you-handle-concurrent-payment-attempts-with-the-same-card) |
| 127 | **Design Payment Gateway System** | What happens if payment processor API fails midway? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q3-what-happens-if-payment-processor-api-fails-midway) |
| 128 | **Design Payment Gateway System** | How to ensure payment data durability and prevent loss? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q4-how-to-ensure-payment-data-durability-and-prevent-loss) |
| 129 | **Design Payment Gateway System** | How does the reconciliation process work? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q5-how-does-the-reconciliation-process-work) |
| 130 | **Design Payment Gateway System** | What if the database goes down during payment processing? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q6-what-if-the-database-goes-down-during-payment-processing) |
| 131 | **Design Payment Gateway System** | How do you handle quota/rate limits per merchant? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q7-how-do-you-handle-quotarate-limits-per-merchant) |
| 132 | **Design Payment Gateway System** | What's the optimal session timeout and why? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q8-whats-the-optimal-session-timeout-and-why) |
| 133 | **Design Payment Gateway System** | How do you prevent replay attacks on payment APIs? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q9-how-do-you-prevent-replay-attacks-on-payment-apis) |
| 134 | **Design Payment Gateway System** | How do you handle refunds and partial refunds? | [17-design-payment-gateway-stripe.md](17-design-payment-gateway-stripe.md#q10-how-do-you-handle-refunds-and-partial-refunds) |
| 135 | **System Design: Google Drive / Dropbox** | How do you implement chunking and deduplication (Fixed-Size vs. Rolling Hash / Rabin Fingerprints)? | [18-design-google-drive-dropbox.md](18-design-google-drive-dropbox.md#q1-how-do-you-implement-chunking-and-deduplication-fixed-size-vs-rolling-hash-rabin-fingerprints) |
| 136 | **System Design: Google Drive / Dropbox** | How do you handle resumable uploads for multi-gigabyte files over unstable networks? | [18-design-google-drive-dropbox.md](18-design-google-drive-dropbox.md#q2-how-do-you-handle-resumable-uploads-for-multi-gigabyte-files-over-unstable-networks) |
| 137 | **System Design: Google Drive / Dropbox** | How do you maintain consistency between Relational Metadata DB and S3 Block Storage under concurrent edits? | [18-design-google-drive-dropbox.md](18-design-google-drive-dropbox.md#q3-how-do-you-maintain-consistency-between-relational-metadata-db-and-s3-block-storage-under-concurrent-edits) |
| 138 | **System Design: Google Drive / Dropbox** | How does the client synchronization engine propagate file updates across millions of connected devices in real time? | [18-design-google-drive-dropbox.md](18-design-google-drive-dropbox.md#q4-how-does-the-client-synchronization-engine-propagate-file-updates-across-millions-of-connected-devices-in-real-time) |
| 139 | **System Design: Online Learning Platform** | How do you achieve <2 second video startup time with adaptive bitrate streaming? | [19-design-online-learning-udemy.md](19-design-online-learning-udemy.md#q1-how-do-you-achieve-2-second-video-startup-time-with-adaptive-bitrate-streaming) |
| 140 | **System Design: Online Learning Platform** | How do you track progress for millions of students watching videos and taking quizzes? | [19-design-online-learning-udemy.md](19-design-online-learning-udemy.md#q2-how-do-you-track-progress-for-millions-of-students-watching-videos-and-taking-quizzes) |
| 141 | **System Design: Online Learning Platform** | How do you prevent double enrollment when payment succeeds but webhook is duplicated or lost? | [19-design-online-learning-udemy.md](19-design-online-learning-udemy.md#q3-how-do-you-prevent-double-enrollment-when-payment-succeeds-but-webhook-is-duplicated-or-lost) |
| 142 | **System Design: Email Delivery System** | How do you ensure reliable email delivery with retry mechanisms and handle failures? | [20-design-email-delivery-gmail.md](20-design-email-delivery-gmail.md#q1-how-do-you-ensure-reliable-email-delivery-with-retry-mechanisms-and-handle-failures) |
| 143 | **System Design: Email Delivery System** | How does the Outbox Delivery Orchestrator implement SLA-based priority routing? | [20-design-email-delivery-gmail.md](20-design-email-delivery-gmail.md#q2-how-does-the-outbox-delivery-orchestrator-implement-sla-based-priority-routing) |
| 144 | **System Design: Email Delivery System** | Walk through the complete SMTP delivery process with SPF, DKIM, and DMARC validation. | [20-design-email-delivery-gmail.md](20-design-email-delivery-gmail.md#q3-walk-through-the-complete-smtp-delivery-process-with-spf-dkim-and-dmarc-validation) |
| 145 | **System Design: Distributed Rate Limiter** | Compare the 5 rate limiting algorithms (Fixed Window, Sliding Window Log, Sliding Window Counter, Token Bucket, Leaky Bucket). Which would you choose for a production API and why? | [21-design-distributed-rate-limiter.md](21-design-distributed-rate-limiter.md#q1-compare-the-5-rate-limiting-algorithms-fixed-window-sliding-window-log-sliding-window-counter-token-bucket-leaky-bucket-which-would-you-choose-for-a-production-api-and-why) |
| 146 | **System Design: Distributed Rate Limiter** | How do you implement distributed rate limiting across multiple API Gateway instances using Redis? Handle race conditions and ensure consistency. | [21-design-distributed-rate-limiter.md](21-design-distributed-rate-limiter.md#q2-how-do-you-implement-distributed-rate-limiting-across-multiple-api-gateway-instances-using-redis-handle-race-conditions-and-ensure-consistency) |
| 147 | **System Design: Distributed Rate Limiter** | Walk through complete request flow: client makes request → rate limiter checks → either allowed or rejected. Include policy lookup, token bucket check, and 429 response with headers. | [21-design-distributed-rate-limiter.md](21-design-distributed-rate-limiter.md#q3-walk-through-complete-request-flow-client-makes-request-rate-limiter-checks-either-allowed-or-rejected-include-policy-lookup-token-bucket-check-and-429-response-with-headers) |
| 148 | **Real-Time Presence System** | How do you prevent presence flapping when mobile users switch between Wi-Fi and 5G? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q1-how-do-you-prevent-presence-flapping-when-mobile-users-switch-between-wi-fi-and-5g) |
| 149 | **Real-Time Presence System** | How do you solve the celebrity / large group presence fanout problem (e.g., 100,000 users in a Discord or Slack channel)? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q2-how-do-you-solve-the-celebrity-large-group-presence-fanout-problem-eg-100000-users-in-a-discord-or-slack-channel) |
| 150 | **Real-Time Presence System** | How do you scale to 50M concurrent WebSockets without running out of Linux file descriptors or memory? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q3-how-do-you-scale-to-50m-concurrent-websockets-without-running-out-of-linux-file-descriptors-or-memory) |
| 151 | **Real-Time Presence System** | How does the system scale if Redis crashes—is presence state permanently lost? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q4-how-does-the-system-scale-if-redis-crashesis-presence-state-permanently-lost) |
| 152 | **Real-Time Presence System** | How do you implement privacy settings (e.g., "Hide Last Seen" or "Nobody can see my status") efficiently? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q5-how-do-you-implement-privacy-settings-eg-hide-last-seen-or-nobody-can-see-my-status-efficiently) |
| 153 | **Real-Time Presence System** | How do you minimize battery and data consumption on mobile clients sending heartbeats? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q6-how-do-you-minimize-battery-and-data-consumption-on-mobile-clients-sending-heartbeats) |
| 154 | **Real-Time Presence System** | Why not use Kafka for real-time presence fanout instead of Redis Pub/Sub? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q7-why-not-use-kafka-for-real-time-presence-fanout-instead-of-redis-pubsub) |
| 155 | **Real-Time Presence System** | What happens to last-seen timestamps during a major network partition or datacenter outage? | [27-design-real-time-presence-system.md](27-design-real-time-presence-system.md#q8-what-happens-to-last-seen-timestamps-during-a-major-network-partition-or-datacenter-outage) |
| 156 | **Real-Time E-Commerce Product Recommendation System** | How do you guarantee the recommendation service returns within $< 50\text{ms}$ when ML models experience tail latency spikes? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q1-how-do-you-guarantee-the-recommendation-service-returns-within-50textms-when-ml-models-experience-tail-latency-spikes) |
| 157 | **Real-Time E-Commerce Product Recommendation System** | How does the nearline Flink pipeline update user intent in real time without race conditions? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q2-how-does-the-nearline-flink-pipeline-update-user-intent-in-real-time-without-race-conditions) |
| 158 | **Real-Time E-Commerce Product Recommendation System** | How do you solve the "Cold Start" problem for brand-new users and zero-history items? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q3-how-do-you-solve-the-cold-start-problem-for-brand-new-users-and-zero-history-items) |
| 159 | **Real-Time E-Commerce Product Recommendation System** | How do you prevent recommending an item the user bought 2 minutes ago (Post-Purchase Suppression)? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q4-how-do-you-prevent-recommending-an-item-the-user-bought-2-minutes-ago-post-purchase-suppression) |
| 160 | **Real-Time E-Commerce Product Recommendation System** | How do you handle position bias in training data (users click top items simply because they appear first)? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q5-how-do-you-handle-position-bias-in-training-data-users-click-top-items-simply-because-they-appear-first) |
| 161 | **Real-Time E-Commerce Product Recommendation System** | How do you prevent recommendation fatigue and maintain catalog diversity? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q6-how-do-you-prevent-recommendation-fatigue-and-maintain-catalog-diversity) |
| 162 | **Real-Time E-Commerce Product Recommendation System** | How do you scale vector similarity search over 100 Million product embeddings? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q7-how-do-you-scale-vector-similarity-search-over-100-million-product-embeddings) |
| 163 | **Real-Time E-Commerce Product Recommendation System** | What happens during Prime Day / Black Friday when traffic surges $10\times$? | [28-design-real-time-ecommerce-recommendation-system.md](28-design-real-time-ecommerce-recommendation-system.md#q8-what-happens-during-prime-day-black-friday-when-traffic-surges-10times) |
| 164 | **Real-Time Market Data Ingestion & Distribution System** | How do you prevent a "NACK Storm" when a network switch drops a UDP multicast packet across 50 processor nodes simultaneously? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q1-how-do-you-prevent-a-nack-storm-when-a-network-switch-drops-a-udp-multicast-packet-across-50-processor-nodes-simultaneously) |
| 165 | **Real-Time Market Data Ingestion & Distribution System** | How do you handle clock skew and timestamp ordering across different global exchanges? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q2-how-do-you-handle-clock-skew-and-timestamp-ordering-across-different-global-exchanges) |
| 166 | **Real-Time Market Data Ingestion & Distribution System** | What happens if a processor falls behind and its in-memory ring buffer overflows? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q3-what-happens-if-a-processor-falls-behind-and-its-in-memory-ring-buffer-overflows) |
| 167 | **Real-Time Market Data Ingestion & Distribution System** | Why use UDP Multicast instead of Kafka for inter-service communication between Loaders and Processors? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q4-why-use-udp-multicast-instead-of-kafka-for-inter-service-communication-between-loaders-and-processors) |
| 168 | **Real-Time Market Data Ingestion & Distribution System** | How do you prevent memory leaks when tracking seen sequence numbers for deduplication? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q5-how-do-you-prevent-memory-leaks-when-tracking-seen-sequence-numbers-for-deduplication) |
| 169 | **Real-Time Market Data Ingestion & Distribution System** | How do you serve trading bots that demand raw ticks while simultaneously supporting retail web users on the same infrastructure? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q6-how-do-you-serve-trading-bots-that-demand-raw-ticks-while-simultaneously-supporting-retail-web-users-on-the-same-infrastructure) |
| 170 | **Real-Time Market Data Ingestion & Distribution System** | What happens if an Upstream Loader crashes during peak trading hours? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q7-what-happens-if-an-upstream-loader-crashes-during-peak-trading-hours) |
| 171 | **Real-Time Market Data Ingestion & Distribution System** | How do you persist 57 Billion daily ticks for historical analysis without slowing down live processing? | [29-design-real-time-market-data-ingestion-system.md](29-design-real-time-market-data-ingestion-system.md#q8-how-do-you-persist-57-billion-daily-ticks-for-historical-analysis-without-slowing-down-live-processing) |

---

## ⚡ Latency Numbers & Capacity Math Cheatsheet

Keep these hardware and networking constraints in mind for all capacity calculations:

```
┌──────────────────────────────────────────────┬───────────────────────────────┐
│ Operation                                    │ Typical Latency               │
├──────────────────────────────────────────────┼───────────────────────────────┤
│ L1 Cache Reference                           │ 0.5 ns                        │
│ Branch Mispredict                            │ 5 ns                          │
│ L2 Cache Reference                           │ 7 ns                          │
│ Mutex Lock / Unlock                          │ 25 ns                         │
│ Main Memory (RAM) Reference                  │ 100 ns                        │
│ Compress 1 KB with Zstandard                 │ 2,000 ns (2 μs)               │
│ Kernel-Bypass Network Packet (DPDK / EF_VI)  │ 2 μs                          │
│ Read 1 MB sequentially from Memory           │ 3,000 ns (3 μs)               │
│ Read 1 MB sequentially from NVMe SSD         │ 50 μs                         │
│ Redis In-Memory Point Lookup (Same DC)       │ 500 μs (0.5 ms)               │
│ Round-trip within Same Datacenter            │ 500 μs (0.5 ms)               │
│ Standard Linux POSIX Network Syscall         │ 25 μs                         │
│ PostgreSQL B-Tree Primary Key Lookup         │ 2–5 ms                        │
│ Cross-Continent Round-Trip (US East to West) │ 60 ms                         │
│ Transatlantic Round-Trip (NY to London)      │ 75 ms                         │
│ Transpacific Round-Trip (US to Tokyo)        │ 150 ms                        │
└──────────────────────────────────────────────┴───────────────────────────────┘
```

### High-Yield Conversions
- **1 Day in Seconds**: $86,400 \approx 10^5 \text{ seconds}$.
- **1 Million Requests / Day**: $\approx 12 \text{ requests / second}$.
- **100 Million Requests / Day**: $\approx 1,160 \text{ requests / second}$.
- **1 Billion Requests / Day**: $\approx 11,574 \text{ requests / second}$ (Peak $\approx 100,000 \text{ req/sec}$).
- **1 KB / Request at 100M Daily Requests**: $\approx 100 \text{ GB / day} \approx 36.5 \text{ TB / year}$.

---

## 🛠️ Technology Selection Rules of Thumb

- **Relational SQL (PostgreSQL / MySQL)**: Use for financial ledgers, transactional checkout, seat holds, and ACID integrity (`02`, `03`, `17`).
- **NoSQL Wide-Column (Cassandra / ScyllaDB)**: Use for massive write throughput, time-series events, chat history, and last-seen states (`05`, `08`, `20`, `27`).
- **In-Memory Cache (Redis Cluster)**: Use for hot key lookups, atomic locks (`SETNX`), sliding window rate limits, session presence, and ranking leaderboards (`01`, `02`, `14`, `21`, `27`).
- **Search & Analytics (Elasticsearch / ClickHouse)**: Use Elasticsearch for faceted text queries; use ClickHouse for high-throughput append-only event analytics and market data (`02`, `16`, `29`).
- **Message Streaming (Apache Kafka)**: Use for decoupling high-throughput producers from consumers, event-driven sagas, and outbox event streaming (`03`, `09`, `15`, `16`, `28`).
- **Low-Latency In-Memory Buffers (LMAX Disruptor / Aeron)**: Use for zero-allocation, sub-microsecond tick processing and auction bidding engines (`24`, `29`).

