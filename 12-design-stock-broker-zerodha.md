# System Design 12: Design Stock Broker Trading Platform (Zerodha / Upstox / Groww)

> **Source**: [Interview with Bunny - Episode 12](https://www.youtube.com/watch?v=W5FJnSywmLE)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/12](https://www.interviewwithbunny.com/systemdesign/12)  
> **Topic**: Architecting an Ultra-Low-Latency Real-Time Stock Brokerage Platform (Zerodha Kite, Groww, Upstox)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Clarifying Questions & Core Scope
- **Stock Broker vs. Stock Exchange**: We are explicitly designing a **retail/institutional stock broker** (like Zerodha Kite, Upstox, Groww), **not** the central matching engine of a national exchange (NSE/BSE/NASDAQ). The exchange is treated as an authoritative external service providing market data feeds and accepting order submissions.
- **Market Coverage**: Domestic market focus (e.g., India's NSE and BSE covering ~5,000 to 8,000 active equity and F&O symbols).
- **Latency Class**: Sub-second execution pipeline ($< 100\text{ ms}$) and real-time tick-to-screen broadcast ($< 50\text{ ms}$).
- **Compliance & Margins**: Pre-trade risk management (RMS), fund reservations, KYC verification, and regulatory audit trail generation.

---

### 1.2 Functional Requirements (FR)
1. **User Onboarding & KYC**: Users can register, submit KYC documentation (Aadhaar, PAN, signature) for automated verification via third-party providers (e.g., UIDAI/DigiLocker), and receive trading permissions.
2. **Real-Time Market Data Streaming**: Users can view sub-second live streaming Last Traded Price (LTP), bid/ask market depth, and day change metrics for subscribed stock symbols.
3. **Historical Charting**: Users can fetch historical candle charts (1-minute, 5-minute, daily, weekly OHLCV bars) across custom date ranges.
4. **Order Placement & Execution**: Users can place Market, Limit, and Stop-Loss buy/sell orders, cancel pending orders, and track order state transitions (`PENDING`, `REJECTED`, `OPEN`, `PARTIALLY_FILLED`, `COMPLETED`, `CANCELLED`).
5. **Watchlist Management**: Users can create multiple custom watchlists (e.g., "Tech Titans", "Banking"), add/remove symbols, and receive live streaming ticks for all symbols in the active watchlist.
6. **Portfolio & Real-Time P&L**: Users can view their settled equity holdings and open intraday positions with dynamically fluctuating real-time Profit and Loss (P&L) calculations.
7. **Fund & Wallet Management**: Users can deposit/withdraw money via payment gateways/bank rails and have their available trading margin recalculated atomically before order routing.

---

### 1.3 Non-Functional Requirements (NFR)
1. **Strict CAP Theorem Posture (CP - Consistency & Partition Tolerance)**:
   - For order routing, margin reservation, trade execution, and wallet ledger balances, the system **must prioritize strict consistency (CP)**. Financial loss or phantom fills due to eventual consistency are unacceptable.
   - For watchlist viewing and historical chart exploration, high availability (AP) with read replicas is acceptable.
2. **Ultra-Low Latency Budgets**:
   - Order validation and exchange routing: $< 100\text{ ms}$.
   - Market tick ingestion from exchange to user device: $< 50\text{ ms}$.
   - Real-time P&L re-calculation: $< 200\text{ ms}$.
3. **High-Frequency Ingestion & Back-Pressure Resilience**:
   - The platform must absorb market tick bursts (market open at 09:15 IST often witnesses $10\times$ baseline volume) without dropping packets or crashing frontend connections.
4. **Geographic Co-location Advantage**:
   - Unlike global social networks, brokerage operations are geographically bounded within national jurisdictions. Backend clusters are co-located in proximate data centers (e.g., AWS Mumbai `ap-south-1` co-located or cross-connected via dedicated leased lines to the NSE/BSE colocation racks in BKC/Bandra).

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Active Scale & Daily Volume
- **Registered Users**: $15\text{ Million}$.
- **Daily Active Users (DAU)**: $3\text{ Million}$ concurrent during market hours (09:15 – 15:30 IST).
- **Active Stock Symbols**: $8,000\text{ listed instruments}$ (Equities, Futures, Options).
- **Daily Orders Submitted**: $10\text{ Million orders/day}$.
  $$\text{Average Order QPS} = \frac{10,000,000}{6.25 \times 3,600\text{ s}} \approx 444\text{ orders/sec}$$
  $$\text{Peak Market-Open Order QPS} = 444 \times 8 \approx 3,550\text{ orders/sec}$$

#### Market Tick Feed Ingestion Volume
- National exchanges broadcast UDP multicast / TCP binary ticks for active symbols.
- Assume $3,000$ active stocks generate an average of $5\text{ price updates/second}$:
  $$\text{Exchange Ingestion Throughput} = 3,000 \times 5 = 15,000\text{ ticks/sec}$$
  $$\text{Peak Ingestion Throughput} = 15,000 \times 4 \approx 60,000\text{ ticks/sec}$$
- At $60,000\text{ ticks/sec}$, relational databases cannot sustain raw write IOPS. Market data must flow through an **in-memory pub/sub layer (Redis)** and write downsampled candles to a **Time-Series Database (InfluxDB / TimescaleDB)**.

#### WebSocket Egress Bandwidth
- $1\text{ Million}$ concurrent connected users viewing an average of $10\text{ stocks}$ in their active watchlist.
- If downsampled ticks are pushed at $1\text{ update/sec}$ per active symbol:
  - Compressed binary/JSON tick payload $\approx 64\text{ bytes}$.
  $$\text{Egress Bandwidth} = 1,000,000 \text{ connections} \times 64\text{ bytes} \times 2 \text{ (symbols updating/sec)} \approx 128\text{ MB/sec} \approx 1.024\text{ Gbps}$$

#### Storage Projections (5-Year Window)
- **Time-Series Market Data (InfluxDB)**:
  - 1-minute OHLCV candles for 8,000 symbols:
    $$8,000 \times 375\text{ mins/day} = 3,000,000\text{ candles/day} \times 40\text{ bytes} \approx 120\text{ MB/day} \approx 30\text{ GB/year}$$
- **Order & Trade Master Ledger (PostgreSQL)**:
  - $10\text{ Million orders/day} \times 1\text{ KB/row} \approx 10\text{ GB/day} \approx 2.5\text{ TB/year} \to 12.5\text{ TB in 5 years}$.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

```
+-----------------------------------------------------------------------------------+
|                                  CORE DOMAIN ENTITIES                             |
+-----------------------------------------------------------------------------------+

   +-----------------------+              1 : 1             +-----------------------+
   |         User          |------------------------------->|       KYCRecord       |
   +-----------------------+                                +-----------------------+
   | user_id (PK, UUID)    |                                | kyc_id (PK)           |
   | email, phone_hash     |                                | user_id (FK)          |
   | password_hash         |                                | pan_encrypted         |
   | is_kyc_verified (BOOL)|                                | aadhaar_vault_ref     |
   | created_at            |                                | verification_status   |
   +-----------------------+                                +-----------------------+
         | 1             | 1
         |               | 1 : 1
    1 : N|               v
         |         +-----------------------+
         |         |     WalletAccount     |
         |         +-----------------------+
         |         | wallet_id (PK)        |
         |         | user_id (FK)          |
         |         | ledger_balance        |
         |         | available_margin      |
         |         | currency, updated_at  |
         |         +-----------------------+
         v
   +-----------------------+              1 : N             +-----------------------+
   |       Watchlist       |------------------------------->|    WatchlistSymbol    |
   +-----------------------+                                +-----------------------+
   | watchlist_id (PK)     |                                | id (PK)               |
   | user_id (FK)          |                                | watchlist_id (FK)     |
   | name (e.g. "Banking") |                                | symbol (e.g. INFY)    |
   | display_order         |                                | exchange (NSE/BSE)    |
   +-----------------------+                                +-----------------------+

   +-----------------------+                                +-----------------------+
   |         Order         |                                |         Trade         |
   +-----------------------+                                +-----------------------+
   | order_id (PK, UUID)   |              1 : N             | trade_id (PK, Exch-ID)|
   | user_id (FK)          |------------------------------->| order_id (FK)         |
   | symbol, exchange      |   (One order can have multiple | user_id (FK)          |
   | side (BUY/SELL)       |    partial fills at exchange)  | execution_price       |
   | order_type (MKT/LIMIT)|                                | filled_quantity       |
   | target_price, quantity|                                | exchange_trade_time   |
   | status (ENUM)         |                                +-----------------------+
   | placed_at, updated_at |                                            |
   +-----------------------+                                            | feeds settled
                                                                        v
                                                            +-----------------------+
                                                            |    PortfolioHolding   |
                                                            +-----------------------+
                                                            | holding_id (PK)       |
                                                            | user_id (FK)          |
                                                            | symbol, isin          |
                                                            | total_quantity        |
                                                            | average_buy_price     |
                                                            +-----------------------+
```

---

## 3. Step 3: API & Interface Design

### 3.1 Authentication & KYC Endpoints

#### Submit KYC Documents
```http
POST /api/v1/kyc/verify
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "pan_number": "ABCDE1234F",
  "aadhaar_otp_session": "sess_8831920",
  "aadhaar_otp": "489201"
}
```
**Response (`200 OK`)**:
```json
{
  "kyc_status": "VERIFIED",
  "verified_at": "2026-10-18T09:15:00Z",
  "trading_allowed": true
}
```

---

### 3.2 Market Data Streaming & History

#### Historical Candle Data
```http
GET /api/v1/market/stocks/RELIANCE/history?interval=5m&from=2026-10-01T09:15:00Z&to=2026-10-18T15:30:00Z
Authorization: Bearer <jwt_token>
```
**Response (`200 OK`)**:
```json
{
  "symbol": "RELIANCE",
  "exchange": "NSE",
  "interval": "5m",
  "candles": [
    {
      "timestamp": "2026-10-18T09:15:00Z",
      "open": 2850.00,
      "high": 2862.50,
      "low": 2848.10,
      "close": 2859.00,
      "volume": 142050
    }
  ]
}
```

---

### 3.3 Order Execution REST Interface

#### Place Stock Order
```http
POST /api/v1/orders
Authorization: Bearer <jwt_token>
X-Idempotency-Key: ord_idem_90918239012
Content-Type: application/json

{
  "symbol": "TCS",
  "exchange": "NSE",
  "side": "BUY",
  "order_type": "LIMIT",
  "product_type": "DELIVERY", // or INTRADAY
  "quantity": 50,
  "limit_price": 3950.00,
  "validity": "DAY"
}
```
**Response (`202 Accepted`)**:
```json
{
  "order_id": "ord_8b1092a-33b1",
  "status": "VALIDATED_AND_QUEUED",
  "blocked_margin": 197500.00,
  "timestamp": "2026-10-18T10:14:22.105Z"
}
```

#### Cancel Pending Order
```http
DELETE /api/v1/orders/ord_8b1092a-33b1
Authorization: Bearer <jwt_token>
```
**Response (`200 OK`)**:
```json
{
  "order_id": "ord_8b1092a-33b1",
  "status": "CANCEL_REQUESTED"
}
```

---

### 3.4 Portfolio & Positions REST Interface

#### Fetch Holdings with Unrealized P&L
```http
GET /api/v1/portfolio/holdings
Authorization: Bearer <jwt_token>
```
**Response (`200 OK`)**:
```json
{
  "total_investment": 500000.00,
  "current_portfolio_value": 542000.00,
  "total_unrealized_pnl": 42000.00,
  "holdings": [
    {
      "symbol": "INFY",
      "quantity": 100,
      "average_price": 1450.00,
      "ltp": 1520.00,
      "current_value": 152000.00,
      "pnl": 7000.00,
      "pnl_percentage": 4.83
    }
  ]
}
```

---

### 3.5 Real-Time WebSocket Interface

#### Market Tick Streaming (Bi-Directional Duplex)
```http
GET /ws/v1/market/stream
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer <jwt_token>
```

*Client Dynamically Subscribes / Unsubscribes to Symbols*:
```json
{
  "action": "SUBSCRIBE",
  "symbols": ["NSE:RELIANCE", "NSE:TCS", "NSE:INFY"]
}
```

*Server Broadcasts Ticks (Sub-50ms Frequency)*:
```json
{
  "type": "TICK",
  "symbol": "NSE:RELIANCE",
  "ltp": 2860.50,
  "day_change": 10.50,
  "day_change_pct": 0.37,
  "day_high": 2870.00,
  "day_low": 2845.00,
  "volume": 2841029,
  "timestamp": 1729275420050
}
```

---

## 4. Step 4: High-Level Architecture (HLD)

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/qj34gj4jnj43f34g45gergeg3y.jpeg?updatedAt=1754338445706)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 12](https://www.interviewwithbunny.com/systemdesign/12))*

### Component Interaction Topology
```
                            +─────────────────────────────────────────+
                            |     Trading Clients (Web/iOS/Android)   |
                            +─────────────────────────────────────────+
                                                 │
                   ┌─────────────────────────────┴─────────────────────────────┐
                   ▼                                                           ▼
   +───────────────────────────────+                           +───────────────────────────────+
   |     REST API Gateway & LB     |                           |   WebSocket Gateway Cluster   |
   | (Auth, Rate Limiting, Routes) |                           | (Sticky Session, Auth, Multiplex)|
   +───────────────────────────────+                           +───────────────────────────────+
                   │                                                           │
     ┌─────────────┼──────────────┬──────────────┐                             │
     ▼             ▼              ▼              ▼                             ▼
+──────────+ +───────────+ +─────────────+ +───────────+             +───────────────────+
| User Svc | | Watchlist | | Portfolio   | | Payment   |             |   Price Tracker   |
|  & KYC   | |  Service  | |   Service   | |  Service  |             |  WebSocket Server |
+──────────+ +───────────+ +─────────────+ +───────────+             +───────────────────+
     │             │              │              │                             ▲
     ▼             ▼              │              ▼                             │ Subscribes
+──────────+ +───────────+        │        +───────────+             +───────────────────+
| User DB  | | Watchlist |        │        | Wallet DB |             |  Redis Pub/Sub    |
| (Postgres| | DB (PG)   |        │        | (Ledger)  |             |  & In-Memory LTP  |
+──────────+ +───────────+        │        +───────────+             +───────────────────+
                                  │                                            ▲
                                  │ Reads Positions                            │ Pushes Ticks
                                  ▼                                            │
                           +───────────────+                         +───────────────────+
                           |   Trade DB    |                         |  Price Ingestor   |
                           | (PostgreSQL)  |                         |  Consumer Worker  |
                           +───────────────+                         +───────────────────+
                                  ▲                                            ▲
                                  │ Writes Fills                               │ Reads Ticks
                                  │                                            │
+─────────────────────────────────┴────────────────────────────────────────────┴──────────────────────+
|                                        Apache Kafka Event Bus                                        |
|   [raw-orders]     [verified-orders]     [rejected-orders]     [stock-prices]     [order-status]     |
+──────────────────────────────────────────────────────────────────────────────────────────────────────+
          │                  ▲                     ▲                   ▲                   ▲
          ▼                  │                     │                   │                   │
+───────────────────+        │                     │                   │                   │
| Order Validator   |────────┴─────────────────────┘                   │                   │
| (RMS/Risk Engine) |                                                  │                   │
+───────────────────+                                                  │                   │
          │                                                            │                   │
          ▼ Consumes verified                                          │                   │
+───────────────────+                                                  │                   │
|   Order Service   |                                                  │                   │
+───────────────────+                                                  │                   │
     │         │                                                       │                   │
     │ Writes  │ Dispatches order                                      │                   │
     ▼ Pending ▼                                                       │                   │
+─────────+  +─────────────────────────────────────────────────────────┴───────────────────+
| Order DB|  |                             Exchange Gateway Cluster                        |
| (PG)    |  |     (Maintains ~10 Multiplexed WebSockets to NSE/BSE & Order FIX Pipes)     |
+─────────+  +─────────────────────────────────────────────────────────────────────────────+
                                                   ▲
                                                   │ FIX Protocol / OUCH / ITCH
                                                   ▼
                                     +───────────────────────────+
                                     |  National Stock Exchange  |
                                     |       (NSE / BSE)         |
                                     +───────────────────────────+
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Real-Time Market Feed Ingestion: WebSocket vs. Server-Sent Events (SSE)

#### The Controversy Addressed by Bunny
In system design interviews, candidates often propose **Server-Sent Events (SSE)** for stock updates because market data is primarily a one-way server-to-client push. However, Bunny highlights why production trading systems (Zerodha Kite, Upstox) adopt **full-duplex WebSockets**:

```
SERVER-SENT EVENTS (SSE) LIMITATION           WEBSOCKET DUPLEX PIPELINE (CHOSEN)
Client                               Server   Client                               Server
  │                                    │        │                                    │
  │─── GET /stream?symbols=RELIANCE ──►│        │◄────── Bi-Directional TCP Handshake ──────►│
  │◄── Price RELIANCE: 2850 ───────────│        │                                    │
  │                                    │        │─── {"action":"SUB","sym":"INFY"} ─►│ (No reconnect!)
  │ User wants to view INFY also:      │        │◄── Price INFY: 1520 ───────────────│
  │ MUST TEAR DOWN TCP CONNECTION!     │        │─── {"action":"UNSUB","sym":"TCS"} ─►│
  │─── GET /stream?symbols=REL,INFY ──►│        │                                    │
  │ (Excessive socket thrashing)       │        │ Zero connection teardown overhead  │
```

1. **Dynamic Client Subscriptions Without Reconnection**: When a trader navigates between watchlists or clicks on a stock chart, they dynamically subscribe/unsubscribe to 50+ symbols in seconds. With SSE, modifying subscriptions requires destroying and re-establishing the HTTP stream. With WebSockets, subscription payloads are sent as lightweight inline frames over the existing persistent TCP connection.
2. **Exchange Tier Multiplexing**: Stock exchanges (NSE/BSE) limit brokers to a specific number of binary WebSocket/TCP sessions. Each connection carries a stream of up to 500 symbols. Zerodha opens ~10 to 15 multiplexed persistent WebSockets to ingest all 5,000 listed Indian stocks.

---

### 5.2 Market Feed Architecture: Kafka Buffering, Redis Pub/Sub & InfluxDB

```
                                      Exchange Gateway
                              (Ingests 60,000 raw ticks/sec)
                                             │
                                             ▼
                             Kafka Topic: [stock-prices]
                    (Partitioned by Symbol Hash: hash(symbol) % N)
                                             │
                                             ▼
                                Price Ingestor Consumer
                                             │
                 ┌───────────────────────────┴───────────────────────────┐
                 ▼                                                       ▼
+─────────────────────────────────+                     +─────────────────────────────────+
|     InfluxDB / TimescaleDB      |                     |         Redis Cluster           |
| (Downsampled OHLCV Candles)     |                     | (Pub/Sub Channel per Symbol     |
+─────────────────────────────────+                     |  + String Key: LTP Cache)       |
                 │                                      +─────────────────────────────────+
                 │ Serves historical                                     │
                 │ charts (Past 10 days)                                 │ Pushes sub-50ms ticks
                 ▼                                                       ▼
         Price Tracker Svc                                      Price Tracker Svc
                 │                                                       │
                 └───────────────────────────┬───────────────────────────┘
                                             ▼
                                 WebSocket Gateway Cluster
                                             │
                                             ▼
                                    Trader App Screen
```

#### Dual-Storage Justification: InfluxDB vs. Redis
- **Redis Cluster (In-Memory Hot Path)**:
  - Stores the current **Last Traded Price (LTP)** as a simple key-value: `SET stock:LTP:NSE:RELIANCE 2860.50`.
  - Publishes real-time tick packets via **Redis Pub/Sub** on channels keyed by symbol: `PUBLISH channel:NSE:RELIANCE <tick_payload>`.
  - Connected WebSocket servers subscribe only to the symbols currently open on their active clients' screens, minimizing memory footprint.
- **InfluxDB (Time-Series Cold/Warm Path)**:
  - Relational databases struggle with continuous write-heavy timeseries inserts. InfluxDB uses a **Time-Structured Merge Tree (TSM)** optimized for append-only timestamped float values.
  - Generates aggregate candlestick rollups (1m, 5m, 1h) via continuous background queries without impacting Redis live tick streaming.

---

### 5.3 Order Lifecycle & Pre-Trade Risk Management (RMS) Engine

Before any order is dispatched to the exchange, it passes through the **Risk Management System (RMS)**. Exchanges heavily penalize brokers whose orders bounce due to insufficient margin or invalid trading states.

```
Trader Places Order
        │
        ▼
API Gateway Ingests & Publishes to Kafka [raw-orders]
        │
        ▼
Order Validator Service (RMS Engine)
        │
        ├── 1. Account Check: Is user KYC verified and not banned?
        ├── 2. Market State Check: Is exchange currently open (09:15 - 15:30)?
        ├── 3. Circuit Breaker Check: Is stock frozen in lower/upper circuit?
        └── 4. Margin Check: Does Wallet have sufficient unencumbered cash?
                    │
                    ├── INSUFFICIENT FUNDS ──────────────┐
                    │                                    ▼
                    │                     Publish to [rejected-orders]
                    │                     Write to Order DB: REJECTED
                    │                     Push Error to User Screen
                    │
                    └── SUFFICIENT FUNDS
                            │
                            ▼
           Atomic Margin Reservation (Wallet DB)
           Publish to Kafka [verified-orders]
                            │
                            ▼
              Order Service Consumes Event
              Insert into Order DB (Status: PENDING)
                            │
                            ▼
              Dispatched to Exchange Gateway
```

#### Atomic Margin Deduction via Database Locking
```sql
-- Deduct margin atomically before marking order verified
UPDATE wallet_accounts
SET available_margin = available_margin - :required_order_amount,
    updated_at = NOW()
WHERE user_id = :user_id 
  AND available_margin >= :required_order_amount;

-- If rows affected == 0, abort order immediately with INSUFFICIENT_MARGIN
```

---

### 5.4 Partial Fills, Limit Orders & Asynchronous Trade Reconciliation

#### The Reality of Order Fulfillment
When placing a limit order for 1,000 shares of Reliance at ₹2,850, the order does not instantly execute. It rests in the exchange order book and may be filled in piecemeal chunks across several minutes:
- **T + 0s**: Order Placed $\to$ Exchange acknowledges with `exchange_order_id`. Status: `OPEN`.
- **T + 12s**: 300 shares matched at ₹2,850. Exchange emits Execution Report 1. Status: `PARTIALLY_FILLED`.
- **T + 45s**: 700 shares matched at ₹2,850. Exchange emits Execution Report 2. Status: `COMPLETED`.

#### Why Separate `Order DB` and `Trade DB`?
Bunny emphasizes this structural separation as a mandatory interview design pattern:
1. **Order DB (`orders` table)**:
   - Tracks the **intent of the user**.
   - Contains rejected orders, cancelled orders, expired orders, and pending orders.
2. **Trade DB (`trades` table)**:
   - Tracks **actual executed transactions confirmed by the exchange**.
   - Contains immutable records containing authoritative `exchange_trade_id`, settlement tokens, and clearing timestamps.
   - **End-of-Day (EOD) Reconciliation**: At 16:00 IST, the broker downloads the official Trade File from the NSE clearing house. A batch reconciliation job verifies that the broker's `Trade DB` matches the clearing corporation's logs to the exact penny.

---

### 5.5 Database Schemas

#### PostgreSQL (ACID Relational Core for Trades & Orders)

```sql
-- User Master
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    is_kyc_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Wallet Ledger & Margin Table
CREATE TABLE wallet_accounts (
    wallet_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    ledger_balance NUMERIC(14, 2) NOT NULL DEFAULT 0.00,
    available_margin NUMERIC(14, 2) NOT NULL DEFAULT 0.00,
    currency VARCHAR(3) DEFAULT 'INR',
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT positive_margin CHECK (available_margin >= 0)
);

-- Master Order Table (User Intent)
CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    symbol VARCHAR(30) NOT NULL,
    exchange VARCHAR(10) NOT NULL, -- NSE, BSE
    side VARCHAR(4) NOT NULL,      -- BUY, SELL
    order_type VARCHAR(10) NOT NULL, -- MARKET, LIMIT, SL
    product_type VARCHAR(10) NOT NULL, -- CNC (Delivery), MIS (Intraday)
    target_price NUMERIC(10, 2),
    quantity INTEGER NOT NULL,
    filled_quantity INTEGER DEFAULT 0,
    status VARCHAR(25) NOT NULL DEFAULT 'PENDING', -- PENDING, REJECTED, OPEN, PARTIALLY_FILLED, COMPLETED, CANCELLED
    exchange_order_id VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_orders_user ON orders (user_id, created_at DESC);
CREATE INDEX idx_orders_exchange_ref ON orders (exchange_order_id);

-- Trade Execution Master Table (Exchange Ground Truth)
CREATE TABLE trades (
    trade_id VARCHAR(50) PRIMARY KEY, -- Official Exchange Trade ID
    order_id UUID NOT NULL REFERENCES orders(order_id),
    user_id UUID NOT NULL REFERENCES users(user_id),
    symbol VARCHAR(30) NOT NULL,
    execution_price NUMERIC(10, 2) NOT NULL,
    filled_quantity INTEGER NOT NULL,
    brokerage_fee NUMERIC(8, 2) DEFAULT 20.00,
    stt_tax NUMERIC(8, 2) NOT NULL,
    exchange_timestamp TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_trades_user_symbol ON trades (user_id, symbol);

-- User Portfolio Holding Table (Settled Equity)
CREATE TABLE portfolio_holdings (
    holding_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    symbol VARCHAR(30) NOT NULL,
    isin VARCHAR(20) NOT NULL,
    total_quantity INTEGER NOT NULL,
    average_price NUMERIC(10, 2) NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (user_id, symbol)
);
```

---

### 5.6 Real-Time Portfolio P&L Calculation Engine

```
+───────────────────────+                        +───────────────────────+
|       Trade DB        |                        |    Redis LTP Cache    |
| (Holds User Quantity  |                        |  (Holds live market   |
|  & Average Buy Price) |                        |   price per symbol)   |
+───────────────────────+                        +───────────────────────+
            │                                                │
            │ Fetches:                                       │ Fetches:
            │ Qty = 100, Avg = ₹1,450                        │ LTP = ₹1,520
            └───────────────────────┬────────────────────────┘
                                    ▼
                         +─────────────────────+
                         |  Portfolio Service  |
                         +─────────────────────+
                                    │
                                    │ Computes:
                                    │ Unrealized P&L = (1520 - 1450) * 100 = +₹7,000 (+4.83%)
                                    ▼
                         +─────────────────────+
                         | Push to Client App  |
                         | Screen in Real Time |
                         +─────────────────────+
```

$$\text{Unrealized P\&L} = \sum_{i=1}^{M} \left( \text{LTP}_i - \text{AvgPrice}_i \right) \times \text{Quantity}_i$$

1. When a user opens their Portfolio tab, the client requests positions.
2. The Portfolio Service pulls settled holdings from `portfolio_holdings` and intraday fills from `trades`.
3. The Portfolio Service reads the current **LTP** from the Redis cache in $< 1\text{ ms}$.
4. If the user keeps the dashboard open, the Portfolio Service connects to the internal Redis Pub/Sub stream for those held symbols, broadcasting updated P&L figures whenever a new tick arrives.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you ensure real-time price updates reach millions of users with <50ms latency?

Real-time price distribution with WebSocket + Redis PubSub:

- **Price ingestion**:
  Price Tracker Svc subscribes to NSE/BSE data feed (WebSocket/FIX protocol), receives tick-by-tick updates for 8-10k stocks, Example: {symbol: 'RELIANCE', price: 2458.75, volume: 125000, timestamp}.

- **Dual write**:
  Write to InfluxDB (time-series persistence): INSERT stock_prices (symbol='RELIANCE', price=2458.75, volume=125000, timestamp), Async write, high throughput (100K+ writes/sec), Used for historical charts, not real-time push. Publish to Redis PubSub: PUBLISH stock_price '{"symbol": "RELIANCE", "price": 2458.75, "change": +0.45%}', Fan-out to all subscribers instantly (<10ms), No persistence (in-memory pub/sub).

- **WebSocket servers subscribe**:
  10 WebSocket gateway servers (horizontally scaled), Each server: SUBSCRIBE stock_price (Redis PubSub channel), Receives all price updates in real-time, Filters and pushes to connected clients.

- **Client connections**:
  User opens app → WebSocket connection: ws://api.trading.com/prices, Handshake: Client sends watchlist: {action: 'subscribe', symbols: ['RELIANCE', 'TCS', 'INFY']}, Server stores subscription: userConnections[userId] = {socket, symbols: ['RELIANCE', 'TCS', 'INFY']}.

- **Selective push**:
  Redis PubSub message received: {symbol: 'RELIANCE', price: 2458.75}, Server iterates connected users: for each user watching 'RELIANCE', send via WebSocket: socket.send({symbol: 'RELIANCE', price: 2458.75}), Only users watching RELIANCE receive update (not all 8-10k stock updates), Reduces bandwidth 1000× (each user watches ~10 stocks, not all 8-10k).

- **Latency breakdown**:
  Exchange → Price Tracker: 5ms (network), Price Tracker → Redis PubSub: 2ms (PUBLISH command), Redis PubSub → WebSocket server: 3ms (in-memory fan-out), WebSocket server → Client: 10ms (internet latency), Total: ~20ms (well under 50ms target).

- **Scaling**:
  Horizontal: Add more WebSocket servers (each handles 10K connections), Load balancer: Sticky sessions (user_id hash → same server), maintains WebSocket state, Redis PubSub: Single channel, all servers subscribe (fan-out), no coordination needed.

- **Fallback**:
  If WebSocket disconnects: Client auto-reconnects with exponential backoff (1s, 2s, 4s, max 30s), On reconnect: Re-subscribe to watchlist, fetch missed updates from REST API (GET /api/v1/stocks/{symbol}). Result: Sub-50ms real-time updates, scales to millions of concurrent users, efficient bandwidth usage (selective push), resilient (auto-reconnect on failure).

---

### Q2: How do you prevent overdraft when multiple orders are placed simultaneously?

Fund locking with database transactions and pessimistic locking:

- **Problem**:
  User has ₹50,000 available balance, Places 2 simultaneous MARKET BUY orders: Order A: Buy 20 RELIANCE @ ₹2459 = ₹49,180, Order B: Buy 5 TCS @ ₹3500 = ₹17,500, Total required: ₹49,180 + ₹17,500 = ₹66,680 > ₹50,000 (overdraft!), Without locking: Both orders check balance (₹50,000 > ₹49,180 ✓, ₹50,000 > ₹17,500 ✓), Both pass validation, both execute → overdraft by ₹16,680.

- **Solution**:
  Pessimistic locking with SELECT FOR UPDATE: Order A processing: BEGIN TRANSACTION, SELECT available_balance FROM funds WHERE user_id={user_id} FOR UPDATE (locks row, other transactions wait), available_balance = ₹50,000, Check: ₹50,000 >= ₹49,180 ✓ (pass), UPDATE funds SET available_balance = 50000 - 49180 = ₹820, locked_balance = 0 + 49180 = ₹49,180, COMMIT (releases lock).

- **Order B processing (simultaneously)**:
  BEGIN TRANSACTION, SELECT available_balance FROM funds WHERE user_id={user_id} FOR UPDATE, Waits for Order A transaction to commit (row locked), Once Order A commits: reads available_balance = ₹820 (updated value), Check: ₹820 >= ₹17,500 ✗ (fail), ROLLBACK, Return error: 400 'Insufficient funds'.

- **Transaction isolation**:
  Isolation level: SERIALIZABLE (highest level), Prevents: Dirty reads (read uncommitted data), Non-repeatable reads (data changes mid-transaction), Phantom reads (new rows appear), Guarantees: Each transaction sees consistent snapshot.

- **Alternative**:
  Optimistic locking with version: funds table has version column (starts at 1), Order A: SELECT available_balance, version FROM funds WHERE user_id={user_id}, Read: balance=₹50,000, version=1, UPDATE funds SET available_balance=820, locked_balance=49180, version=2 WHERE user_id={user_id} AND version=1, If version still 1: update succeeds (1 row affected), If version changed (Order B updated first): update fails (0 rows affected), Retry with new version.

- **Optimistic vs Pessimistic**:
  Pessimistic (SELECT FOR UPDATE): Use when contention is high (many users trading simultaneously), Blocks concurrent updates (serializes), Guarantees correctness at cost of throughput. Optimistic (version check): Use when contention is low (rare simultaneous orders from same user), Allows concurrent reads, retry on conflict, Higher throughput, but retries add latency.

- **Production choice**:
  Use pessimistic locking for fund updates (correctness critical, can't overdraft), Use optimistic locking for portfolio updates (can tolerate retries, less critical). Result: No overdraft possible, ACID guarantees maintained, concurrent orders handled correctly, trade-off: slight latency increase for correctness (acceptable for financial transactions).

---

### Q3: Walk through complete order flow from placement to execution with all validations and state transitions.

Complete order flow (MARKET BUY order):

- **User places order**:
  POST /api/v1/orders {stockSymbol: 'RELIANCE', orderType: 'MARKET', tradeType: 'BUY', quantity: 10}.

- **Order Svc validation**:
  Check JWT token valid (user authenticated), Validate stock_symbol: SELECT EXISTS FROM stocks WHERE stock_symbol='RELIANCE', if not exists → 400 'Invalid stock', Check market hours: if currentTime NOT BETWEEN 9:15 AM AND 3:30 PM IST → 400 'Market closed', Check quantity: must be positive int, multiple of lot size (1 for equity), Generate order_id: UUID.

- **Fund check**:
  Query current price: Redis GET stock_price:RELIANCE → ₹2458.75, Estimated cost: 10 × 2458.75 + ₹20 commission = ₹24,607.50, Fund locking: BEGIN TRANSACTION, SELECT available_balance FROM funds WHERE user_id={user_id} FOR UPDATE, if available_balance < 24607.50 → ROLLBACK, return 400 'Insufficient funds', UPDATE funds SET available_balance -= 24607.50, locked_balance += 24607.50, COMMIT.

- **Insert order**:
  INSERT INTO orders (order_id, user_id, stock_symbol, order_type: 'MARKET', trade_type: 'BUY', quantity: 10, price: null, status: 'PENDING', placed_at: now()), Status: PENDING (initial state).

- **Kafka publish**:
  Kafka.send('new_orders', {orderId, userId, stockSymbol, orderType, tradeType, quantity}), Response to user: 200 OK {orderId: 'ORD123', status: 'PENDING'}.

- **Validator consumes 'new_orders'**:
  Duplicate check: Redis SETNX order_lock:{orderId} 1 EX 30, if already exists (SETNX returns 0) → duplicate, reject, Risk check: Total exposure: SELECT SUM(quantity × price) FROM orders WHERE user_id={user_id} AND status IN ('PENDING','PLACED'), if exposure > 10× available_balance → reject (excessive risk), Circuit breaker: SELECT price FROM stock_prices WHERE symbol='RELIANCE' AND timestamp > NOW() - 5 MIN, if price volatility > 10% in 5 min → reject (market volatility), If all pass: UPDATE orders SET status='VERIFIED', Publish Kafka 'verified_orders': {orderId}.

- **Exchange Gateway consumes 'verified_orders'**:
  Format FIX message (Financial Information eXchange): Tag 35=D (NewOrderSingle), Tag 55='RELIANCE' (Symbol), Tag 54=1 (Side: Buy), Tag 38=10 (Quantity), Tag 40=1 (OrdType: Market), Send via TCP to NSE matching engine, UPDATE orders SET status='PLACED', exchange_order_id={assigned_by_exchange}.

- **NSE matching engine**:
  Matches BUY order with best available SELL order(s), Execution price: ₹2459.00 (market price at execution time, may differ from cached ₹2458.75), Sends FIX ExecutionReport: Tag 35=8, Tag 150=2 (ExecType: Filled), Details: {executedPrice: 2459.00, executedQuantity: 10}.

- **Exchange Gateway receives execution**:
  Publish Kafka 'order_status': {orderId, status: 'EXECUTED', executedPrice: 2459.00, executedQuantity: 10, executedAt: timestamp}.

- **Order Tracker Svc consumes 'order_status'**:
  UPDATE orders SET status='EXECUTED', executed_at=now(), INSERT INTO trades (trade_id, order_id, stock_symbol, quantity: 10, executed_price: 2459.00, executed_at: now(), exchange: 'NSE', commission_fee: 20).

- **Fund update**:
  Actual cost: 10 × 2459 + 20 = ₹24,610 (vs estimated ₹24,607.50), BEGIN TRANSACTION, UPDATE funds SET locked_balance -= 24607.50 (unlock estimated), available_balance -= (24610 - 24607.50) = -2.50 (deduct difference), COMMIT.

- **Portfolio Svc consumes 'order_status'**:
  Check existing holding: SELECT quantity, avg_buy_price FROM portfolio WHERE user_id={user_id} AND stock_symbol='RELIANCE', If new holding: INSERT INTO portfolio (user_id, stock_symbol, quantity: 10, avg_buy_price: 2459.00, total_investment: 24590), If existing: UPDATE portfolio SET quantity += 10, avg_buy_price = (prev_qty × prev_avg + 10 × 2459) / (prev_qty + 10), total_investment = quantity × avg_buy_price.

- **Notification Svc consumes 'order_status'**:
  Push notification via FCM: {title: 'Order Executed', body: 'RELIANCE BUY 10 @ ₹2459.00'}, WebSocket push (if user online): {type: 'ORDER_STATUS', orderId, status: 'EXECUTED'}. State transitions: PENDING (order created, funds locked) → VERIFIED (validations passed) → PLACED (sent to exchange) → EXECUTED (trade confirmed) OR REJECTED (validation failed, funds unlocked). Result: End-to-end order flow with validations, fund locking prevents overdraft, state tracking enables user visibility, Kafka decouples stages for scalability.

---

## 7. Architectural Trade-Off Analysis Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Market Data Streaming** | **Bi-directional WebSocket Gateway** | Server-Sent Events (SSE) | SSE forces tearing down and reopening HTTP connections whenever a user changes watched symbols. WebSockets allow dynamic multiplexed subscription frames over an open socket. |
| **Market Price Storage** | **Time-Series DB (InfluxDB)** | Relational DB (PostgreSQL) | 60,000 ticks/sec would exhaust PostgreSQL WAL and B-Tree indexes. InfluxDB's TSM engine handles append-only high-frequency writes and automatic candle rollups. |
| **Order Ingestion Pipeline** | **Asynchronous Kafka Queue (`[raw-orders]`)** | Synchronous Direct RPC to Exchange | Direct RPC cascades external exchange latency ($> 200\text{ ms}$) to client threads, saturating web workers. Kafka provides back-pressure absorption and guaranteed delivery. |
| **Trade Data Architecture** | **Separate `Order DB` and `Trade DB`** | Unified `orders` table only | Orders represent volatile user intents (including rejected and cancelled). Trades represent immutable financial reality required for regulatory EOD clearing and audit compliance. |
| **Consistency vs. Availability** | **Strict Consistency (CP)** | Eventual Consistency (AP) | Financial trading requires ACID compliance. A temporary service error is vastly preferable to executing an order with unverified margins or desynchronized wallet balances. |
| **Live Tick Distribution** | **Redis Pub/Sub Layer** | Direct Kafka-to-Client Streaming | Exposing Kafka directly to 1M end-users causes massive consumer group overhead. Redis Pub/Sub provides sub-millisecond in-memory fan-out to local WebSocket gateway pods. |
