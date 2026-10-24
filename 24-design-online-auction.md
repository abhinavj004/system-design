# System Design: Online Auction System (eBay / Sotheby's / GovDeals)

> **Source Reference**: [Dodesimo System Design Notes: Online Auction](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20Online%20Auction.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG standard).  
> **Topic**: Designing a High-Throughput, Strongly Consistent Online Auction Platform with Real-Time Bid Updates, Concurrency Controls, and Fault-Tolerant Storage.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Decouple read-heavy **Auction Service** from write-heavy **Bidding Service**. Route incoming bids to **Apache Kafka partitioned by `auction_id`**—because all bids for the same auction land on the *exact same partition*, they are processed in strict sequential FIFO order, naturally eliminating race conditions! The consumer worker validates that `new_bid > current_max_bid`, updates the database, and publishes to **Redis Pub/Sub**, which broadcasts to multiple **Server-Sent Events (SSE)** servers to push live price updates to watchers in $< 50\text{ms}$.

| Metric / Dimension | Production Target / Value (Dodesimo Scale) |
| :--- | :--- |
| **Concurrent Auctions** | **10,000,000 active auctions** simultaneously running |
| **Bid Throughput** | Average: **$1,400\text{ bids/sec}$**; Peak ($10\times$ surge): **$15,000\text{ bids/sec}$** |
| **Annual Storage** | 520M auctions/year $\times 51\text{ KB} \approx$ **$25\text{ TB / year}$** (Sharded PostgreSQL) |
| **Real-Time Latency** | $< 50\text{ms}$ fanout to active auction watchers via SSE |
| **Consistency Model** | **Strong Consistency** for highest bid; Eventual for browsing |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **The Concurrency Trap (Two users bid at the exact same millisecond)**:
   - *Bad Answer*: "I will use database row locking with `SELECT MAX(bid) FOR UPDATE`." (*Trap: Causes massive lock contention, deadlocks, and does not prevent concurrent phantom inserts.*)
   - *Winning Defense*: Use **Apache Kafka partitioned by `auction_id`**. Partitioning by `auction_id` guarantees all bids for that specific auction queue up in a single sequential log, executing one-by-one with zero race conditions!
2. **Real-Time Updates: Polling vs Long Polling vs Server-Sent Events (SSE)**:
   - *Trap*: *"Why not just poll the server every 2 seconds?"*
   - *Defense*: Polling hammers the database with useless queries when prices haven't changed and adds unacceptable latency. Long polling suffers from connection timeout lag and thundering herd re-connections. **Server-Sent Events (SSE)** provide a persistent, lightweight unidirectional HTTP stream where the server pushes new highest bids instantly.
3. **Multi-Server SSE Coordination**:
   - *Trap*: *"If 10,000 users are watching an auction across 10 different web servers, how does Server A notify clients connected to Server B?"*
   - *Defense*: Use **Redis Pub/Sub** as a central message bus. When a new bid is accepted on Server A, it publishes an event to channel `auction:{id}`. All SSE servers subscribed to that channel receive the event and push it down their local client connections.

> **Golden Architectural Rule**: Never mix high-frequency bidding writes with auction catalog reads. Partition bidding queues and database tables strictly by `auction_id` so that contention is isolated per auction.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Post Item for Auction**: Sellers can list an item with a starting price, start date, and end date (`POST /auctions`).
2. **Place a Bid**: Buyers can place a bid on an active auction. The system **must only accept bids if they are strictly higher than the current highest bid** (`POST /auctions/:id/bids`).
3. **View an Auction & Current Highest Bid**: Buyers can view auction details and see the **current highest bid update in real time** without refreshing the page (`GET /auctions/:id`).
4. **Dynamic Auction Extension (Anti-Sniping)**: If a bid arrives in the closing minutes of an auction, the end time automatically extends by 2 minutes to prevent sniping bots.
5. **Auction Settlement & Purchasing**: When an auction ends, the winning bidder is notified to complete payment. If payment fails, the system automatically offers the item to the next highest bidder.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Strong Consistency for Bids**: All users must see the exact same highest bid. Two identical winning bids must never be accepted simultaneously.
2. **Fault Tolerant & Durable**: Bids must never be lost, dropped, or corrupted, even during extreme traffic spikes or node crashes.
3. **Real-Time Display**: Display the current highest bid in real time ($< 50\text{ms}$ propagation latency).
4. **Massive Scale**: Must support **10 Million concurrent active auctions** and handle peak bidding bursts.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput Math (Direct Dodesimo Formula)
- **Active Concurrent Auctions**: $10,000,000$ auctions.
- **Average Auction Duration**: 1 week ($7 \text{ days} = 604,800\text{ seconds}$).
- **Average Bids per Auction**: $\approx 100\text{ bids}$ over its lifetime.
- **Total Bids per Week**: $10,000,000 \times 100 = 1,000,000,000\text{ bids/week}$ ($1\text{ Billion bids}$).
- **Average Write QPS**:
  $$\text{Average Bids/sec} = \frac{1,000,000,000\text{ bids}}{604,800\text{ seconds}} \approx 1,400\text{ bids/sec}$$
- **Peak Write QPS ($10\times$ Surge during busy closing seconds)**:
  $$\text{Peak Bids/sec} \approx 14,000 - 15,000\text{ bids/sec}$$

#### Storage Math
- **Auctions per Year**: $10\text{M concurrent} \times 52\text{ weeks} = 520\text{ Million auctions/year}$.
- **Storage Breakdown per Auction**:
  - `Auction` metadata: $\approx 1\text{ KB}$
  - `Bid` records: $100\text{ bids} \times 500\text{ bytes} = 50\text{ KB}$
  - Total per auction: $\approx 51\text{ KB}$
- **Annual Storage Footprint**:
  $$\text{Annual Storage} = 520,000,000 \times 51\text{ KB} \approx 26.5\text{ TB / year} \approx 25\text{ TB / year}$$
- *Conclusion*: While $25\text{ TB}$ easily fits on modern NVMe drives, a single database instance cannot sustain $15,000\text{ writes/sec}$ without connection saturation $\implies$ **Shard the PostgreSQL database by `auction_id`**.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

Separating **Item** from **Auction** is an essential domain modeling principle:
- **`Item`**: Represents the physical or digital asset listed for sale (name, description, images, condition, seller).
- **`Auction`**: Represents the time-bounded sales event (starting price, current maximum bid, highest bidder, start/end dates, state). If an auction fails to meet reserve or is relisted, the **Item** entity remains unchanged while a new **Auction** record is created.
- **`Bid`**: The immutable offer made by a bidder on an active auction.
- **`User`**: Buyers and sellers participating in the marketplace.

```
┌──────────────────┐        1:N        ┌──────────────────┐        1:1        ┌──────────────────┐
│       User       │──────────────────▶│       Item       │──────────────────▶│     Auction      │
├──────────────────┤                   ├──────────────────┤                   ├──────────────────┤
│ user_id (PK)     │                   │ item_id (PK)     │                   │ auction_id (PK)  │
│ name             │                   │ seller_id (FK)   │                   │ item_id (FK)     │
│ email            │                   │ name             │                   │ starting_price   │
│ created_at       │                   │ description      │                   │ current_max_bid  │
└──────────────────┘                   │ image_url        │                   │ highest_bidder_id│
         │                             └──────────────────┘                   │ start_date       │
         │ 1:N                                                                │ end_date         │
         ▼                                                                    │ status (ACTIVE..)│
┌──────────────────┐                                                          │ version (for OCC)│
│       Bid        │                                                          └──────────────────┘
├──────────────────┤                                                                   ▲
│ bid_id (PK)      │                                                                   │
│ auction_id (FK)  │───────────────────────────────────────────────────────────────────┘
│ bidder_id (FK)   │                                  1:N
│ bid_amount       │
│ status (ACC/REJ) │
│ created_at       │
└──────────────────┘
```

### Relational Schema (PostgreSQL)

```sql
CREATE TABLE items (
    item_id VARCHAR(64) PRIMARY KEY,
    seller_id VARCHAR(64) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    image_url VARCHAR(512),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE auctions (
    auction_id VARCHAR(64) PRIMARY KEY,
    item_id VARCHAR(64) NOT NULL REFERENCES items(item_id),
    starting_price DECIMAL(12,2) NOT NULL,
    current_max_bid DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    highest_bidder_id VARCHAR(64),
    start_date TIMESTAMPTZ NOT NULL,
    end_date TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, CLOSED, CANCELLED
    version INT DEFAULT 1,               -- Used for Optimistic Concurrency Control
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE bids (
    bid_id VARCHAR(64) PRIMARY KEY,
    auction_id VARCHAR(64) NOT NULL REFERENCES auctions(auction_id),
    bidder_id VARCHAR(64) NOT NULL,
    bid_amount DECIMAL(12,2) NOT NULL,
    status VARCHAR(20) NOT NULL,          -- ACCEPTED, REJECTED
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast lookup of bid history per auction in chronological order
CREATE INDEX idx_bids_auction_created ON bids(auction_id, created_at DESC);
```

---

## 3. Step 3: API & Interface Design

### 3.1 Create Auction & Item
```http
POST /api/v1/auctions
Content-Type: application/json

Request:
{
  "item": {
    "name": "Vintage 1968 Gibson Les Paul Guitar",
    "description": "Mint condition electric guitar with original case.",
    "image_url": "https://cdn.example.com/items/gibson_1968.jpg"
  },
  "starting_price": 5000.00,
  "start_date": "2026-11-01T10:00:00Z",
  "end_date": "2026-11-08T10:00:00Z"
}

Response: 201 Created
{
  "auction_id": "auc_77812",
  "item_id": "itm_55104",
  "status": "ACTIVE",
  "current_max_bid": 5000.00
}
```

### 3.2 Place a Bid
```http
POST /api/v1/auctions/{auction_id}/bids
Content-Type: application/json

Request:
{
  "bidder_id": "usr_9921",
  "bid_amount": 5250.00
}

Response: 202 Accepted (Async Queue Ingestion)
{
  "status": "QUEUED",
  "message": "Bid submitted for evaluation",
  "auction_id": "auc_77812"
}
```

### 3.3 Get Auction Details & Highest Bid
```http
GET /api/v1/auctions/{auction_id}

Response: 200 OK
{
  "auction_id": "auc_77812",
  "item": {
    "item_id": "itm_55104",
    "name": "Vintage 1968 Gibson Les Paul Guitar",
    "description": "Mint condition electric guitar with original case.",
    "image_url": "https://cdn.example.com/items/gibson_1968.jpg"
  },
  "current_max_bid": 5250.00,
  "highest_bidder_id": "usr_9921",
  "end_date": "2026-11-08T10:00:00Z",
  "status": "ACTIVE"
}
```

### 3.4 Real-Time Highest Bid Stream (Server-Sent Events)
```http
GET /api/v1/auctions/{auction_id}/bid-stream
Accept: text/event-stream

Stream Frames Pushed by Server:
event: new-highest-bid
data: {"auction_id": "auc_77812", "new_max_bid": 5500.00, "bidder_id": "usr_3310", "timestamp": "2026-11-08T09:59:45Z"}
```

---

## 4. Step 4: High-Level Architecture (HLD)

### Component Interaction Topology

```
[ Bidders & Watchers ]
        │
        ├─── (HTTP POST /bids) ────────────────────────────────────────────────────────┐
        │                                                                              │
        ├─── (SSE /bid-stream: Real-Time Stream) ──┐                                   │
        │                                          │                                   │
        ▼                                          ▼                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 API Gateway & Load Balancer                                  │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
          │                                                                  │
  (Read Auction Details)                                            (High-Write Bids)
          │                                                                  │
          ▼                                                                  ▼
┌─────────────────────────┐                                        ┌─────────────────────────┐
│     Auction Service     │                                        │     Bidding Service     │
│ (Metadata, Search, CRUD)│                                        │  (Validates & Enqueues) │
└─────────────────────────┘                                        └─────────────────────────┘
          │                                                                  │
          │ Read Lookups                                                     │ Produce with key = auction_id
          ▼                                                                  ▼
┌─────────────────────────┐                                        ┌─────────────────────────┐
│  Sharded PostgreSQL DB  │                                        │ Apache Kafka Event Bus  │
│ (Auctions & Items Table)│◀───────────────────────────────────────│  Topic: `auction-bids`  │
└─────────────────────────┘      Write Accepted Bids to DB         └─────────────────────────┘
          ▲                                                                  │
          │                                                                  │ Partition Consumer
          │ Update Max Bid                                                   ▼
          │                                                        ┌─────────────────────────┐
          │                                                        │   Bid Consumer Worker   │
          │                                                        │(Checks bid > current_max)
          │                                                        └─────────────────────────┘
          │                                                                  │
          │                                                                  │ Publish New Max Bid
          │                                                                  ▼
          │                                                        ┌─────────────────────────┐
          │                                                        │   Redis Pub/Sub Bus     │
          │                                                        └─────────────────────────┘
          │                                                                  │
          │                                                 Broadcast to all │ Server Nodes
          │                                                                  ▼
          │                                                        ┌─────────────────────────┐
          │                                                        │   SSE Stream Servers    │
          │                                                        │ (Pushes to Client UI)   │
          └────────────────────────────────────────────────────────┴─────────────────────────┘
```

---

### 4.1 Why is Bidding Different than Auction? (Service Separation)

In an interview, justify splitting `Bidding Service` from `Auction Service` using these 3 foundational arguments:
1. **Independent Scaling**: Bidding traffic is dramatically higher and more bursty than auction creation and browsing ($15,000\text{ writes/sec}$ peak vs standard reads). They have fundamentally different scaling profiles.
2. **Isolation of Concerns**: Bidding involves complex race conditions, strict validation, and real-time push streaming. Auction Service is standard CRUD (title, description, images). Isolating them maintains clean service boundaries.
3. **High-Write vs. High-Read Optimization**: Bidding requires an ultra-optimized, high-write ingestion path (Kafka queue buffers). If a viral auction spikes, the Bidding Service absorbs the load while the Auction Service remains completely healthy for users browsing other items.

---

### 4.2 Step-by-Step Architectural Evolution (How to Drive the Interview Whiteboard)

#### Phase 1: The Naive Baseline MVP (Monolith + Single Database)
- **What to draw**: `Client` ──► `API Server` ──► `PostgreSQL Database`.
- **How to explain the flow**:
  - *"In the simplest design, when a user bids, the API server executes `SELECT current_max_bid FROM auctions WHERE id = :id FOR UPDATE`, checks if `new_bid > current_max_bid`, updates the row, and inserts into `bids`."*
- **Bottlenecks identified (Why it breaks)**:
  1. *Row Lock Contention*: In the final 10 seconds of a popular auction, 5,000 users bid at once. Locking the same row in PostgreSQL causes severe connection starvation, deadlocks, and dropped bids.
  2. *Polling Meltdown*: 50,000 watchers polling `GET /auctions/:id` every 2 seconds crush the database CPU.

#### Phase 2: Decouple Services & Buffer Spikes with Apache Kafka
- **What to add next**: Split into `Auction Service` and `Bidding Service`, with **Apache Kafka** between Bidding Service and Database.
- **How to explain the transition**:
  - *"To prevent database deadlocks and handle bursts without dropping bids, the Bidding Service pushes every bid to Kafka and immediately returns `HTTP 202 Accepted` to the user. Kafka buffers the spike safely."*

#### Phase 3: Enforcing Strict Bid Ordering via Kafka Partitioning
- **What to add next**: Partition the Kafka topic by **`key = auction_id`**.
- **How to explain the transition**:
  - *"Here is the secret to concurrency: by hashing on `auction_id`, **all bids for the same auction land on the exact same Kafka partition**. Kafka guarantees strict FIFO ordering per partition. A single consumer worker processes bids for that auction one-by-one. Race conditions disappear completely without database locks!"*

#### Phase 4: Replace Polling with Server-Sent Events (SSE)
- **What to add next**: Add **SSE Stream Endpoints** (`GET /auctions/:id/bid-stream`).
- **How to explain the transition**:
  - *"Instead of clients polling the database, clients open a persistent SSE connection. The server holds an open HTTP pipe and pushes new bid values instantly only when the price actually changes."*

#### Phase 5: Multi-Server Coordination with Redis Pub/Sub
- **What to add next**: Connect SSE servers via **Redis Pub/Sub**.
- **How to explain the transition**:
  - *"When we scale horizontally, User A is on Server 1 and User B is on Server 2. When the consumer processes a new winning bid, it publishes to Redis channel `auction:{id}`. All SSE servers receive the message and push it to their local connected clients."*

#### Phase 6: Shard the Database by `auction_id`
- **What to add next**: Shard PostgreSQL across multiple database nodes by `auction_id`.
- **How to explain the transition**:
  - *"With 10M concurrent auctions producing 15,000 writes/sec and 25TB/year, a single DB node will fail. We shard the database using `hash(auction_id)`, spreading write load evenly across multiple PostgreSQL instances."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: Strong Consistency (What if two people simultaneously update the bid?)

#### The Core Problem:
Alice and Bob simultaneously bid **\$105** when the current highest bid is **\$100**. Who wins? How do we prevent both from being accepted?

```
         Alice submits $105 ──┐
                              ├────► [ Both read max = $100 simultaneously! ]
         Bob submits $105   ──┘
```

#### The Solutions Evaluated (Direct Dodesimo Progression):

#### Solution 1 (Bad): Row Locking with Bids Query
- **Implementation**:
  ```sql
  BEGIN;
  -- Lock all bid rows for auction
  SELECT MAX(bid_amount) FROM bids WHERE auction_id = 'auc_123' FOR UPDATE;
  -- Compare new bid against max
  -- If higher:
  INSERT INTO bids (auction_id, bidder_id, bid_amount, status) VALUES ('auc_123', 'usr_alice', 105.00, 'ACCEPTED');
  COMMIT;
  ```
- **Why it Fails (Challenges)**:
  1. **Doesn't prevent concurrent inserts (Phantom Reads)**: `FOR UPDATE` only locks *existing* rows. If two transactions execute simultaneously, both can verify against existing rows and both insert \$105!
  2. **Severe Locking Bottleneck**: Row locks choke database throughput to $< 500\text{ writes/sec}$.

#### Solution 2 (Good): Cache the Max Bid Externally in Redis
- **Implementation**:
  - Store `auction:{id}:max_bid` in Redis.
  - Read cache for an auction from Redis.
  - If new bid is higher, update cache with new max bid.
  - Write the bid to the database with ACCEPTED/REJECTED status.
- **Issues & Trade-offs**:
  - Read and write of cache must be atomic to avoid race conditions. Redis is single-threaded, so it supports atomic operations.
  - **Cache vs. Database Consistency**:
    - *Option A*: Accept Redis as the source of truth and write to DB asynchronously.
    - *Option B (Optimistic Concurrency & Retry)*: Update cache atomically; if the DB write fails, roll back cache.
    - *Option C*: Write to DB first, then update cache. If cache fails, invalidate cache entry and repopulate on next read.

#### Solution 3 (Better): Cache within the Database Table (Optimistic Concurrency Control - OCC)
- **Implementation**:
  - Add a `current_max_bid` and `version` column directly to the `auctions` table.
  - Read the max bid from the `auctions` table.
  - Write the new bid to the DB using an atomic conditional update (OCC / CAS):
    ```sql
    UPDATE auctions
    SET current_max_bid = 105.00,
        version = version + 1
    WHERE auction_id = 'auc_123'
      AND version = 1
      AND 105.00 > current_max_bid;
    ```
  - **How it works**:
    - Alice's query updates 1 row $\implies$ **ACCEPTED**.
    - Bob's query sees `version` is now 2 or `105.00 > 105.00` is FALSE $\implies$ updates 0 rows $\implies$ **REJECTED**.

#### Solution 4 (Champion / Production Scale): Apache Kafka Partitioned by `auction_id`
- **Why this is the best**: Instead of fighting concurrency at the database layer, **serialize bids before they touch the database**!
- By assigning `key = auction_id`, Kafka places all bids for an auction in the exact same partition. A single consumer thread executes bids sequentially. Race conditions vanish naturally!

---

### 5.2 Deep Dive 2: Fault Tolerance & Durability via Message Queue (Kafka)

#### Why add a Message Queue?
1. **Immediate Write Durability**: When a bid comes in, write to the queue immediately. Even if the backend bidding service fails, bids are safely preserved in the queue.
2. **Buffer Against Load Spikes**: During the final seconds of an auction, traffic surges from 1,400 to 15,000+ bids/sec. Without a message queue, the database would crash or drop bids. A queue buffers the spike so consumers process at a sustainable rate.
3. **Enforce Ordering per Auction**: **Partitioning by `auctionId`** ensures all bids for the same auction land on the **exact same partition**, guaranteeing in-order processing. This is critical for fairness.

#### The End-to-End Queue Flow:
```
1. User submits bid (POST /auctions/:id/bids)
2. API Gateway routes request to Bidding Producer
3. Producer writes bid to Kafka topic `auction-bids` with key = auction_id
4. Kafka acknowledges write -> Server returns HTTP 202 "Bid Received" to User
5. Bid Consumer Worker pulls from partition in strict FIFO order
6. If bid > current_max_bid:
     - Write to DB as ACCEPTED
     - Publish new max bid to Redis Pub/Sub
   Else:
     - Write to DB as REJECTED
```

---

### 5.3 Deep Dive 3: How to Display Highest Current Bid in Real Time?

#### Why Short Polling Fails:
- **Too Slow**: Polling every 2–5 seconds is not fast enough for an active auction where prices change every 100ms.
- **Inefficient**: The client hits the database on every single request, even when the maximum bid has not changed (wasteful I/O).

#### Long Polling & Its 4 Critical Limitations:
- *How it works*: Client opens a connection; the server holds the connection open until a new bid arrives or timeout occurs. When a new bid is accepted, the server responds to all waiting requests. The client immediately initiates a new long-poll.
- **Why Long Polling has serious problems**:
  1. **High Server Resource Consumption**: The server must maintain thousands of open HTTP connections for popular auctions.
  2. **Thundering Herd**: When a new bid arrives, all 10,000 waiting clients are answered simultaneously, and all 10,000 immediately reconnect at the exact same millisecond!
  3. **Timeout Latency**: If a bid arrives right after a client's long-poll times out, the client won't see the update until its new connection completes.
  4. **Horizontal Scaling Difficulty**: Each server maintains its own set of connections; redistributing connections across servers is complex.

#### The Best Solution: Server-Sent Events (SSE) with Redis Pub/Sub
- **Why SSE?**:
  - **Unidirectional channel** from server to client. The server pushes updates whenever the price changes without any client polling.
  - Native browser support (`EventSource`), runs over standard HTTP/2, and automatically reconnects on network drops.
- **Server Implementation**:
  - The server maintains an in-memory map of auction IDs to active connections:
    ```java
    Map<String, Set<SseEmitter>> auctionConnections = new ConcurrentHashMap<>();
    ```
- **The Multi-Server Coordination Problem & Redis Pub/Sub Solution**:
  - *Problem*: When the user base grows, multiple servers handle SSE connections. If User A is connected to Server 1 and a new bid arrives on Server 2, Server 2 does not know about Server 1's connections!
  - *Solution*: Use **Redis Pub/Sub** where all SSE servers subscribe to `auction:*`.
  - When Server 2 accepts a bid for auction #123, it publishes to Redis channel `auction:123`.
  - Server 1 receives the notification from Redis, checks its local `auctionConnections["123"]`, and pushes the new bid down the wire to User A!

```
[ New Bid Accepted on Server 2 ]
             │
             ▼
[ Redis Pub/Sub Topic `auction:123` ]
             │
      ┌──────┴───────────────────────────┐
      ▼                                  ▼
[ SSE Server 1 ]                  [ SSE Server 2 ]
      │                                  │
      ▼ Pushes to local clients          ▼ Pushes to local clients
[ User A (Browser) ]              [ User B (Browser) ]
```

---

### 5.4 Deep Dive 4: Supporting 10 Million Concurrent Auctions (Scaling Database & Queue)

1. **Message Queue Parallelism**:
   - By partitioning Kafka by `auctionId`, different consumer instances process bids for different auctions completely in parallel.
2. **Stateless Horizontally Scalable Bidding Service**:
   - The Bidding Service is stateless. We autoscale instances based on memory and CPU utilization to handle burst traffic.
3. **Database Sharding by `auction_id`**:
   - Storage math: 10M concurrent auctions $\times$ 52 weeks = 520M auctions/year $\implies \mathbf{25\text{ TB/year}}$.
   - Throughput: Peak **$15,000\text{ writes/sec}$**.
   - Because 15k writes/sec exceeds a single PostgreSQL node, we **shard the database by `auction_id`**.
   - All queries and updates for Auction #123 hit the same shard. Sharding eliminates cross-node transactions and distributes disk I/O evenly across 10–20 PostgreSQL instances.

---

### 5.5 Deep Dive 5: Dynamic Auction End Times (Anti-Sniping Soft Close)

- **The Problem**: Sniping bots place bids with 0.5 seconds remaining, preventing human participants from countering.
- **The Solution**: Maintain `end_date` in the `auctions` table.
- When a bid arrives, check if it falls within the final 2 minutes:
  ```python
  if auction.end_date - now < timedelta(minutes=2):
      auction.end_date = now + timedelta(minutes=2)
      # Notify all active viewers via Redis Pub/Sub
      redis_pubsub.publish(f"auction:{auction.id}", {"event": "AUCTION_EXTENDED", "new_end_date": auction.end_date})
  ```
- **Auction Closure Poller**: A scheduled cron or delay task runs periodically to check if `end_date <= NOW()` and marks the auction as `CLOSED`.

---

### 5.6 Deep Dive 6: Purchasing & Handling Winner Payment Defaults

- When an auction expires:
  1. The system marks the listing as `CLOSED`.
  2. An automated notification sends an invoice to the highest bidder with a **24-hour payment deadline**.
  3. **If Payment Fails / User Defaults**:
     - The winner account is penalized.
     - The system queries the `bids` table for the **Second-Highest Bidder** and automatically sends them a Second-Chance Offer.
     - If rejected, the item is returned to the seller or scheduled for relisting.

---

### 5.7 Deep Dive 7: Bid History

- Users want to see the full chronological history of bids for an item.
- The `bids` table has a composite index:
  ```sql
  CREATE INDEX idx_bids_auction_created ON bids(auction_id, created_at DESC);
  ```
- Clients fetch paginated bid history using keyset pagination:
  ```sql
  SELECT bid_id, bidder_id, bid_amount, created_at 
  FROM bids 
  WHERE auction_id = 'auc_123' 
  ORDER BY created_at DESC 
  LIMIT 20;
  ```

---

## 6. Step 6: Comparative Trade-Off Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Concurrency Control** | **Kafka Partitioning by `auction_id`** | PostgreSQL Row Locks (`SELECT ... FOR UPDATE`) | Row locks cause deadlocks and do not prevent phantom inserts. Kafka partitions serialize bids per auction into a clean sequential queue, eliminating race conditions naturally. |
| **Real-Time Client Streaming** | **Server-Sent Events (SSE) + Redis Pub/Sub** | Short Polling or Long Polling | Polling hammers the DB with redundant queries; long polling causes thundering herds on reconnect. SSE is lightweight, unidirectional, and pushes updates in $<30\text{ms}$. |
| **Database Architecture** | **PostgreSQL Sharded by `auction_id`** | Single Global PostgreSQL Instance | A single node cannot sustain 15,000 writes/sec or 25TB/year. Sharding by `auction_id` guarantees localized ACID transactions per auction without distributed joins. |
| **Service Boundaries** | **Separate Bidding from Auction Service** | Single Monolithic Auction Service | Bidding traffic is spiky and write-heavy; auction browsing is read-heavy. Separation isolates failure domains and enables independent autoscaling. |
| **Anti-Sniping Engine** | **Dynamic Soft-Close (+2 mins if bid near end)** | Strict Hard-Close (Fixed Timestamp) | Fixed hard-close encourages bot sniping. Soft-close simulates physical auction rooms, maximizing revenue and fairness for human bidders. |

---

## 7. Key Interview Follow-Ups & Quick Answers

### Q1: How do you guarantee the highest bid is never lost if a server crashes?
- **Answer**: Bids are written to Apache Kafka with replication factor of 3 and `acks=all` before acknowledging the user. Even if the consumer worker or database crashes, the bid is safely persisted in the Kafka commit log and will be replayed upon worker restart.

### Q2: Why is Server-Sent Events (SSE) preferred over WebSockets for this problem?
- **Answer**: Price updates are **strictly unidirectional** (server $\to$ client). SSE runs over standard HTTP/2, supports automatic re-connection out of the box, passes seamlessly through corporate firewalls, and consumes fewer server resources than maintaining stateful duplex WebSocket handshakes. Bids are simply sent via standard HTTP `POST /bids`.

### Q3: How do you prevent two users from winning with the exact same bid?
- **Answer**: Because bids are queued in the same Kafka partition by `auction_id`, they arrive sequentially at the consumer. The worker checks `if bid.amount > current_max_bid`. The first bid accepted increments `current_max_bid`; the second bid arriving with the identical amount fails the strictly-greater check and is immediately marked `REJECTED`.

### Q4: How do you handle millions of auctions expiring at the exact same minute?
- **Answer**: Auctions are scheduled into a **Redis Sorted Set (`ZSET`)** where the score is the `end_date` epoch timestamp. A pool of distributed closer workers polls `ZRANGEBYSCORE 0 {current_time} LIMIT 1000` to batch-process expired auctions without scanning the primary relational database.
