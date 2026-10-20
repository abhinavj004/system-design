# System Design: Online Auction System (eBay / Sotheby's / GovDeals)

> **Source Reference**: [Dodesimo System Design Notes: Online Auction](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20Online%20Auction.md)  
> **Methodology**: Formatted strictly according to the [Master 5-Step Architectural Blueprint](file:///d:/system%20design%20generation/00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG standard).  
> **Topic**: Designing a High-Throughput, Strongly Consistent, Ultra-Low Latency Online Auction Platform Supporting Flash Bidding Contention, Anti-Sniping Soft Closes, and Secure Settlement.

---

## 1. Step 1: Requirements Gathering & Estimations

An online auction platform facilitates the real-time sale of goods and services where participants submit competitive bids. Unlike standard e-commerce carts with static inventory checkout, auctions are characterized by **extreme concurrent write contention** on a single database record during the closing seconds, **strict linearizable bid ordering**, and **sub-50ms fan-out broadcasting** to thousands of simultaneous watchers.

```
       Seller                                  Bidders (1 to N)
         │                                            │
         ▼                                            ▼
+──────────────────+                       +──────────────────────+
|  List Item & Set |                       | Watch Real-Time Bids |
| Starting/Reserve |                       |  Place Valid Bids    |
+──────────────────+                       +──────────────────────+
         │                                            │
         └──────────────────┬─────────────────────────┘
                            │
                            ▼
              +───────────────────────────+
              |   Online Auction Engine   |
              | ───────────────────────── |
              | • Atomic Bid Validation   |
              | • Anti-Sniping Soft-Close |
              | • Scheduled Auto-Closure  |
              | • Escrow Settlement       |
              +───────────────────────────+
```

---

### 1.1 Functional Requirements (FR)

1. **Auction Lifecycle Management**:
   - Sellers can create and schedule auction listings specifying item details, category, high-resolution media, starting bid price, minimum bid increments, optional reserve price, and duration (start/end timestamps).
   - Sellers can define an optional "Buy-It-Now" fixed price which immediately concludes the auction if satisfied before the reserve price is crossed.
2. **Real-Time Bid Placement & Validation**:
   - Authenticated bidders can place bids. A bid is valid if and only if:
     - The auction is in an `ACTIVE` or `EXTENDED` status.
     - The bid amount is strictly greater than $\text{Current Highest Bid} + \text{Minimum Increment}$.
     - The bidder is authorized, has verified payment details, and is not the seller (anti-shill rule).
   - System must support **Proxy Bidding** (automatic incremental bidding up to a user's secret maximum ceiling).
3. **Live State & Price Streaming**:
   - Active viewers and participating bidders must receive real-time price updates, outbid notifications, and remaining time countdowns with sub-50ms propagation latency without page refreshes.
4. **Anti-Sniping Dynamic Soft-Close (Overtime)**:
   - If a valid bid is placed within the final 60 seconds of an auction, the auction end time automatically extends by 2 minutes. This soft-close mechanism prevents automated sniping bots from unfairly capturing listings at the last millisecond.
5. **Deterministic Auction Closure & Settlement**:
   - The system must automatically and deterministically close auctions at the scheduled timestamp (accounting for extensions).
   - Upon closure, the winning bidder and final hammer price are locked, and the platform transitions the listing into escrow settlement and payment capture.

---

### 1.2 Non-Functional Requirements (NFR)

1. **Strong Consistency & Linearizability on Bids (CP Focus)**:
   - For any single auction, bid admission must be strictly linearizable. Two bids at the same dollar amount can never both be accepted; the earliest arrival must win, and subsequent equal or lower bids must be rejected immediately.
2. **Ultra-Low Latency**:
   - **Bid Placement Round-Trip**: $p99 < 15\text{ms}$.
   - **Real-Time Fan-Out Broadcast**: Sub-$50\text{ms}$ propagation to all connected WebSocket clients watching the hot auction.
3. **High Availability (99.99%)**:
   - High availability for browsing, viewing, and listing items. Bidding subsystem utilizes distributed memory clusters with automatic failover to prevent downtime during peak closing windows.
4. **Fault Tolerance & Absolute Durability (Zero Financial Data Loss)**:
   - Every accepted bid and financial authorization event must be durably recorded in an immutable audit ledger. If a cache node or broker crashes, no accepted bid state can be lost.
5. **Contention Resilience**:
   - The platform must absorb extreme write contention during viral/hot-item auctions (e.g., rare art, celebrity memorabilia) experiencing up to **1,000 bids/second on a single auction record** without database deadlocks or thread pool starvation.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Traffic & User Scale
- **Daily Active Users (DAU)**: $10\text{ Million}$.
- **Concurrent Active Auctions**: $1,000,000\text{ active auctions}$ running globally at any given moment.
- **New Auctions Created**: $100,000\text{ listings/day}$.
  $$\text{Auction Creation Write QPS} = \frac{100,000}{86,400\text{ s}} \approx 1.16\text{ writes/sec (Peak: } 15\text{ writes/sec)}$$

#### Bidding Throughput
- **Bids Placed Per Day**: $10\text{ Million bids/day}$.
  $$\text{Average Bid Write QPS} = \frac{10,000,000}{86,400\text{ s}} \approx 116\text{ bids/sec}$$
- **Global Peak Bid QPS**: $5,000\text{ to } 10,000\text{ bids/sec}$ across all auctions during peak evening shopping hours.
- **Hot-Spot Item Contention**: In a viral auction closing window, a single auction item can experience **500 to 1,000 concurrent bids/second**.

#### Read & Live Streaming Scale
- Each user views an average of 10 auctions per day: $100\text{ Million page/item views/day}$.
- **Average Active WebSocket Connections**: $1,000,000\text{ concurrent connected clients}$.
- **WebSocket Broadcast Fan-Out**:
  - Suppose at any time, 1,000 auctions are hot (each watched by 1,000 concurrent bidders and receiving 2 bids/sec).
  $$\text{Outbound WebSocket Egress Messages} = 1,000\text{ auctions} \times 2\text{ bids/sec} \times 1,000\text{ watchers} = 2,000,000\text{ msgs/sec}$$

#### Storage Requirements (5-Year Horizon)
1. **Auction Listing Metadata**:
   - Record size: $\approx 2\text{ KB}$ (Title, description, attributes, seller ID, timestamps, rules).
   - $100,000\text{ listings/day} \times 2\text{ KB} = 200\text{ MB/day} \to 73\text{ GB/year} \to \mathbf{365\text{ GB (5 Years)}}$.
2. **Bid History Ledger**:
   - Record size: $\approx 128\text{ bytes}$ (Bid UUID, Auction ID, Bidder ID, Amount, Timestamp, Status, Client IP).
   - $10\text{ Million bids/day} \times 128\text{ bytes} \approx 1.28\text{ GB/day} \to 467\text{ GB/year} \to \mathbf{2.33\text{ TB (5 Years)}}$.
3. **Media Storage (Images/Videos in Object Store)**:
   - 5 high-resolution images per listing $\times 500\text{ KB} = 2.5\text{ MB/auction}$.
   - $100,000 \times 2.5\text{ MB} = 250\text{ GB/day} \to \mathbf{91.25\text{ TB/year}}$. Fronted by Global CDN.

#### Memory Sizing for Hot State
- Active in-memory auction working set: $1,000,000\text{ auctions}$.
- State per active auction in Redis cache (current highest bid, bidder ID, reserve met flag, dynamic end time, version): $\approx 256\text{ bytes}$.
$$\text{Redis Working Memory} = 1,000,000 \times 256\text{ bytes} \approx 256\text{ MB}$$
- Including bid history ring buffers (last 20 bids per auction) and connection maps: $\approx \mathbf{5\text{ to } 10\text{ GB}}$ of operational RAM in Redis. This fits comfortably in a standard Redis cluster, ensuring lightning-fast in-memory processing.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

### 2.1 Domain Entities

```
+---------------------------------------------------------------------------------------------------------+
|                                           Domain Model Overview                                         |
|                                                                                                         |
|  +-------------------+        +--------------------+        +--------------------+                      |
|  |       User        | 1    N |        Item        | 1    1 |      Auction       |                      |
|  |-------------------|--------|--------------------|--------|--------------------|                      |
|  | user_id (PK)      |        | item_id (PK)       |        | auction_id (PK)    |                      |
|  | email             |        | seller_id (FK)     |        | item_id (FK)       |                      |
|  | kyc_status        |        | title              |        | starting_price     |                      |
|  | payment_vault_id  |        | media_urls[]       |        | reserve_price      |                      |
|  +-------------------+        +--------------------+        | current_highest_bid|                      |
|            │                                                | status             |                      |
|            │ 1                                              | end_time           |                      |
|            │                                                +--------------------+                      |
|            ▼ N                                                         │ 1                              |
|  +-------------------+                                                 │                                |
|  |        Bid        |                                                 ▼ N                              |
|  |-------------------|                                       +--------------------+                     |
|  | bid_id (PK)       | ◀─────────────────────────────────────|    EscrowHold      |                     |
|  | auction_id (FK)   |                                       |--------------------|                     |
|  | bidder_id (FK)    |                                       | hold_id (PK)       |                     |
|  | amount            |                                       | auction_id (FK)    |                     |
|  | status            |                                       | winner_id (FK)     |                     |
|  | placed_at         |                                       | escrow_status      |                     |
|  +-------------------+                                       +--------------------+                     |
+---------------------------------------------------------------------------------------------------------+
```

1. **User**: Represents buyers, sellers, and auctioneers with verified identity and payment capabilities.
2. **Item**: The physical or digital asset listed for sale, with condition reports, provenance, and CDN media links.
3. **Auction**: The state machine governing the sale lifecycle (`SCHEDULED`, `ACTIVE`, `EXTENDED`, `SOLD`, `UNSOLD`, `CANCELLED`).
4. **Bid**: The immutable record of an offer submitted by a bidder, recording dollar amount, timestamp, and evaluation status.
5. **ProxyBid**: User's confidential maximum ceiling configured for automated step increments.
6. **EscrowHold & Payment**: Financial pre-authorization and final settlement records tracking fund transfers, collateral holds, and payouts.

---

### 2.2 Relational Data Schema (PostgreSQL)

Relational storage guarantees transactional ACID guarantees for auction creation, final financial settlements, and audit logs.

#### `users` Table
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(150) NOT NULL,
    kyc_verified BOOLEAN DEFAULT FALSE,
    payment_vault_token VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

#### `items` Table
```sql
CREATE TABLE items (
    item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id UUID NOT NULL REFERENCES users(user_id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    category_id INT NOT NULL,
    item_condition VARCHAR(50) NOT NULL,
    media_urls JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_items_seller ON items(seller_id);
```

#### `auctions` Table
```sql
CREATE TYPE auction_status AS ENUM (
    'DRAFT', 'SCHEDULED', 'ACTIVE', 'EXTENDED', 'ENDED_SOLD', 'ENDED_UNSOLD', 'CANCELLED'
);

CREATE TABLE auctions (
    auction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id UUID UNIQUE NOT NULL REFERENCES items(item_id),
    seller_id UUID NOT NULL REFERENCES users(user_id),
    starting_price NUMERIC(14, 2) NOT NULL CHECK (starting_price > 0),
    reserve_price NUMERIC(14, 2), -- Nullable: seller can choose no reserve
    buy_now_price NUMERIC(14, 2), -- Nullable: optional instant purchase price
    current_highest_bid NUMERIC(14, 2) DEFAULT 0.00,
    current_highest_bidder_id UUID REFERENCES users(user_id),
    bid_increment NUMERIC(10, 2) NOT NULL DEFAULT 5.00,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    original_end_time TIMESTAMPTZ NOT NULL,
    extension_count INT DEFAULT 0,
    soft_close_window_seconds INT DEFAULT 60,
    soft_close_extension_seconds INT DEFAULT 120,
    status auction_status NOT NULL DEFAULT 'SCHEDULED',
    version BIGINT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_auctions_status_end_time ON auctions(status, end_time);
CREATE INDEX idx_auctions_seller ON auctions(seller_id);
```

#### `bids` Table (Immutable Append-Only Audit Ledger)
```sql
CREATE TYPE bid_status AS ENUM ('ACCEPTED', 'OUTBID', 'WINNING', 'REJECTED');

CREATE TABLE bids (
    bid_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    auction_id UUID NOT NULL REFERENCES auctions(auction_id),
    bidder_id UUID NOT NULL REFERENCES users(user_id),
    amount NUMERIC(14, 2) NOT NULL,
    bid_timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status bid_status NOT NULL DEFAULT 'ACCEPTED',
    idempotency_key VARCHAR(128) UNIQUE NOT NULL,
    ip_address INET,
    user_agent TEXT
);

CREATE INDEX idx_bids_auction_timestamp ON bids(auction_id, bid_timestamp DESC);
CREATE INDEX idx_bids_bidder ON bids(bidder_id);
```

#### `escrow_settlements` Table
```sql
CREATE TYPE escrow_status AS ENUM ('PENDING', 'FUNDS_HELD', 'CAPTURED', 'RELEASED_TO_SELLER', 'REFUNDED');

CREATE TABLE escrow_settlements (
    settlement_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    auction_id UUID UNIQUE NOT NULL REFERENCES auctions(auction_id),
    buyer_id UUID NOT NULL REFERENCES users(user_id),
    seller_id UUID NOT NULL REFERENCES users(user_id),
    final_amount NUMERIC(14, 2) NOT NULL,
    platform_fee NUMERIC(14, 2) NOT NULL,
    status escrow_status NOT NULL DEFAULT 'PENDING',
    payment_gateway_ref VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    settled_at TIMESTAMPTZ
);
```

---

## 3. Step 3: API & Interface Design

### 3.1 Public REST Contracts

#### 1. Create Auction Listing
- **Endpoint**: `POST /api/v1/auctions`
- **Headers**: `Authorization: Bearer <JWT>`, `Content-Type: application/json`
- **Request Payload**:
  ```json
  {
    "item": {
      "title": "1967 Shelby GT500 Fastback Original",
      "description": "Mint condition, numbers matching 428 Cobra Jet V8 engine.",
      "category_id": 104,
      "condition": "EXCELLENT",
      "media_urls": [
        "https://cdn.auctionplatform.com/media/shelby1.jpg",
        "https://cdn.auctionplatform.com/media/shelby2.jpg"
      ]
    },
    "starting_price": 75000.00,
    "reserve_price": 95000.00,
    "buy_now_price": 140000.00,
    "bid_increment": 1000.00,
    "start_time": "2026-10-20T14:00:00Z",
    "end_time": "2026-10-27T18:00:00Z",
    "soft_close_window_seconds": 60,
    "soft_close_extension_seconds": 120
  }
  ```
- **Response**: `201 Created`
  ```json
  {
    "auction_id": "auc_89a3f120-ef92-4299",
    "status": "SCHEDULED",
    "start_time": "2026-10-20T14:00:00Z",
    "end_time": "2026-10-27T18:00:00Z",
    "starting_price": 75000.00,
    "created_at": "2026-10-18T19:30:00Z"
  }
  ```

---

#### 2. Place Bid (High-Contention Hot Path)
- **Endpoint**: `POST /api/v1/auctions/{auctionId}/bids`
- **Headers**: 
  - `Authorization: Bearer <JWT>`
  - `X-Idempotency-Key: idemp_key_003_bid`
  - `Content-Type: application/json`
- **Request Payload**:
  ```json
  {
    "amount": 105000.00,
    "max_proxy_amount": 115000.00
  }
  ```
- **Response (Success - Winning Bid)**: `201 Created`
  ```json
  {
    "bid_id": "bid_4091ab1a-9821-4fce",
    "auction_id": "auc_89a3f120-ef92-4299",
    "amount": 105000.00,
    "status": "ACCEPTED",
    "is_current_highest": true,
    "current_highest_bid": 105000.00,
    "end_time": "2026-10-27T18:02:00Z",
    "extended": true,
    "timestamp": "2026-10-27T17:59:22.450Z"
  }
  ```
- **Response (Outbid / Race Condition Failure)**: `409 Conflict`
  ```json
  {
    "error_code": "OUTBID_BY_COMPETING_BIDDER",
    "message": "Your bid of $105,000.00 was rejected. The current highest bid is already $106,000.00.",
    "current_highest_bid": 106000.00,
    "minimum_next_bid": 107000.00
  }
  ```
- **Response (Auction Closed)**: `410 Gone`
  ```json
  {
    "error_code": "AUCTION_ALREADY_CLOSED",
    "message": "This auction closed at 2026-10-27T18:02:00Z. No further bids accepted."
  }
  ```

---

#### 3. Get Auction State & Bid History
- **Endpoint**: `GET /api/v1/auctions/{auctionId}`
- **Response**: `200 OK`
  ```json
  {
    "auction_id": "auc_89a3f120-ef92-4299",
    "status": "ACTIVE",
    "current_highest_bid": 105000.00,
    "current_highest_bidder_masked": "b***9",
    "bid_count": 47,
    "start_time": "2026-10-20T14:00:00Z",
    "end_time": "2026-10-27T18:02:00Z",
    "time_remaining_ms": 37550,
    "min_next_bid": 106000.00,
    "item": {
      "title": "1967 Shelby GT500 Fastback Original",
      "media_urls": ["https://cdn.auctionplatform.com/media/shelby1.jpg"]
    }
  }
  ```

---

### 3.2 Real-Time WebSocket Streaming Interface

To prevent polling storms on high-traffic auctions, clients establish a persistent full-duplex WebSocket connection to the **WebSocket Gateway Cluster**.

#### Connection Handshake
```text
GET /v1/live/auctions HTTP/1.1
Host: ws.auctionplatform.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Authorization: Bearer <optional_jwt_for_personal_outbid_alerts>
```

#### Client Subscribe Message
```json
{
  "action": "SUBSCRIBE_AUCTION",
  "auction_id": "auc_89a3f120-ef92-4299"
}
```

#### Server Broadcast Message (Price Update & Extension Event)
```json
{
  "event": "AUCTION_PRICE_UPDATED",
  "auction_id": "auc_89a3f120-ef92-4299",
  "current_highest_bid": 105000.00,
  "bid_count": 48,
  "highest_bidder_masked": "u***4",
  "end_time": "2026-10-27T18:02:00Z",
  "extended": true,
  "extension_reason": "SOFT_CLOSE_ANTI_SNIPE",
  "timestamp": 1793124000450
}
```

#### Server Direct Notification (Personal Outbid Alert)
```json
{
  "event": "PERSONAL_OUTBID",
  "auction_id": "auc_89a3f120-ef92-4299",
  "your_last_bid": 104000.00,
  "new_highest_bid": 105000.00,
  "recommended_bid": 106000.00
}
```

---

## 4. Step 4: High-Level Architecture

The architecture decouples the **write-intensive, latency-critical bidding engine** from the **read-heavy auction browsing and post-auction settlement workflows**.

```
                                      +───────────────────────────────────+
                                      |      Clients (Web / iOS / Android)|
                                      +───────────────────────────────────+
                                            │                     ▲
                      HTTPS REST API (Bids) │                     │ Real-Time WebSocket Frames
                                            ▼                     │ (<50ms Broadcast)
                            +─────────────────────────+           │
                            |  Cloudflare Edge / WAF  |           │
                            +─────────────────────────+           │
                                        │                         │
                                        ▼                         │
                            +─────────────────────────+           │
                            | API Gateway (Envoy / K8s|           │
                            +─────────────────────────+           │
                                   │            │                 │
                POST /bids         │            │ GET /auctions   │
         ┌─────────────────────────┘            └────────┐        │
         ▼                                               ▼        │
+──────────────────────────+                  +──────────────────────────+
|      Bidding Service     |                  |      Auction Service     |
|   (Go / High Performance)|                  |       (Metadata/Search)  |
+──────────────────────────+                  +──────────────────────────+
         │                                               │
         │ Atomic Lua CAS                                │ Read Replica Lookups
         ▼                                               ▼
+──────────────────────────+                  +──────────────────────────+
|  In-Memory Hot State     |                  | Read Cache (Elastic /    |
| (Redis Cluster Primary)  |                  | Redis Cluster)           |
+──────────────────────────+                  +──────────────────────────+
    │               │                                    ▲
    │ Pub/Sub       │ Reliable Append                    │ Search Index Sync
    ▼               ▼                                    │
+──────────────+  +──────────────────────────────────+   │
| Redis Pub/Sub|  | Apache Kafka Event Bus           |───┘
| Engine       |  |  • topic: bids-accepted          |
+──────────────+  |  • topic: auction-extended       |
       │          |  • topic: auction-closed         |
       ▼          +──────────────────────────────────+
+──────────────+               │                 │
|  WebSocket   |               ▼                 ▼
|  Gateway     |       +───────────────+ +─────────────────────────+
|  Cluster     |       | Async DB      | | Scheduled Closer Engine |
+──────────────+       | Write-Behind  | | (Redis ZSET / Poller)   |
                       | Worker        | +─────────────────────────+
                       +───────────────+                 │
                               │                         ▼
                               ▼               +───────────────────+
                       +───────────────+       | Payment & Escrow  |
                       |  PostgreSQL   |       | Settlement Service|
                       | ACID Storage  |       +───────────────────+
                       +───────────────+                 │
                                                         ▼
                                               +───────────────────+
                                               | Stripe / Bank ACH |
                                               +───────────────────+
```

### 4.1 Component Breakdown & Separation of Concerns

1. **Edge Tier & API Gateway**:
   - Terminates TLS, validates JWT tokens, evaluates IP-based rate limiting, and routes requests.
   - Forwards WebSocket handshake requests directly to the stateful **WebSocket Gateway Cluster**.
2. **Bidding Service**:
   - A stateless, lightweight service optimized for extreme throughput (written in Go or Rust).
   - Validates incoming bid payloads, verifies pre-authorization hold limits, and invokes **Atomic Lua Scripts on Redis**.
   - Emits accepted bid events to Kafka asynchronously for durable persistence.
3. **Auction Service**:
   - Handles standard CRUD operations for listings, categories, and sellers.
   - Manages rich media uploads to S3 and syncs search indexes with OpenSearch/Elasticsearch for catalog browsing.
4. **WebSocket Gateway Cluster**:
   - Maintains $1\text{ Million}+$ persistent TCP connections across distributed pods.
   - Subscribes to Redis Pub/Sub channels corresponding to active auctions. When a bid succeeds, the gateway fans out the event to all connected clients viewing that specific listing.
5. **Scheduled Closer & Anti-Sniping Engine**:
   - Monitors auction expiry using a **Redis Sorted Set (ZSET)** delay queue.
   - When an auction expires without further extensions, workers transition the auction state to `CLOSED` and emit an `auction-closed` Kafka event.
6. **Payment & Escrow Settlement Service**:
   - Consumes `auction-closed` events. Initiates post-auction payment captures against the winning bidder's pre-authorized card or escrow account.

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Extreme Contention & Race Conditions (Why Naive DB Locks Fail vs. Atomic Redis Lua)

#### The Problem: The Naive Database Locking Catastrophe
In a viral auction ending in 10 seconds, 500 bidders submit bids simultaneously. A standard relational approach uses row-level pessimistic locking:

```sql
-- Naive Database Transaction
BEGIN TRANSACTION;
SELECT current_highest_bid, status, end_time 
FROM auctions 
WHERE auction_id = 'auc_123' 
FOR UPDATE; -- Acquires exclusive row lock

-- Application checks: if new_bid > current_highest_bid and NOW() < end_time:
INSERT INTO bids (auction_id, bidder_id, amount, status) 
VALUES ('auc_123', 'usr_888', 10500.00, 'ACCEPTED');

UPDATE auctions 
SET current_highest_bid = 10500.00, 
    current_highest_bidder_id = 'usr_888',
    version = version + 1
WHERE auction_id = 'auc_123';

COMMIT;
```

#### Why This Fails Catastrophically Under Load:
1. **Thread Pool Exhaustion**: 500 concurrent HTTP threads line up waiting for the single exclusive row lock on `auc_123`. The database connection pool is depleted in milliseconds.
2. **Deadlocks & Lock Contention Spikes**: If other queries concurrently read or lock related tables (e.g., `bids`, `users`), lock escalation and deadlocks occur.
3. **High Latency**: Disk I/O, Write-Ahead Logging (WAL) flushes, and lock queuing push request latency from $5\text{ms}$ to $>2,000\text{ms}$. By the time thread #200 gets the lock, the auction has already closed, or the price is completely stale.

---

#### The Production Solution: In-Memory Atomic Check-And-Set (CAS) via Redis Lua
Redis executes commands on a single-threaded event loop per shard. A Lua script executed in Redis is **strictly atomic**: no other command or Lua script can execute concurrently on that shard.

```
Incoming Bid Requests
(500 bids/sec)
     │
     ▼
+─────────────────────────────────────────────────────────────────+
|               Redis Atomic Execution Environment                |
|                                                                 |
|   1. EVAL SHA <bid_script>                                      |
|   2. Check Status == 'ACTIVE'                                   |
|   3. Check Current Time < end_time                              |
|   4. Verify new_bid >= current_max + min_increment              |
|   5. Soft-Close Check: if (end_time - now) < 60s                |
|         ──▶ end_time = end_time + 120s                          |
|   6. Update Redis Hash (max_bid, bidder_id, end_time)           |
|   7. PUBLISH auction:auc_123:stream (price_update_payload)      |
+─────────────────────────────────────────────────────────────────+
     │
     ▼
Return JSON Success / Rejection Code (< 2ms)
```

#### Production Lua Script for Bid Placement (`place_bid.lua`)

```lua
-- KEYS[1]: auction:{auction_id}:state (Redis Hash)
-- KEYS[2]: auction:closers (Redis ZSET for Scheduled Expiry)
-- ARGV[1]: bidder_id
-- ARGV[2]: bid_amount (float)
-- ARGV[3]: min_increment (float)
-- ARGV[4]: current_timestamp_ms (integer epoch)
-- ARGV[5]: soft_close_window_ms (e.g. 60000)
-- ARGV[6]: soft_close_extension_ms (e.g. 120000)

local auction_state = redis.call('HMGET', KEYS[1], 'status', 'current_highest_bid', 'end_time_ms', 'highest_bidder_id', 'extension_count')
local status = auction_state[1]
local current_highest_bid = tonumber(auction_state[2]) or 0
local end_time_ms = tonumber(auction_state[3])
local current_bidder = auction_state[4]
local extension_count = tonumber(auction_state[5]) or 0

local bid_amount = tonumber(ARGV[2])
local min_increment = tonumber(ARGV[3])
local now_ms = tonumber(ARGV[4])
local window_ms = tonumber(ARGV[5])
local extend_ms = tonumber(ARGV[6])

-- 1. Validate Auction is Active
if status ~= 'ACTIVE' and status ~= 'EXTENDED' then
    return cjson.encode({ success = false, error = "AUCTION_NOT_ACTIVE", status = status })
end

-- 2. Validate Auction Expiry
if now_ms >= end_time_ms then
    return cjson.encode({ success = false, error = "AUCTION_EXPIRED", end_time_ms = end_time_ms })
end

-- 3. Prevent Self-Bidding (Shill Prevention)
if current_bidder == ARGV[1] then
    return cjson.encode({ success = false, error = "ALREADY_HIGHEST_BIDDER" })
end

-- 4. Validate Minimum Increment
local required_min_bid = current_highest_bid + min_increment
if bid_amount < required_min_bid then
    return cjson.encode({ 
        success = false, 
        error = "BID_TOO_LOW", 
        current_highest_bid = current_highest_bid, 
        required_min_bid = required_min_bid 
    })
end

-- 5. Anti-Sniping Dynamic Soft-Close Logic
local extended = false
local remaining_ms = end_time_ms - now_ms

if remaining_ms <= window_ms then
    end_time_ms = end_time_ms + extend_ms
    extension_count = extension_count + 1
    extended = true
    -- Update Expiry Delay Queue (ZSET)
    redis.call('ZADD', KEYS[2], end_time_ms, KEYS[1])
end

-- 6. Atomically Update Auction State Hash
redis.call('HMSET', KEYS[1], 
    'current_highest_bid', tostring(bid_amount),
    'highest_bidder_id', ARGV[1],
    'end_time_ms', tostring(end_time_ms),
    'extension_count', tostring(extension_count),
    'status', (extended and 'EXTENDED' or status)
)

-- 7. Publish Event to Redis Pub/Sub for Real-Time Fan-Out
local event_payload = cjson.encode({
    event = "NEW_BID",
    bidder_id = ARGV[1],
    amount = bid_amount,
    end_time_ms = end_time_ms,
    extended = extended,
    timestamp_ms = now_ms
})
redis.call('PUBLISH', 'channel:' .. KEYS[1], event_payload)

return cjson.encode({
    success = true,
    bid_amount = bid_amount,
    end_time_ms = end_time_ms,
    extended = extended,
    extension_count = extension_count
})
```

#### Asynchronous Write-Behind via Kafka for Durability
Once Redis returns `success = true`, the Bidding Service asynchronously dispatches a message to the Kafka partition topic `bids-accepted` (partitioned by `auction_id` to preserve strict per-auction order):

```
Bidding Service ──(Async Producer Ack=all)──▶ Kafka [topic: bids-accepted]
                                                      │
                                                      ▼
                                       Postgres Write-Behind Consumer
                                       (Micro-Batched: 1,000 bids / 50ms)
                                                      │
                                                      ▼
                                       INSERT INTO bids ...
                                       UPDATE auctions ...
```
- **Durability Guarantee**: Kafka with `acks=all` and replication factor $3$ guarantees zero bid loss even if the Redis primary crashes immediately after the Lua execution.

---

### 5.2 Deep Dive 2: Real-Time Bid Broadcast (<50ms Latency)

Broadcasting updates to $1\text{ Million}$ concurrent connections requires an efficient publish-subscribe layer and connection gateway.

```
+──────────────────────────+
| Redis Pub/Sub Master     |
| channel:auction:auc_123  |
+──────────────────────────+
             │
             ├─── Subscribed ───▶ [ Gateway Pod A ] ──▶ WebSockets to Bidders 1..500
             ├─── Subscribed ───▶ [ Gateway Pod B ] ──▶ WebSockets to Bidders 501..1200
             └─── Subscribed ───▶ [ Gateway Pod C ] ──▶ WebSockets to Bidders 1201..2000
```

1. **Epoll / Netty Non-Blocking Gateway**:
   - Each WebSocket Gateway pod (written in Go with `epoll` or Java with `Netty`) maintains up to **50,000 idle TCP connections**.
   - Memory footprint per idle WebSocket: $\approx 4\text{ KB} \to 200\text{ MB}$ per 50,000 connections.
2. **Channel Subscription Optimization**:
   - Instead of every gateway pod subscribing to all $1\text{ Million}$ auction channels in Redis, gateway pods dynamically register listeners only for auctions that their currently connected clients have subscribed to.
   - When a client sends `{ action: "SUBSCRIBE", auction_id: "auc_123" }`, the local gateway pod adds the client socket descriptor to an in-memory `Set<ClientSocket>` mapped to `auc_123`. If this is the first client on that pod watching `auc_123`, the pod subscribes to `channel:auction:auc_123` on Redis.
3. **Payload Delta Compression**:
   - Full auction metadata is never broadcast on a bid update. Only minimal delta frames ($\approx 90\text{ bytes}$) are sent over the wire:
     ```json
     {"e":"BID","a":105000,"b":"u***9","t":1793124000,"x":1}
     ```
   - This keeps bandwidth utilization under $5\text{ MB/sec}$ per pod even under extreme broadcast fan-out.

---

### 5.3 Deep Dive 3: Anti-Sniping Engine (Dynamic Soft-Close Algorithm)

#### The Sniping Exploit
In traditional auctions without anti-sniping (such as early eBay), automated sniping bots wait until $200\text{ms}$ before auction end to place a bid. This prevents other human participants from evaluating the new price or countering, resulting in suboptimal hammer prices for sellers and severe frustration for buyers.

#### Dynamic Soft-Close State Transition Machine

```
                   [ Scheduled Auction ]
                             │
                             ▼
                    [ ACTIVE Auction ]
                             │
                             ├─────────────────────────────────────────────────┐
                             │                                                 │
                             ▼                                                 ▼
               Bid placed outside final 60s                       Bid placed in final 60s
                             │                                                 │
                             ▼                                                 ▼
                  Normal Bid Acceptance                            [ EXTENDED Auction ]
                   (End time unchanged)                             end_time += 120 seconds
                             │                                                 │
                             │                                                 │
                             └─────────────────┬───────────────────────────────┘
                                               │
                                               ▼
                              Timer reaches end_time (No new bids)
                                               │
                                               ▼
                                      [ CLOSED / SETTLING ]
```

#### Rule Enforcement & Abuse Prevention:
1. **The Overtime Window**: If a bid arrives where $\text{end\_time} - \text{now} \le 60\text{ seconds}$, extend the auction end time by $120\text{ seconds}$.
2. **Anti-Griefing Hard Ceiling**: Malicious bidders could theoretically keep extending an auction indefinitely to lock up seller inventory. To prevent this, the engine enforces a **Maximum Extension Limit** (e.g., maximum 20 extensions, or a hard stop at $\text{Original End Time} + 2\text{ hours}$). Once the hard cap is reached, no further extensions are granted, and standard final-second bid rules apply.

---

### 5.4 Deep Dive 4: Auction Lifecycle & Scheduled Closure (Redis ZSET Delay Queue vs. Quartz)

Auctions must close with millisecond precision without executing expensive database full-table polling queries like `SELECT * FROM auctions WHERE end_time <= NOW()`.

#### Redis ZSET Distributed Delay Queue
All active auctions are maintained in a Redis Sorted Set `auction:closers`:
- **Key**: `auction:closers`
- **Score**: `end_time_epoch_ms` (e.g., `1793124120000`)
- **Member**: `auction_id` (e.g., `"auc_89a3f120"`)

```
Redis ZSET (auction:closers)
Score: 1793124000000 ──▶ Member: auc_001
Score: 1793124005000 ──▶ Member: auc_002
Score: 1793124012000 ──▶ Member: auc_003
```

#### Distributed Closer Worker Algorithm
Stateless Closer Worker pods run a tight non-blocking polling loop:

```python
# Distributed Closer Worker Routine
import time
import redis
import json

r = redis.Redis(host='redis-primary', port=6379)

def poll_expired_auctions():
    while True:
        now_ms = int(time.time() * 1000)
        
        # Atomically pop auctions whose end_time <= current time
        # Using Lua script to prevent multiple workers from claiming the same expired auction
        lua_pop_script = """
        local expired_auctions = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'LIMIT', 0, 50)
        if #expired_auctions > 0 then
            redis.call('ZREM', KEYS[1], unpack(expired_auctions))
        end
        return expired_auctions
        """
        
        expired_ids = r.eval(lua_pop_script, 1, 'auction:closers', now_ms)
        
        for auction_id in expired_ids:
            process_auction_closure(auction_id.decode('utf-8'))
            
        time.sleep(0.1) # 100ms polling tick

def process_auction_closure(auction_id):
    # Transition Redis Hash state to CLOSED
    state_key = f"auction:{auction_id}:state"
    r.hset(state_key, "status", "CLOSED")
    
    # Read final winning details
    auction_data = r.hgetall(state_key)
    winner_id = auction_data.get(b'highest_bidder_id')
    final_amount = auction_data.get(b'current_highest_bid')
    
    # Emit closure event to Kafka
    kafka_producer.send('auction-closed', {
        'auction_id': auction_id,
        'winner_id': winner_id.decode('utf-8') if winner_id else None,
        'final_amount': float(final_amount) if final_amount else 0.0,
        'closed_at': int(time.time() * 1000)
    })
```

---

### 5.5 Deep Dive 5: Asynchronous Payment & Escrow Settlement

Post-auction settlement presents significant financial risk: what if the winning bidder's credit card declines, or they abandon the purchase?

```
                                  +────────────────────────────+
                                  | Kafka Event: auction-closed|
                                  +────────────────────────────+
                                                │
                                                ▼
                                  +────────────────────────────+
                                  | Escrow Settlement Worker   |
                                  +────────────────────────────+
                                                │
                     ┌──────────────────────────┴──────────────────────────┐
                     ▼                                                     ▼
           [ Pre-Authorized Hold ]                               [ Charge On-Demand ]
    (High-Value Sotheby's Style)                                (eBay Consumer Style)
  Pre-auth captured instantly ($10k deposit)               Charge winner's default card on file
                     │                                                     │
          ┌──────────┴──────────┐                               ┌──────────┴──────────┐
          ▼                     ▼                               ▼                     ▼
     [ Success ]            [ Fails ]                      [ Success ]            [ Fails ]
  Hold moved to       Release asset to               Funds locked in        Notify winner;
  escrow; notify      Runner-Up bidder               Escrow until delivery  24h Grace Period;
  seller to ship                                     confirmation           fallback to 2nd bidder
```

#### Pre-Authorization vs. Post-Auction Capture Models
1. **Consumer Auctions (eBay Style)**:
   - For standard consumer goods ($<\$1,000$), placing a hold on every bid is impractical because a user might bid on 10 laptops simultaneously with only sufficient balance for one.
   - **Mechanism**: The winning bidder is granted a **24-hour checkout grace period**. If payment is not completed within 24 hours, the system marks the transaction as defaulted, penalizes the user's account rating, and offers a **Second Chance Offer** to the second-highest bidder at their last maximum bid.
2. **High-Value Luxury & Industrial Auctions (Sotheby's / GovDeals Style)**:
   - For real estate, fine art, or heavy machinery ($>\$50,000$), bidders must register with a **Verified Pre-Authorization Hold** (e.g., \$5,000 or 10% collateral hold) or verified bank wire guarantee before submitting any bid.
   - The hold is locked via Stripe/Adyen at registration. Upon winning, the hold is captured towards the purchase price, and the remainder is settled via Escrow Wire. Non-winning bidders have their holds immediately released upon auction conclusion.

---

### 5.6 Deep Dive 6: Fraud Prevention, Shill Bidding & Bid Retractions

1. **Shill Bidding Detection**:
   - Shill bidding occurs when a seller uses dummy accounts, family members, or automated collusive bots to artificially drive up the price of their listing without intending to buy.
   - **Detection Engine**:
     - Asynchronous graph analysis workers cross-reference bidder IP subnets, hardware fingerprints, payment card billing addresses, and device IDs against the seller.
     - An anomaly scoring pipeline flags bidders who consistently bid exclusively on a specific seller's listings and drop out immediately after pushing the price past the reserve.
2. **Bid Retraction Guardrails**:
   - Unrestricted bid retractions destroy market integrity.
   - Retractions are disallowed within the final **12 hours** of an auction. Prior to that, retractions require administrative approval or must demonstrate a clear typographical error (e.g., entering \$100,000 instead of \$1,000) within 15 minutes of bid placement.

---

## 6. Step 6: Comparative Trade-Off Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Bid Contention Engine** | **In-Memory Atomic Redis Lua Scripts** | Relational Row Locking (`SELECT ... FOR UPDATE`) | Row locks create massive thread-pool exhaustion and deadlocks under 500+ bids/sec. Redis Lua provides guaranteed single-threaded atomicity with $<2\text{ms}$ execution latency. |
| **Bid Data Durability** | **Write-Behind via Apache Kafka (acks=all)** | Synchronous Dual-Write (Redis + PostgreSQL) | Synchronous dual-writing exposes the hot path to PostgreSQL disk I/O and latency spikes. Write-behind via partitioned Kafka guarantees zero bid loss and $p99 < 15\text{ms}$ response times. |
| **Real-Time Client Streaming** | **WebSocket Cluster + Redis Pub/Sub** | Long Polling / Server-Sent Events (SSE) | Long polling creates an unbearable HTTP request flood on viral auctions. WebSockets provide full-duplex, low-overhead persistent framing with sub-50ms update latency. |
| **Auction Closure Scheduling** | **Redis ZSET Delay Queue (`ZRANGEBYSCORE`)** | Relational Database Cron Job (`SELECT WHERE end_time <= NOW()`) | Database cron jobs require heavy index scans every second, failing under millions of listings. Redis ZSET provides $O(\log N)$ inserts and instant $O(M)$ range queries for expired items. |
| **Anti-Sniping Enforcement** | **Dynamic Soft-Close Overtime (+2m in last 60s)** | Strict Hard-Close (Fixed End Timestamp) | Hard-closes incentivize bot snipers and deprive sellers of fair market value. Soft-closes simulate authentic physical auction rooms ("Going once, going twice..."), maximizing revenue. |
| **Post-Auction Payment** | **Tiered Strategy: Pre-Auth Holds (Luxury) vs. Grace Period (Consumer)** | Mandatory Pre-Authorization Hold for all bids | Placing authorization holds on every consumer bid locks up user credit limits across multiple competing auctions, causing severe checkout abandonment. |

---

## 7. Operational Resilience, Disaster Recovery & Summary

```
                       Production High Availability Topology
                       
     [ Active Region US-East ]                       [ Passive Region US-West ]
  ┌──────────────────────────────┐                ┌──────────────────────────────┐
  | • Envoy API Gateway          |                | • Warm Standby Gateway       |
  | • Bidding Service Pods       |                | • Standby Bidding Pods       |
  | • Redis Cluster (Primary)    |──Async Repl───▶| • Redis Cluster (Replica)    |
  | • Kafka Cluster (Active)     |──MirrorMaker──▶| • Kafka Cluster (Replica)    |
  | • PostgreSQL (Primary DB)    |──WAL Stream───▶| • PostgreSQL (Read Replica)  |
  └──────────────────────────────┘                └──────────────────────────────┘
```

1. **Redis Shard Partitioning**:
   - Redis keys are partitioned by `auction:{auction_id}` using hash tags (e.g., `{auc_123}:state` and `{auc_123}:bids`) to ensure all Lua operations for a single auction execute on the same Redis shard.
2. **Disaster Recovery**:
   - Cross-region replication via Kafka MirrorMaker and PostgreSQL streaming replication provides an **RPO (Recovery Point Objective) $\approx 0$** and **RTO (Recovery Time Objective) $< 60\text{ seconds}$**.
3. **Graceful Degradation Under Extreme Load**:
   - If WebSocket fan-out exceeds network limits, the gateway degrades to broadcast throttling (aggregating bids into 250ms batch tick updates), preserving server stability while keeping the user experience responsive.
