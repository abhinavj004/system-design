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
4. [Latency Numbers & Capacity Math Cheatsheet](#latency-numbers-cheatsheet)
5. [Common Interview Follow-Ups & Cross-Questions Index](#cross-questions-index)

---

## 🎯 The 5-Step Delivery Framework

All blueprints in this repository strictly adhere to the universal delivery framework codified in [`00-system-design-blueprint-pattern.md`](file:///d:/system%20design%20generation/00-system-design-blueprint-pattern.md):

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
| **Concurrency & Double-Booking** | Redis `SETNX` Lock + Postgres `SELECT FOR UPDATE` | [`02-design-ticket-booking-system.md`](file:///d:/system%20design%20generation/02-design-ticket-booking-system.md) |
| **High-Throughput Unique IDs** | Twitter Snowflake (Timestamp + Worker ID + Seq) | [`01-design-tinyurl.md`](file:///d:/system%20design%20generation/01-design-tinyurl.md) |
| **Geospatial Proximity Search** | Dynamic QuadTrees, Geohashing, Uber H3 Hexagons | [`07-design-proximity-search.md`](file:///d:/system%20design%20generation/07-design-proximity-search.md) & [`11-design-ride-sharing-uber.md`](file:///d:/system%20design%20generation/11-design-ride-sharing-uber.md) |
| **Real-Time Bidirectional Sync** | Stateful WebSocket Gateways & Epoll Socket Tuning | [`08-design-chat-app-whatsapp.md`](file:///d:/system%20design%20generation/08-design-chat-app-whatsapp.md) & [`27-design-real-time-presence-system.md`](file:///d:/system%20design%20generation/27-design-real-time-presence-system.md) |
| **Change Data Capture (CDC)** | DB Commitlogs $\to$ Debezium $\to$ Kafka $\to$ Elasticsearch | [`02-design-ticket-booking-system.md`](file:///d:/system%20design%20generation/02-design-ticket-booking-system.md) & [`03-design-ecommerce-platform.md`](file:///d:/system%20design%20generation/03-design-ecommerce-platform.md) |
| **Multi-Stage Recommendation** | Retrieval (HNSW) $\to$ ML Scoring (DLRM) $\to$ Re-ranking | [`04-design-ott-platform-netflix.md`](file:///d:/system%20design%20generation/04-design-ott-platform-netflix.md) & [`28-design-real-time-ecommerce-recommendation-system.md`](file:///d:/system%20design%20generation/28-design-real-time-ecommerce-recommendation-system.md) |
| **Financial Ledger Integrity** | Double-Entry Bookkeeping & Idempotent Keys | [`17-design-payment-gateway-stripe.md`](file:///d:/system%20design%20generation/17-design-payment-gateway-stripe.md) |
| **Heavy Binary Chunking** | Content-Defined Chunking (FastCDC / Rabin Fingerprints) | [`18-design-google-drive-dropbox.md`](file:///d:/system%20design%20generation/18-design-google-drive-dropbox.md) |
| **Surge Traffic Mitigation** | Virtual Waiting Room (Token Bucket Priority Queue) | [`02-design-ticket-booking-system.md`](file:///d:/system%20design%20generation/02-design-ticket-booking-system.md) & [`21-design-distributed-rate-limiter.md`](file:///d:/system%20design%20generation/21-design-distributed-rate-limiter.md) |
| **Collaborative Conflict Res.**| Operational Transformation (OT) & CRDTs | [`10-design-collaborative-editor-google-docs.md`](file:///d:/system%20design%20generation/10-design-collaborative-editor-google-docs.md) |
| **High-Frequency Ingestion** | UDP Multicast + LMAX Disruptor Ring Buffer + NACK | [`29-design-real-time-market-data-ingestion-system.md`](file:///d:/system%20design%20generation/29-design-real-time-market-data-ingestion-system.md) |

---

## 🗂️ System Design Catalog

### Part I: Interview with Bunny Series (Complete 21 Lessons)

Each problem below contains the complete architectural breakdown, capacity estimations, database schemas, and Section 6 with dedicated **Common Interview Follow-Ups & Cross-Questions**:

| # | Document | Primary Architectural Themes | Core Trade-offs & Focus Areas | Cross Qs |
| :---: | :--- | :--- | :--- | :---: |
| **00** | [`00-system-design-blueprint-pattern.md`](file:///d:/system%20design%20generation/00-system-design-blueprint-pattern.md) | Master 5-Step Framework | Universal template, estimation constants, comparative matrices. | — |
| **01** | [`01-design-tinyurl.md`](file:///d:/system%20design%20generation/01-design-tinyurl.md) | URL Shortener (TinyURL / Bitly) | Snowflake + ZK Worker ID, Base62 encoding, 302 vs 301, Bloom filters, abuse prevention. | 3 Qs |
| **02** | [`02-design-ticket-booking-system.md`](file:///d:/system%20design%20generation/02-design-ticket-booking-system.md) | Ticket Booking (BookMyShow / District) | `SETNX` distributed locks, Postgres `SELECT FOR UPDATE`, CDC sync to ES, virtual waiting room. | 9 Qs |
| **03** | [`03-design-ecommerce-platform.md`](file:///d:/system%20design%20generation/03-design-ecommerce-platform.md) | E-Commerce Platform (Amazon / Flipkart) | Distributed inventory reservation Saga, transactional outbox, cart sync, flash sales. | 10 Qs |
| **04** | [`04-design-ott-platform-netflix.md`](file:///d:/system%20design%20generation/04-design-ott-platform-netflix.md) | OTT Video Platform (Netflix / Prime) | Chunking & Transcoding DAGs, CDN Open Connect, Dynamic ABR (HLS/DASH), DRM tokens. | 10 Qs |
| **05** | [`05-design-hotel-booking-airbnb.md`](file:///d:/system%20design%20generation/05-design-hotel-booking-airbnb.md) | Hotel Booking (Airbnb / Booking.com) | Date-range calendar indices (`GiST`), optimistic locking, multi-currency escrow. | 10 Qs |
| **06** | [`06-design-food-delivery-zomato-swiggy.md`](file:///d:/system%20design%20generation/06-design-food-delivery-zomato-swiggy.md) | Food Delivery (Zomato / Swiggy) | Dynamic routing ETA, driver batching, live GPS ingestion via Redis Geo, order state machines. | 10 Qs |
| **07** | [`07-design-proximity-search.md`](file:///d:/system%20design%20generation/07-design-proximity-search.md) | Proximity Search (Yelp / Google Maps) | Dynamic QuadTree vs Geohash vs Google S2, memory sharding, bounding box pre-filtering. | 10 Qs |
| **08** | [`08-design-chat-app-whatsapp.md`](file:///d:/system%20design%20generation/08-design-chat-app-whatsapp.md) | Chat Application (WhatsApp / Telegram) | Offline message queues, WebSocket session managers, Cassandra message schema, tick receipts. | 10 Qs |
| **09** | [`09-design-distributed-job-scheduler.md`](file:///d:/system%20design%20generation/09-design-distributed-job-scheduler.md) | Distributed Job Scheduler (Airflow) | Leader election, DAG dependency resolution, heartbeat crash detection, exponential backoff. | 10 Qs |
| **10** | [`10-design-collaborative-editor-google-docs.md`](file:///d:/system%20design%20generation/10-design-collaborative-editor-google-docs.md) | Collaborative Editor (Google Docs) | Operational Transformation (OT) vs CRDTs, central sequencer, cursor broadcasting, snapshotting. | 8 Qs |
| **11** | [`11-design-ride-sharing-uber.md`](file:///d:/system%20design%20generation/11-design-ride-sharing-uber.md) | Ride Sharing (Uber / Lyft) | Dynamic surge pricing via Uber H3 hex cells, driver dispatch lock, WebSocket GPS tracking. | 5 Qs |
| **12** | [`12-design-stock-broker-zerodha.md`](file:///d:/system%20design%20generation/12-design-stock-broker-zerodha.md) | Stock Broker Platform (Zerodha) | Sub-50ms tick streaming, atomic ledger balance checks, order state machine. | 3 Qs |
| **13** | [`13-design-social-media-feed.md`](file:///d:/system%20design%20generation/13-design-social-media-feed.md) | Social Media Feed (Instagram / Facebook) | Hybrid Fan-out (write for normal users, read for celebrities), like counts in Redis, pagination. | 10 Qs |
| **14** | [`14-design-top-k-leaderboard.md`](file:///d:/system%20design%20generation/14-design-top-k-leaderboard.md) | Real-time Leaderboard & Ranking | Redis Sorted Sets (`ZSET`), Count-Min Sketch, YouTube trending velocity decay scoring. | 3 Qs |
| **15** | [`15-design-notification-system.md`](file:///d:/system%20design%20generation/15-design-notification-system.md) | Scalable Notification Service | At-least-once delivery, APNs/FCM provider circuit breakers, priority Kafka queues. | 3 Qs |
| **16** | [`16-design-distributed-logging-splunk.md`](file:///d:/system%20design%20generation/16-design-distributed-logging-splunk.md) | Distributed Logging (Splunk / ELK) | Kafka ingestion buffer, out-of-order log watermarks, Flink stream deduplication, index tiering. | 10 Qs |
| **17** | [`17-design-payment-gateway-stripe.md`](file:///d:/system%20design%20generation/17-design-payment-gateway-stripe.md) | Payment Gateway (Stripe) | PCI-DSS tokenization, idempotency keys, double-entry ledger, bank reconciliation. | 10 Qs |
| **18** | [`18-design-google-drive-dropbox.md`](file:///d:/system%20design%20generation/18-design-google-drive-dropbox.md) | Cloud Storage (Google Drive / Dropbox) | FastCDC / Rabin Fingerprints rolling hash deduplication, S3 multipart resumable uploads. | 4 Qs |
| **19** | [`19-design-online-learning-udemy.md`](file:///d:/system%20design%20generation/19-design-online-learning-udemy.md) | Online Learning (Udemy / Coursera) | Adaptive Bitrate Video (HLS), video resume checkpoints, idempotent course enrollment. | 3 Qs |
| **20** | [`20-design-email-delivery-gmail.md`](file:///d:/system%20design%20generation/20-design-email-delivery-gmail.md) | Email Delivery Engine (Gmail / SendGrid) | Outbox orchestrator, SLA priority routing, SPF / DKIM / DMARC cryptographic validation. | 3 Qs |
| **21** | [`21-design-distributed-rate-limiter.md`](file:///d:/system%20design%20generation/21-design-distributed-rate-limiter.md) | Distributed Rate Limiter | Token Bucket, Leaky Bucket, Sliding Window Counter, Redis Lua atomic synchronization. | 3 Qs |

---

### Part II: Advanced Distributed & High-Scale Systems (08 Core Additions)

| # | Document | Primary Architectural Themes | Core Trade-offs & Focus Areas | Cross Qs |
| :---: | :--- | :--- | :--- | :---: |
| **22** | [`22-design-yelp-proximity.md`](file:///d:/system%20design%20generation/22-design-yelp-proximity.md) | Yelp Proximity & Search | BKD-Trees vs QuadTrees, spatial sharding, location cache invalidation. | 5 Qs |
| **23** | [`23-design-distributed-cache.md`](file:///d:/system%20design%20generation/23-design-distributed-cache.md) | Distributed Cache (Memcached/Redis) | Consistent hashing with virtual nodes, LRU eviction, thundering herd / cache stampede. | 5 Qs |
| **24** | [`24-design-online-auction.md`](file:///d:/system%20design%20generation/24-design-online-auction.md) | Real-Time Online Auction (eBay) | Redis Lua atomic bid processing, proxy bidding, anti-sniping dynamic extension, CDC ledger. | 6 Qs |
| **25** | [`25-design-news-aggregator.md`](file:///d:/system%20design%20generation/25-design-news-aggregator.md) | News Aggregator (Google News) | Distributed web scrapers, SimHash near-duplicate article deduplication, inverted search indexing. | 6 Qs |
| **26** | [`26-design-leetcode-judge.md`](file:///d:/system%20design%20generation/26-design-leetcode-judge.md) | Online Code Judge (LeetCode) | Secure untrusted code execution sandbox (Docker/gVisor), cgroups limits, worker queue scheduling. | 6 Qs |
| **27** | [`27-design-real-time-presence-system.md`](file:///d:/system%20design%20generation/27-design-real-time-presence-system.md) | Real-Time Presence (WhatsApp/Slack) | 50M WebSockets, 1.67M QPS heartbeats, debounce state machine, dynamic viewport subscriptions. | 8 Qs |
| **28** | [`28-design-real-time-ecommerce-recommendation-system.md`](file:///d:/system%20design%20generation/28-design-real-time-ecommerce-recommendation-system.md) | E-Commerce Recommendation (Amazon) | Asynchronous clickstream ingestion, Flink nearline feature vectors, 3-stage funnel (HNSW + DLRM). | 8 Qs |
| **29** | [`29-design-real-time-market-data-ingestion-system.md`](file:///d:/system%20design%20generation/29-design-real-time-market-data-ingestion-system.md) | Financial Market Data Ingestion | 20M ticks/sec UDP multicast, MoldUDP64/ITCH, NACK TCP retransmission, LMAX Disruptor ring buffer. | 8 Qs |

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

