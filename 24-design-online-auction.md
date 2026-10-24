# System Design: Online Auction System (eBay / Sotheby's / GovDeals)

> **Source Reference**: [Dodesimo System Design Notes: Online Auction](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20Online%20Auction.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG standard).  
> **Topic**: Designing a High-Throughput, Strongly Consistent Online Auction Platform with Real-Time Bid Updates, Concurrency Controls, and Fault-Tolerant Storage.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Decouple read-heavy **Auction Service** from write-heavy **Bidding Service**. Route incoming bids to **Apache Kafka partitioned by `auction_id`**—because all bids for the same auction land on the *exact same partition*, they are processed in strict sequential FIFO order, naturally eliminating race conditions! The worker validates that `new_bid > current_max_bid`, updates the database, and publishes to **Redis Pub/Sub**, which broadcasts to multiple **Server-Sent Events (SSE) / WebSocket** servers to push live price updates to watchers in $< 50\text{ms}$.

| Metric / Dimension | Production Target / Value (Dodesimo Scale) |
| :--- | :--- |
| **Concurrent Auctions** | **10,000,000 active auctions** simultaneously running |
| **Bid Throughput** | Average: $1,650\text{ bids/sec}$; Peak: **$15,000\text{ bids/sec}$** |
| **Annual Storage** | 520M auctions/year $\approx$ **$25\text{ TB / year}$** (Sharded PostgreSQL) |
| **Real-Time Latency** | $< 50\text{ms}$ fanout to active auction watchers via SSE |
| **Consistency Model** | **Strong Consistency** for highest bid; Eventual for browsing |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **The Concurrency Trap (Two users bid at the exact same millisecond)**:
   - *Bad Answer*: "I will use database row locking with `SELECT MAX(bid) FOR UPDATE`." (*Trap: Causes massive lock contention, deadlocks, and does not prevent concurrent phantom inserts.*)
   - *Winning Defense*: Use **Apache Kafka partitioned by `auction_id`** . Partitioning by `auction_id` guarantees all bids for that specific auction queue up in a single sequential log, executing one-by-one with zero race conditions!
2. **Real-Time Updates: Polling vs Long Polling vs Server-Sent Events (SSE)**:
   - *Trap*: *"Why not just poll the server every 2 seconds?"*
   - *Defense*: Polling hammers the database with useless queries when prices haven't changed and adds unacceptable latency. Long polling suffers from connection timeout lag and thundering herd re-connections. **Server-Sent Events (SSE)** provide a persistent, lightweight unidirectional HTTP stream where the server pushes new highest bids instantly.
3. **Multi-Server SSE Coordination**:
   - *Trap*: *"If 10,000 users are watching an auction across 10 different web servers, how does Server A notify clients connected to Server B?"*
   - *Defense*: Use **Redis Pub/Sub** as a central message bus. When a new bid is accepted on Server A, it publishes an event to channel `auction:{id}:bids`. All SSE servers subscribed to that channel receive the event and push it down their local client connections.

> **Golden Architectural Rule**: Never mix high-frequency bidding writes with auction catalog reads. Partition bidding queues and database tables strictly by `auction_id` so that contention is isolated per auction.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Post Item for Auction**: Sellers can list an item with a title, description, starting price, start date, and end date (`POST /auctions`).
2. **Place a Bid**: Buyers can place a bid on an active auction. A bid is valid **only if it is strictly higher than the current maximum bid** (`POST /auctions/:id/bids`).
3. **View an Auction & Current Highest Bid**: Buyers can view auction details and see the **current highest bid update in real time** without refreshing the page (`GET /auctions/:id`).
4. **Dynamic Auction Extension (Anti-Sniping)**: If a bid is placed in the closing minutes of an auction, the end time automatically extends by 2 minutes to prevent sniping bots.
5. **Auction Settlement**: When an auction ends, the highest bidder wins; the system initiates payment collection.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Strong Consistency for Bids**: All users must see the exact same highest bid. Two identical winning bids must never be accepted simultaneously.
2. **Real-Time Low Latency**: Highest bid price updates must propagate to all connected viewers in **$< 50\text{ms}$**.
3. **High Availability & Fault Tolerance**: Bids must never be dropped under load spikes (e.g. closing seconds of popular items). Durable queuing guarantees zero lost bids.
4. **Massive Scale**: Must support **10 Million concurrent active auctions** and handle peak bidding surges.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput Math
- **Active Concurrent Auctions**: $10,000,000$ auctions.
- **Average Auction Duration**: 1 week ($7 \text{ days} = 604,800\text{ seconds}$).
- **Average Bids per Auction**: $\approx 100\text{ bids}$ over its lifetime.
- **Total Bids per Week**: $10,000,000 \times 100 = 1,000,000,000\text{ bids/week}$ ($1\text{ Billion bids}$).
- **Average Write QPS**:
  $$\text{Average Bids/sec} = \frac{1,000,000,000\text{ bids}}{604,800\text{ seconds}} \approx 1,650\text{ bids/sec}$$
- **Peak Write QPS ($10\times$ Surge during busy hours / endings)**:
  $$\text{Peak Bids/sec} \approx 15,000 - 20,000\text{ bids/sec}$$

#### Storage Math
- **Auctions per Year**: $10\text{M concurrent} \times 52\text{ weeks} = 520\text{ Million auctions/year}$.
- **Storage per Auction**:
  - Auction metadata: $\approx 1\text{ KB}$
  - 100 Bids $\times 500\text{ bytes} = 50\text{ KB}$
  - Total per auction: $\approx 51\text{ KB}$
- **Annual Storage Footprint**:
  $$\text{Annual Storage} = 520,000,000 \times 51\text{ KB} \approx 26.5\text{ TB / year}$$
- *Conclusion*: A single database cannot handle 15,000 writes/sec or 26.5 TB/year comfortably without connection saturation $\implies$ **Shard the PostgreSQL database by `auction_id`**.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

Separating **Item** from **Auction** is an essential domain modeling principle:
- **Item**: Represents the physical or digital asset listed for sale (name, description, images, condition, seller).
- **Auction**: Represents the time-bounded sales event (starting price, current maximum bid, highest bidder, start/end dates, state). If an auction fails to meet reserve or is relisted, the **Item** entity remains unchanged while a new **Auction** record is created.
- **Bid**: The immutable offer made by a bidder on an active auction.
- **User**: Buyers and sellers participating in the marketplace.

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

Response: 202 Accepted (Async Ingestion)
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
  "item_title": "Vintage 1968 Gibson Les Paul Guitar",
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
│    (Auctions Table)     │◀───────────────────────────────────────│  Topic: `auction-bids`  │
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

### 4.1 Why Separate `Auction Service` from `Bidding Service`?
1. **Independent Scaling**: Auction browsing and creation is read-heavy ($95\%$ reads, $5\%$ writes). Bidding is write-heavy and experiences violent bursts in the final 60 seconds of a listing.
2. **Failure Isolation**: If an auction goes viral and receives 20,000 bids/second, the `Bidding Service` and Kafka queues absorb the surge. The `Auction Service` remains completely healthy so other users can still browse and list items without interruption.
3. **Clean Boundaries**: Bidding contains specialized concurrency logic (race conditions, strict ordering, outbid notifications), whereas Auction Service handles static metadata, categories, and search.

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

### 5.1 The Concurrency & Strong Consistency Deep Dive: Solving the Race Condition

#### The Problem:
Alice and Bob simultaneously bid **\$105** when the current highest bid is **\$100**. Who wins? How do we prevent both from being accepted?

```
         Alice submits $105 ──┐
                              ├────► [ Both read max = $100 simultaneously! ]
         Bob submits $105   ──┘
```

#### The 4 Approaches Evaluated:

| Approach | Implementation | Pros | Cons / Verdict |
| :--- | :--- | :--- | :--- |
| **Approach 1: Row Locking on Bids Table** | `BEGIN; SELECT MAX(bid) FROM bids WHERE auction_id = X FOR UPDATE; ... COMMIT;` | Traditional ACID transaction. | ❌ **Rejected**: Does not prevent concurrent phantom inserts; terrible performance under high contention ($< 500\text{ writes/s}$). |
| **Approach 2: In-Memory Redis Atomic Check** | Redis in-memory check: validates `new_bid > current` before touching DB. | Sub-millisecond latency ($< 1\text{ms}$). | ⚠️ **Good**: Extremely fast, but Redis state must be kept consistent with DB on crash. |
| **Approach 3: Optimistic Concurrency Control (OCC) in DB** | `UPDATE auctions SET current_max_bid = 105, version = version + 1 WHERE auction_id = X AND current_max_bid < 105 AND version = 2` | Clean, no external cache needed; DB enforces truth. | ⚠️ **Good for moderate scale**: First write succeeds (returns 1 row affected); second write fails (returns 0 rows affected). |
| **Approach 4: Kafka Partitioned by `auction_id` (The Dodesimo Champion Pattern)** | Bids published to Kafka with `key = auction_id`. Dedicated worker processes partition sequentially. | **Best for High Scale**: Absorbs massive traffic bursts, guarantees strict FIFO order per auction, zero locks. | ✅ **Chosen Production Standard**: Bids are serialized naturally by Kafka partition before hitting the DB. |

---

### 5.2 Real-Time Price Streaming: Polling vs. Long Polling vs. Server-Sent Events (SSE)

| Dimension | Short Polling | Long Polling | Server-Sent Events (SSE) | WebSockets |
| :--- | :--- | :--- | :--- | :--- |
| **Mechanism** | Client calls `GET` every 2s | Client calls `GET`, server holds connection open until new bid | Server pushes updates over single open HTTP connection | Full-duplex persistent TCP socket |
| **Latency** | 0 – 2 seconds (stale) | $< 100\text{ms}$ | **$< 10\text{ms}$ (instant)** | $< 10\text{ms}$ (instant) |
| **Server Overhead** | High (constant HTTP handshakes) | Moderate (thundering herd on reconnect) | **Very Low (native HTTP/2 multiplexing)** | Low (connection stateful memory) |
| **Direction** | Client $\to$ Server | Client $\to$ Server | **Server $\to$ Client (Unidirectional)** | Bidirectional |
| **Verdict** | ❌ Too slow & heavy | ❌ Complex timeout edge cases | ✅ **Optimal Choice for Auction Viewing** | ⚠️ Overkill (bids can be sent via standard HTTP POST) |

#### How SSE Works with Multi-Server Redis Pub/Sub:
1. Client opens `GET /auctions/123/bid-stream`. Server adds client's response object to an in-memory list: `connections["123"].add(client)`.
2. When a new bid is accepted, the consumer publishes `{auction_id: 123, max_bid: 105}` to Redis Pub/Sub channel `auction:123`.
3. Every SSE server subscribed to `auction:123` receives the event.
4. Each server iterates through its local connections for `123` and writes:
   ```
   data: {"max_bid": 105, "bidder": "Alice"}\n\n
   ```
5. Total end-to-end fanout latency: **$< 30\text{ms}$**.

---

### 5.3 Scaling to 10 Million Concurrent Auctions: Database Sharding

- **The Problem**: 10M concurrent auctions with 15,000 peak writes/sec exceed the capabilities of a single PostgreSQL primary.
- **The Solution: Shard by `auction_id`**:
  - Partition key = `hash(auction_id) % num_shards`.
  - All data for Auction #123 (`auctions` row and all related `bids` rows) resides on the **exact same database shard**.
  - Transactions on Auction #123 never require distributed two-phase commits (2PC) across multiple database instances!
  - As traffic doubles, we add more shards linearly without changing query architecture.

---

### 5.4 Dynamic Auction Extension: Anti-Sniping Soft-Close

- **The Problem**: Sniping bots place bids with 0.1 seconds remaining, preventing human participants from countering.
- **The Solution: Dynamic Soft-Close**:
  ```python
  def check_and_extend_auction(auction, new_bid_time):
      time_left = auction.end_date - new_bid_time
      if time_left < timedelta(minutes=2):
          # Extend auction by 2 additional minutes
          auction.end_date = auction.end_date + timedelta(minutes=2)
          db.save(auction)
          # Publish extension event to watchers
          redis_pubsub.publish(f"auction:{auction.id}", {"event": "AUCTION_EXTENDED", "new_end_date": auction.end_date})
  ```
- This simulates live auctioneers ("Going once... going twice..."): any last-second bid resets the countdown clock, maximizing seller revenue and ensuring fairness.

---

### 5.5 Post-Auction Payment Failure Handling (What if Winner Defaults?)

1. When the auction countdown reaches zero, status changes to `CLOSED`.
2. An automated settlement job sends the winner a payment link with a **24-hour payment window**.
3. **If Payment Succeeds**: Item transitions to `SOLD`, logistics fulfillment begins.
4. **If Payment Fails or 24-Hour Window Expires**:
   - Winner is flagged and penalized.
   - The system retrieves the **Second-Highest Bidder** from the `bids` table and sends them a Second-Chance Offer at their highest bid amount.
   - If rejected, the item automatically re-lists or returns to the seller.

---

## 6. Step 6: Comparative Trade-Off Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Concurrency Control** | **Kafka Partitioning by `auction_id`** | PostgreSQL Row Locks (`SELECT FOR UPDATE`) | Row locks bottleneck at 500 writes/s and cause deadlocks. Kafka partitions serialize bids per auction into a clean sequential queue, eliminating race conditions. |
| **Real-Time Client Streaming** | **Server-Sent Events (SSE) + Redis Pub/Sub** | Client Polling every 2s | Polling creates massive redundant HTTP requests and stale views. SSE is lightweight, unidirectional, and pushes updates in $<30\text{ms}$. |
| **Database Architecture** | **PostgreSQL Sharded by `auction_id`** | Single Global PostgreSQL Database | A single node cannot sustain 15,000 writes/sec or 25TB/year. Sharding by `auction_id` guarantees localized ACID transactions per auction without distributed joins. |
| **Service Boundaries** | **Separate Bidding from Auction Service** | Single Monolithic Auction Service | Bidding traffic is spiky and write-heavy; auction browsing is read-heavy. Separation isolates failure domains and enables independent autoscaling. |
| **Anti-Sniping Engine** | **Dynamic Soft-Close (+2 mins if bid near end)** | Strict Hard-Close | Fixed hard-close encourages bot sniping. Soft-close simulates physical auction rooms, maximizing revenue and fairness for human bidders. |

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

