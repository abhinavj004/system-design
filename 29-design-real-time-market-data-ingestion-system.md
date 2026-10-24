# System Design 29: Real-Time Market Data Ingestion & Distribution System

> **Topic**: Designing a Ultra-Low Latency, High-Frequency Market Data Ingestion and Distribution Platform  
> **Key Protocols**: High-Throughput UDP Multicast Feed Ingestion, Reliable SBE/ITCH Streaming, NACK-based Retransmission, and Multi-Tier Downstream Fanout  
> **Scale**: 20 Million Ticks/Sec Peak Ingestion across Global Exchanges (NASDAQ, NIFTY, S&P 500), Sub-Millisecond Internal Latency, 1 Million Connected Retail & Trading Actor Terminals

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Global Exchanges (NASDAQ, NIFTY) stream raw ticks to Upstream Loaders $	o$ Loaders broadcast to Processor servers over **UDP Multicast using the MoldUDP64 protocol** (64-bit sequence numbers) $	o$ Processors ingest via **Kernel-Bypass NICs (Solarflare EF_VI / DPDK)** directly into an **LMAX Disruptor Lock-Free Ring Buffer** (zero GC allocation) $	o$ Packet loss detected via sequence gaps and recovered via **TCP NACK Retransmission** $	o$ Real-time ticks stream via **Aeron IPC** to algorithmic trading bots ($< 50\mu	ext{s}$) while conflated 100ms updates fan out to retail WebSockets $	o$ All ticks batch-flushed to **ClickHouse** for historical replay.

| Metric / Dimension | Production Target / Value |
| :--- | :--- |
| **Ingestion Throughput** | 20,000,000 ticks/sec peak across global market sessions |
| **Internal Latency** | $< 500\mu	ext{s}$ loader-to-processor; $< 50\mu	ext{s}$ ring buffer |
| **Retail Fanout SLA** | $< 10	ext{ms}$ conflated updates over WebSockets |
| **Daily Data Ingestion** | 57 Billion daily ticks ($pprox 4.5	ext{ TB/day}$ uncompressed) |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **Why UDP Multicast instead of TCP or Apache Kafka?**:
   - *Defense*: TCP suffers from head-of-line blocking (1 dropped packet stalls all subsequent packets); Kafka adds 5–15ms disk/broker overhead. UDP Multicast allows 1 loader to broadcast to 100 processor nodes simultaneously with single-digit microsecond latency without duplicating network packets.
2. **Handling UDP Packet Loss without TCP Connection Overhead**:
   - *Defense*: Packets use the **MoldUDP64** protocol containing an incremental 64-bit sequence number per feed. If a processor detects a sequence gap (e.g. receives seq 104 after 101), it fires an out-of-band **NACK over TCP** to the loader's in-memory ring buffer (storing the last 60 seconds of ticks) for immediate retransmission.
3. **Preventing a "NACK Storm" during Network Switch Drops**:
   - *Defense*: Processors apply **Randomized Exponential Backoff Jitter** (50–200 $\mu$s) before requesting a NACK. Loaders broadcast retransmitted packets onto a dedicated **Recovery Multicast Group**, allowing all 50 dropped processors to recover simultaneously from a single retransmitted packet.

> **Golden Architectural Rule**: Use UDP Multicast with MoldUDP64 sequence tracking and out-of-band TCP NACK retransmission for zero-copy tick ingestion; process events via LMAX Disruptor ring buffers with zero garbage-collection pauses.

---

## 1. Architectural Pattern Lineage & Transferred Learnings

This financial streaming engine connects directly to the core primitives established across our master notes:

```
┌───────────────────────────────────────┬───────────────────────────────────────────┬──────────────────────────────────────────┐
│ Subsystem Challenge in Market Data    │ Transferred Architectural Pattern         │ Foundational Lineage in Our Master Notes │
├───────────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────┤
│ Real-Time Market Tick Broadcasting    │ WebSocket Gateways & Price Conflation     │ Episode 12 (Zerodha Stock Broker)        │
│ Atomic Tick Sequencing & Zero-Locking │ In-Memory Ring Buffer (LMAX Disruptor)    │ Problem 24 (Online Auction Platform)     │
│ Packet Reordering & Deduplication     │ Sliding Window Buffer & Bitmask Tracking  │ Episode 16 (Distributed Logging / Splunk)│
│ High-Density Concurrent User Fanout   │ Epoll Tuning & Viewport Subscriptions     │ Episode 27 (Real-Time Presence System)   │
│ Asynchronous Historical Tick Archival │ Zero-Loss Append Log & Kafka Partitioning │ Episode 16 (Splunk) & Ep 03 (E-Commerce) │
│ Rapid State Recovery on Catastrophic  │ Snapshot + Commitlog Delta Replay         │ Episode 10 (Google Docs) & Ep 02 (BMS)   │
│ Packet Gap Mitigation & Retries       │ NACK Exponential Backoff & Circuit Breaker│ Episode 09 (Scheduler) & Ep 21 (Limiter) │
└───────────────────────────────────────┴───────────────────────────────────────────┴──────────────────────────────────────────┘
```

---

## 2. Jargon Buster: Financial & Low-Latency Concepts in Plain English

Before diving into the architecture, here are the essential industry terms translated into intuitive software engineering concepts:

1. **What is a Market Tick?**
   - *Plain English*: The smallest atomic event emitted by a stock exchange. Every time a trade happens or an order is placed/cancelled, the exchange broadcasts a tick:
     `{Symbol: "AAPL", Price: 182.50, Size: 100, Side: BUY, Timestamp: 14:02:01.123456789, SeqNum: 8849201}`.
2. **Why does Finance use UDP instead of TCP for Ingestion?**
   - *Plain English*: **TCP has Head-of-Line Blocking**. If packet #3 is dropped over TCP, the operating system stops packet #4, #5, and #6 until packet #3 is re-sent. In volatile markets where 100,000 ticks arrive every millisecond, waiting 20ms for TCP retransmission stalls the entire market view.
   - **UDP fires packets without waiting**. If packet #3 drops, packets #4 and #5 are processed instantly. We build our own lightweight recovery mechanism specifically tailored for financial data.
3. **What is MoldUDP64 / ITCH / FAST?**
   - *Plain English*: Standard binary exchange protocols (used by NASDAQ and CME). Instead of bulky JSON text (`{"price": 182.5}` $\approx 50\text{ bytes}$), exchanges encode ticks into tiny binary byte arrays ($\approx 20\text{–}40\text{ bytes}$) with strict integer sequence numbers.
4. **What is a NACK (Negative Acknowledgment)?**
   - *Plain English*: In normal web systems (TCP), the receiver sends an **ACK** ("I got packet 1, I got packet 2"). Sending millions of ACKs floods the network.
   - In financial data, we use **NACK**: The receiver stays silent as long as numbers increment (`1, 2, 3, 4`). Only when it detects a gap (`1, 2, 4` $\to$ missing #3) does it shout: *"Hey Loader, I missed packet #3! Send it to me via the TCP backup channel!"*
5. **What is Tick Conflation (Throttling)?**
   - *Plain English*: During a market crash, Apple stock might change price 10,000 times in one second. Human eyes cannot see 10,000 updates a second, and mobile phone screens will freeze.
   - **Conflation** merges intermediate price ticks over a small window (e.g., every 100ms) and sends only the *latest state* to retail browsers, while streaming raw, unthrottled ticks to algorithmic trading servers.

---

## 3. Step 1: Requirements Gathering & Estimations

### 3.1 Functional Requirements (FR)
1. **Multi-Exchange Ingestion**: Ingest high-frequency market feeds from global exchanges (NASDAQ, NIFTY, S&P 500) via upstream loaders.
2. **Lossless UDP Processing**: Ingest data over UDP while guaranteeing zero permanent data loss, zero duplicates, and strict chronological ordering across downstream consumers.
3. **Internal Gap Detection & Recovery**: Detect packet drops, deduplicate redelivered packets, and re-request missing gaps via a secondary TCP backchannel.
4. **Order Book Maintenance**: Aggregate raw trade and quote ticks into live Top-of-Book (BBO: Best Bid / Offer) and Level 2 market depth states.
5. **Multi-Tiered Downstream Distribution**:
   - Ultra-low latency raw streaming for institutional trading actors and automated bots ($< 1\text{ms}$).
   - Conflated (throttled) WebSocket streaming for 1 Million concurrent retail mobile/web traders ($< 100\text{ms}$).
6. **Historical Audit Trail**: Persist 100% of raw market ticks for audit compliance, chart generation, and algorithmic backtesting.

### 3.2 Non-Functional Requirements (NFR)
1. **Ultra-Low Latency (Sub-Millisecond Ingestion-to-Processor SLA)**: Internal processing latency from UDP packet arrival to normalized memory publication must be $< 500\text{ microseconds}$ (p99).
2. **High Throughput**: Must comfortably ingest **20 Million ticks/second** at market opening bell and economic data release spikes.
3. **High Availability (Zero Single Point of Failure)**: Dual active-active upstream loaders per exchange. Failure of any single loader or processor must cause zero downtime and zero gap in the processed feed.
4. **Data Consistency**: Monotonically increasing sequence order per financial instrument; no out-of-order execution states.

---

### 3.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Ingestion Throughput Math
- **Global Exchanges Tracked**: NASDAQ, NYSE, CME, NIFTY, Eurex ($\approx 50,000$ active symbols).
- **Average Traffic**: 2 Million ticks/second.
- **Peak Market Spike (Opening Bell / FOMC Rate Decision)**: **20 Million ticks/second**.
- **Tick Packet Size (Binary SBE / MoldUDP64)**:
  - Header: 24 bytes (Session ID, Sequence Number, Timestamp, Message Count).
  - Payload per Tick: 40 bytes (Symbol ID, Price [int64], Quantity [int32], Flags, Bid/Ask Price).
  - Total wire size per packet (carrying 1 tick) $\approx \mathbf{64 \text{ bytes}}$.
- **Inbound Network Bandwidth at Peak**:
  $$\text{Peak Network Bandwidth} = 20,000,000 \text{ ticks/sec} \times 64 \text{ bytes} \approx 1,280,000,000 \text{ bytes/sec} \approx \mathbf{1.28 \text{ GB/sec (10.24 Gbps)}}$$
  *(Fits cleanly over dual-bonded 10GbE / 25GbE enterprise network interface cards).*

#### Downstream Fanout Math
- **Retail Concurrent Users**: 1,000,000 active app sessions.
- **Conflation Window**: 100 milliseconds (10 updates per second per subscribed stock).
- Average watchlist size per user: 10 symbols.
- Outbound retail fanout:
  $$\text{Outbound Retail QPS} = 1,000,000 \times 10 \text{ updates/sec} = \mathbf{10,000,000 \text{ updates/sec across WebSocket gateway fleet}}.$$

#### Daily Storage Math
- Active trading day: 8 hours.
- Average ticks per day: $2\text{M ticks/sec} \times 3600 \times 8 \approx \mathbf{57.6 \text{ Billion ticks/day}}$.
- Raw Daily Storage:
  $$\text{Storage} = 57.6\text{B} \times 64 \text{ bytes} \approx \mathbf{3.68 \text{ TB / day (uncompressed)}}.$$
  With Zstandard columnar compression (Parquet in ClickHouse/TimescaleDB), compresses to **$\approx 600 \text{ GB / day}$**.

---

## 4. Step 2: Identification of Core Entities

```
+------------------------------------+             +------------------------------------+
|       ExchangeTick (Binary SBE)    |             |          OrderBookLevel2           |
+------------------------------------+             +------------------------------------+
| sequence_number : UINT64 (Monotonic)             | symbol_id       : UINT32 (PK)      |
| exchange_id     : UINT8 (1=NASDAQ) |             | best_bid_price  : INT64            |
| symbol_id       : UINT32           | 1         1 | best_bid_size   : UINT32           |
| price           : INT64 (Fixed Dec)|<----------->| best_ask_price  : INT64            |
| volume          : UINT32           |             | best_ask_size   : UINT32           |
| trade_side      : ENUM(BUY, SELL)  |             | bids_ladder     : ARRAY[Price,Qty] |
| exchange_ts_ns  : UINT64 (Nano-UTC)|             | asks_ladder     : ARRAY[Price,Qty] |
| ingest_ts_ns    : UINT64 (Nano-UTC)|             | last_sequence_no: UINT64           |
+------------------------------------+             +------------------------------------+
                  │                                                 │
                  │ 1                                               │ 1
                  ▼ *                                               ▼ *
+------------------------------------+             +------------------------------------+
|        GapRequest (NACK Entity)    |             |       ClientSubscription           |
+------------------------------------+             +------------------------------------+
| request_id      : UUID             |             | client_id       : UUID             |
| exchange_id     : UINT8            |             | channel_type    : ENUM(HFT, RETAIL)|
| channel_id      : UINT16           |             | subscribed_syms : SET[UINT32]      |
| missing_from_seq: UINT64           |             | conflation_ms   : UINT16 (0 or 100)|
| missing_to_seq  : UINT64           |             | gateway_node_id : VARCHAR(32)      |
+------------------------------------+             +------------------------------------+
```

---

## 5. Step 3: Wire Protocols & Ingestion Packet Design

### 5.1 The UDP Packet Format (MoldUDP64 Framing)
To minimize overhead, loaders transmit UDP datagrams containing a standardized 24-byte header followed by binary ticks:

```
+---------------------------------------------------------------------------------------+
|                               MOLDUDP64 PACKET STRUCTURE                              |
+---------------------------------------------------------------------------------------+
| Session ID (10 Bytes) | Sequence Number (8 Bytes) | Message Count (2 Bytes)           |
+-----------------------+---------------------------+-----------------------------------+
| Msg #1 Length (2 B)   | Binary Market Tick Payload (38 Bytes)                         |
+-----------------------+---------------------------------------------------------------+
| Msg #2 Length (2 B)   | Binary Market Tick Payload (38 Bytes)                         |
+-----------------------+---------------------------------------------------------------+
```

- **Session ID (10 Bytes)**: Distinguishes between exchange sessions (resets daily).
- **Sequence Number (8 Bytes, Uint64)**: Monotonically increasing counter per message.
- **Message Count (2 Bytes)**: Number of packed ticks in this UDP frame (for micro-batching under load).

### 5.2 The Retransmission Backchannel (TCP NACK Request)
When a processor detects a missing sequence range, it fires an immediate binary NACK over an established, persistent TCP connection to the loader's dedicated **Retransmission Server**:

```protobuf
message RetransmissionRequest {
  string session_id = 1;
  uint32 exchange_id = 2;
  uint16 stream_id = 3;
  uint64 start_sequence_num = 4;
  uint32 message_count = 5;      // Max 10,000 per NACK chunk
}
```

---

## 6. Step 4: High-Level Architecture (HLD)

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 EXTERNAL EXCHANGES (Direct Optical Cross-Connects)                     │
│                                                                                                        │
│   [ NASDAQ ITCH Feed ]             [ NIFTY Multicast Feed ]             [ S&P 500 CME Feed ]           │
└───────────────┬─────────────────────────────────┬───────────────────────────────────┬──────────────────┘
                │                                 │                                   │
                ▼                                 ▼                                   ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              UPSTREAM LOADER FLEET (Active-Active Pairs per Exchange)                  │
│                                                                                                        │
│   - Receives raw exchange protocols (TCP/Multicast)                                                    │
│   - Assigns Global Ingestion Sequence Numbers & Normalizes to SBE Binary Format                        │
│   - Dual-Transmits via UDP Multicast (Feed A on Port 9001, Feed B on Port 9002)                        │
│   - Runs In-Memory 60-Second Retransmission Ring Buffer (Serves TCP NACK Recovery)                     │
└───────────────┬─────────────────────────────────────────────────────────────────────┬──────────────────┘
                │                                                                     │
                ▼ (Primary UDP Feed A)                                                ▼ (Secondary UDP Feed B)
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                PROCESSOR SERVERS (Zero-Loss Deduplication Engine)                      │
│                                                                                                        │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ Network Card (Solarflare EF_VI / DPDK Kernel-Bypass Ring Buffer)                               │   │
│   └────────────────────────────────┬───────────────────────────────────────────────────────────────┘   │
│                                    ▼                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ LMAX Disruptor Lock-Free Circular Ring Buffer (10 Million Slots)                               │   │
│   └────────────────────────────────┬───────────────────────────────────────────────────────────────┘   │
│                                    ▼                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ Sequence Gap Detector, Bitmask Deduplicator & Sliding-Window Reorder Machine                   │   │
│   │ (Fires TCP NACK to Loader if gap detected; drops duplicates from Feed B)                       │   │
│   └────────────────────────────────┬───────────────────────────────────────────────────────────────┘   │
│                                    ▼                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ In-Memory Order Book Engine (Constructs BBO & L2 Depth per Symbol)                             │   │
│   └────────────────────────────────┬───────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┼───────────────────────────────────────────────────────────────────┘
                                     │
           ┌─────────────────────────┴───────────────────────────────────────┐
           │                                                                 │
           ▼ (Shared Memory IPC / Aeron UDP, < 100μs)                        ▼ (Internal Broadcast Bus, < 2ms)
┌────────────────────────────────────────┐                 ┌────────────────────────────────────────┐
│   Tier 1: HFT & Algorithmic Bots       │                 │   Tier 2: Retail WebSocket Fleet       │
│   - Direct Shared Memory Access        │                 │   - 100ms Tick Conflation Engine       │
│   - Raw, Unthrottled Tick Streams      │                 │   - Serves 1 Million Web/Mobile Traders│
│   - Dedicated Bare-Metal Co-location   │                 │   - Pub/Sub Symbol Filtering           │
└────────────────────────────────────────┘                 └────────────────────────────────────────┘
                                                                             │
                                                                             ▼ (Async Bulk Writes)
                                                           ┌────────────────────────────────────────┐
                                                           │   Historical TimescaleDB / ClickHouse   │
                                                           │   - Daily Compressed Parquet Storage   │
                                                           │   - Powers Interactive Charts & Backtest│
                                                           └────────────────────────────────────────┘
```


---

### 4.2 Step-by-Step Architectural Evolution (How to Drive the Interview Whiteboard)

In a FAANG system design interview, **never jump straight to kernel-bypass UDP and Disruptor buffers**. Walk the interviewer through the physics and network limits of standard messaging:

#### Phase 1: Naive Message Broker Baseline (TCP & Apache Kafka)
- **What to draw on the whiteboard**: `Exchange Loaders` ──► `Apache Kafka Cluster` ──► `Processor Servers`.
- **How to explain the flow**:
  - *"A typical distributed systems approach puts an Apache Kafka cluster between upstream market loaders and downstream processing nodes."*
- **Bottlenecks identified (Why it breaks in High-Frequency Trading)**:
  1. *Head-of-Line Blocking (TCP)*: TCP guarantees in-order delivery; if a single IP packet drops, the OS stalls all subsequent packets until retransmitted, causing latency spikes of 50–200ms.
  2. *Broker Overhead*: Kafka broker hops and page-cache flushes add 5–15ms of latency. Financial markets demand sub-millisecond execution.
  3. *Network Bandwidth Duplication*: Fanout over TCP requires sending the same 20M ticks/sec stream individually to each processor, saturating 100GbE network switches.

#### Phase 2: High-Frequency UDP Multicast with MoldUDP64 Framing
- **What to add next**: Replace TCP with **UDP Multicast** using the exchange-standard **MoldUDP64 Protocol**.
- **How to explain the transition**:
  - *"UDP Multicast sends 1 packet from the loader, and the physical network switch duplicates it in hardware to all 100 processor nodes simultaneously with single-digit microsecond latency. Each packet header carries an incremental 64-bit sequence number per market feed."*

#### Phase 3: Reliable Out-of-Band TCP NACK Retransmission
- **What to add next**: Add a **Loss Recovery NACK Channel over TCP**.
- **How to explain the transition**:
  - *"Because UDP is unreliable, processors track seen sequence numbers. If a processor receives sequence 104 after 101, it detects a gap (missing 102 and 103). It immediately sends an out-of-band NACK request over TCP to the Loader's 60-second in-memory ring buffer, which replays the missing ticks without stalling other nodes."*

#### Phase 4: Lock-Free In-Memory Processing (LMAX Disruptor Ring Buffer)
- **What to add next**: Deploy an **LMAX Disruptor Ring Buffer** inside each Processor node.
- **How to explain the transition**:
  - *"Standard Java queues (`LinkedBlockingQueue`) use mutex locks and generate GC garbage, causing 10ms GC pauses. LMAX Disruptor uses a pre-allocated circular ring buffer with lock-free atomic CAS operations and cacheline padding, achieving 20M events/sec with sub-microsecond latency."*

#### Phase 5: Kernel-Bypass Networking & Dual-Feed Deduplication
- **What to add next**: Add **Kernel-Bypass NICs (Solarflare EF_VI / DPDK)** and dual redundant exchange feeds (Feed A + Feed B).
- **How to explain the transition**:
  - *"To bypass the Linux OS network stack overhead (saving 25$\mu$s per packet), NIC drivers write packets directly into user-space application memory. Processors consume redundant Feed A and Feed B simultaneously, using an atomic sequence bitset to discard duplicates in $< 100	ext{ns}$."*

#### Phase 6: Tiered Fanout (Aeron IPC vs Conflated WebSockets)
- **What to add next**: Split distribution into **Institutional Ultra-Fast Path** and **Retail Web Dashboard Path**.
- **How to explain the transition**:
  - *"Trading algorithms receive raw ticks via Aeron IPC shared memory ($< 50\mu	ext{s}$). For retail web users who cannot process 20M ticks/sec, a conflation worker merges ticks into 100ms OHLC snapshots and fans them out over WebSockets."*

#### Phase 7: High-Volume Historical Persistence (ClickHouse)
- **What to add next**: Stream processed market ticks into a columnar **ClickHouse Cluster**.
- **How to explain the transition**:
  - *"To store 57 Billion daily ticks for quantitative backtesting, an asynchronous batch writer streams ticks into ClickHouse. Columnar partitioning by `(symbol, date)` with Zstandard compression achieves $10	imes$ storage reduction with instant analytical queries."*

---
---

## 7. Step 5: Low-Level Design & Deep Dives

### 7.1 The Triad Challenge: Solving Packet Loss, Duplicates, and Out-of-Order Delivery

Because UDP does not guarantee packet delivery, processors must enforce reliability in software. We solve this using a **Dual-Feed Sliding Reorder Machine**:

```
[ Loader 1 (Feed A) ] ──(UDP)──► [ Packet Arrival ]
                                        │
[ Loader 2 (Feed B) ] ──(UDP)──►        ▼
                            ┌──────────────────────────────────────┐
                            │ Step 1: Sequence Check & Deduplication│
                            │ Is SeqNum <= ExpectedSeq?             │
                            └──────────────────┬───────────────────┘
                                               │
                       ┌───────────────────────┴───────────────────────┐
                       ▼ (YES: Duplicate from Feed B)                  ▼ (NO: New Packet)
                [ Drop Immediately ]                           ┌───────────────────────────────┐
                                                               │ Step 2: In-Order Check        │
                                                               │ Is SeqNum == ExpectedSeq?     │
                                                               └───────────────┬───────────────┘
                                                                               │
                                       ┌───────────────────────────────────────┴───────────────────────┐
                                       ▼ (YES: Exact Next Tick)                                        ▼ (NO: SeqNum > Expected)
                         ┌─────────────────────────────┐                                 ┌─────────────────────────────┐
                         │ Process Tick Immediately    │                                 │ Out-of-Order / Packet Loss! │
                         │ ExpectedSeq = ExpectedSeq+1 │                                 │ Store in Reorder Ring Buffer│
                         └─────────────────────────────┘                                 │ Start 2ms NACK Timer        │
                                                                                         └──────────────┬──────────────┘
                                                                                                        │
                                                                            ┌───────────────────────────┴───────────────────────────┐
                                                                            ▼ (Missing Packet Arrives via B)                        ▼ (Timer Expires without Packet)
                                                                 [ Insert into Buffer Gap ]                              [ Fire TCP NACK to Loader ]
                                                                 [ Drain Contiguous Chain ]                              [ Loader Streams Missing Block ]
```

#### 1. Deduplication (Feed A vs Feed B)
- To achieve ultra-high availability, loaders transmit identical data across two independent network routes: **Feed A** and **Feed B** (standard financial practice, e.g., NASDAQ dual feed).
- The processor maintains a 64-bit integer: `last_processed_sequence_num`.
- If packet arrives with $\text{SeqNum} \le \text{last\_processed\_sequence\_num}$, it is dropped instantly in zero clock cycles via hardware pointer comparison.
- To prevent memory leaks, a **Sliding 10,000-bit Bitmask** tracks out-of-order received sequence numbers within the active window.

#### 2. Out-of-Order Reordering
- If the processor expects sequence `1001`, but receives `1003` from Feed A (Feed B delayed):
  1. The packet `1003` is **not dropped**; it is placed into an in-memory **Circular Index Array** at slot `1003 % 65536`.
  2. The processor marks slot `1002` as `MISSING`.
  3. It starts an ultra-short **2-millisecond jitter wait timer**. In 95% of cases, Feed B delivers `1002` within microseconds!
  4. Once `1002` arrives, the processor processes `1002`, immediately drains `1003` from the buffer, advances `last_processed_sequence_num = 1003`, and resets the pipeline.

#### 3. Packet Loss & The TCP NACK Engine
- If the 2ms jitter timer expires and `1002` has still not arrived from either Feed A or Feed B:
  1. The processor immediately constructs a `RetransmissionRequest` for gap `[1002, 1002]`.
  2. The NACK is fired over a dedicated TCP socket directly to the Upstream Loader’s Retransmission Buffer.
  3. The Loader fetches packet `1002` from its in-memory 60-second circular replay cache and streams it over TCP.
  4. The processor inserts the recovered tick, drains the ring buffer, and resumes live processing.

---

### 7.2 Microsecond-Level Architecture: Zero-Copy & The LMAX Disruptor

At 20 Million ticks/second, traditional software design patterns break:
- **Thread Context Switching**: Wakes up OS schedulers, costing 2–5 microseconds per switch.
- **Java Garbage Collection (GC)**: Allocating 20M objects/sec triggers GC pauses lasting 50–500ms, blowing through SLAs.
- **Mutex / Synchronized Queues**: Thread contention on locks degrades throughput by 90%.

#### The Zero-Copy Solution:
1. **Kernel Bypass Networking (DPDK / Solarflare EF_VI)**:
   - Packets bypass the standard Linux kernel network stack entirely.
   - The Network Interface Card (NIC) writes UDP frames directly into user-space RAM via Direct Memory Access (DMA), eliminating CPU context switches and buffer copies.
2. **Pre-Allocated LMAX Disruptor Ring Buffer**:
   - A single contiguous array of 10 Million pre-allocated `ExchangeTick` structs allocated at application startup.
   - Zero dynamic memory allocation (`malloc`/`new`) during live trading.
   - Utilizes a single writer pointer and multiple lock-free reader pointers using atomic CPU memory barriers (`volatile` CAS), processing **50 Million operations/second per core**.
3. **CPU Core Pinning & Isolation (`taskset` / `isolcpus`)**:
   - Processor worker threads are pinned to isolated physical CPU cores with hyperthreading disabled, guaranteeing dedicated L1/L2 cache locality with zero OS preemption.

---

### 7.3 Multi-Tier Downstream Distribution (HFT vs. Retail Conflation)

Different consumer groups have radically opposing latency and bandwidth requirements:

```
+--------------------------------------------------------------------------------------------------------------------+
|                                     DOWNSTREAM CONSUMER SEGREGATION MATRIX                                         |
+-------------------+----------------------------+-----------------------+-------------------------------------------+
| Consumer Tier     | Transport Mechanism        | Update Frequency      | Latency SLA & Data Completeness           |
+-------------------+----------------------------+-----------------------+-------------------------------------------+
| Tier 1: HFT Bots  | Shared Memory IPC /        | Real-Time (Tick by    | < 50 Microseconds; 100% full tick-by-tick |
| & Algorithmic Desk| Kernel-Bypass UDP          | Tick, 0ms delay)      | stream without conflation.                |
+-------------------+----------------------------+-----------------------+-------------------------------------------+
| Tier 2: Retail    | Multiplexed WebSockets     | Conflated (100ms      | < 50 Milliseconds; Merges micro-ticks into|
| Mobile & Web Users| over TLS (HTTP/2)          | sliding window)       | latest quote snapshots.                   |
+-------------------+----------------------------+-----------------------+-------------------------------------------+
| Tier 3: Analytics | Apache Kafka -> ClickHouse | Batched Micro-chunks  | < 5 Seconds; Columnar compressed storage   |
| & Charting Engine | Bulk Table Writer          | (1-second intervals)  | for candlestick and historical graphs.    |
+-------------------+----------------------------+-----------------------+-------------------------------------------+
```

#### How the 100ms Retail Conflation Engine Works:
- If a stock changes price 2,000 times in 100ms, pushing 2,000 frames over WebSockets to 1 Million mobile apps will saturate consumer home Wi-Fi and burn server bandwidth ($100\text{ Gbps+}$).
- The **Conflation Engine** maintains an in-memory hash table:
  `conflation_map[symbol_id] = {latest_price, high, low, volume, updated_flag}`.
- Ticks update this memory map in place ($O(1)$ mutation).
- Every 100ms, an asynchronous timer thread sweeps the map, collects all symbols where `updated_flag == true`, bundles them into a compact binary WebSocket frame, and pushes them to connected client gateways.
- Bandwidth drops by **98%**, while delivering an imperceptible, buttery-smooth 10 updates/second to user screens.

---

## 8. Common Interview Follow-Ups & Cross-Questions (Deep Dive)

### Q1: How do you prevent a "NACK Storm" when a network switch drops a UDP multicast packet across 50 processor nodes simultaneously?
- **The Core Issue**: If a top-of-rack switch drops packet #5000, all 50 processor instances detect the gap simultaneously and flood the Upstream Loader with 50 concurrent TCP NACK requests for the same missing packet, overwhelming the loader CPU and network buffers.
- **The Solution**:
  1. **Randomized Exponential Jitter**: Before firing a NACK, each processor waits $T_{\text{wait}} = \text{base} + \text{random}(0, \text{jitter\_window})$ (e.g., between 500μs and 2,000μs).
  2. **Multicast NACK Resends**: When the Upstream Loader receives the first NACK for packet #5000, it broadcasts the retransmitted packet back onto a dedicated **UDP Multicast Retransmission Channel**.
  3. **Snooping & Cancellation**: The other 49 processors listen to this channel; when they see packet #5000 re-broadcast, they immediately cancel their pending NACK requests before transmitting.

---

### Q2: How do you handle clock skew and timestamp ordering across different global exchanges?
- **The Core Issue**: A tick from NASDAQ (New York) and a tick from NIFTY (Mumbai) have different local timestamps. Server clocks drift by milliseconds if relying on standard NTP.
- **The Solution**:
  1. **PTP (Precision Time Protocol - IEEE 1588)**: Financial datacenters replace NTP with hardware PTP synchronized with GPS satellite clocks, achieving **sub-microsecond time synchronization** across all loaders and processors.
  2. **Dual-Timestamp Tracking**: Every record maintains two explicit timestamps:
     - `exchange_ts_ns`: Clock assigned by the exchange matching engine at trade execution (used for financial auditing).
     - `ingest_ts_ns`: Hardware NIC timestamp assigned upon packet arrival at our loader (used for internal latency monitoring).
  3. **Strict Sequence Monotonicity**: Processing ordering is **strictly governed by `sequence_number` per exchange channel**, never by wall-clock timestamps.

---

### Q3: What happens if a processor falls behind and its in-memory ring buffer overflows?
- **The Core Issue**: During an extreme market flash crash, a processor instance experiences a garbage collection pause or hardware thread stall. By the time it resumes, 500,000 packets have passed, exceeding the ring buffer size.
- **The Solution (The "Poisoned Buffer" Snapshot Recovery Protocol)**:
  1. **Self-Eviction**: When `incoming_seq - last_processed_seq > ring_buffer_capacity`, the processor detects a **Catastrophic Gap Breach**. It stops firing NACKs (requesting 500k packets via NACK would stall the loader).
  2. **State Drop & Reconnect**: The processor disconnects from the live UDP feed, flags its status as `DEGRADED`, and connects to the **Snapshot Recovery Service** over TCP.
  3. **Book Sync**: It downloads a full compressed Level 2 Order Book snapshot at sequence number $S$ (e.g., $S = 2,500,000$).
  4. **Delta Catch-Up**: It buffers incoming live UDP ticks with sequence $> S$, applies the delta changes on top of the snapshot, verifies state consistency, and rejoins the live distribution cluster in $< 1\text{ second}$.

---

### Q4: Why use UDP Multicast instead of Kafka for inter-service communication between Loaders and Processors?
- **The Core Issue**: Candidates often propose Kafka for every distributed messaging problem.
- **The Solution**:
  - **Kafka Latency**: Standard Kafka commitlog roundtrip through disk serialization, network sockets, and ZooKeeper/KRaft coordination adds **5 to 20 milliseconds** of latency. In high-frequency finance, 5ms is an eternity.
  - **UDP Multicast Latency**: Hardware-accelerated UDP multicast operates at physical wire speed with **$< 5 \text{ microseconds}$** latency.
  - **1-to-N Duplication**: With UDP Multicast, the Loader sends **one single physical packet** onto the network wire; network switch ASICs duplicate the packet at the hardware layer to all 50 processors simultaneously with zero CPU load on the sender.

---

### Q5: How do you prevent memory leaks when tracking seen sequence numbers for deduplication?
- **The Core Issue**: Storing millions of 64-bit integer sequence IDs in a standard hash set will consume gigabytes of memory and trigger garbage collection pauses.
- **The Solution**:
  - **Sliding Bitmask Array**: Since sequence numbers are monotonically increasing, we do not need to store every individual number.
  - We maintain a single `uint64_t base_sequence` and a **fixed-size bitmask ring of 65,536 bits (8 KB of RAM)**.
  - Bit position $P = (\text{SeqNum} - \text{base\_sequence})$ is flipped to `1` when received.
  - As `base_sequence` advances, bits are shifted out. Any packet arriving with $\text{SeqNum} < \text{base\_sequence}$ is instantly rejected as a historical duplicate.
  - Memory consumption is constant at **8 Kilobytes**, with zero allocations during runtime.

---

### Q6: How do you serve trading bots that demand raw ticks while simultaneously supporting retail web users on the same infrastructure?
- **The Core Issue**: Algorithmic bots need raw tick updates within 50 microseconds; retail browsers need conflated snapshots every 100ms. Mixing their delivery pathways slows down the bots.
- **The Solution (Physical Pathway Isolation)**:
  - **Trading Bots (Tier 1)**: Co-located on the same local physical network. They read directly from the processor's output via **Kernel-Bypass UDP Multicast** or **Shared Memory IPC (Inter-Process Communication via `/dev/shm` ring buffers)**, achieving sub-100 microsecond delivery.
  - **Retail Web/Mobile Users (Tier 2)**: Isolated behind a dedicated fleet of **Conflation Gateway Nodes**. The gateways subscribe to the processors, conflate the ticks to 10 updates/sec, and stream them over WebSockets. A slow or disconnected retail mobile app has zero impact on institutional bot pipelines.

---

### Q7: What happens if an Upstream Loader crashes during peak trading hours?
- **The Core Issue**: If the loader reading the direct exchange cross-connect dies, does the entire downstream processor cluster stall?
- **The Solution (Active-Active Hot Standby)**:
  - Every exchange connection is deployed with **Dual Active-Active Loaders (Loader A and Loader B)** on physically separate network switches and power supplies.
  - Both loaders ingest the raw exchange feed simultaneously and broadcast onto independent multicast groups (Port 9001 and Port 9002).
  - Processors read both streams concurrently. If Loader A drops or crashes, the processor seamlessly consumes identical packets from Loader B with **zero microsecond failover time and zero dropped packets**.

---

### Q8: How do you persist 57 Billion daily ticks for historical analysis without slowing down live processing?
- **The Core Issue**: Writing 20 Million ticks/second to a database will saturate disk I/O controllers.
- **The Solution**:
  - Live processors **never write to disk**.
  - A dedicated **Persistence Consumer Node** listens to the UDP stream as an independent passive observer.
  - The persistence worker buffers incoming ticks in a 1-second memory queue, converts them into columnar **Apache Parquet format**, and streams bulk blocks directly to **ClickHouse / TimescaleDB** and **AWS S3**.
  - Query workloads (chart rendering, backtesting) are completely air-gapped from live trading memory paths.

---

## 9. Architectural Trade-Off Analysis Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Ingestion Protocol** | UDP Multicast (Feed A + Feed B) | TCP / HTTP REST Streaming | Eliminates TCP head-of-line blocking; hardware switch ASIC duplicates packets to 50 nodes in $< 5\mu\text{s}$. |
| **Reliability Model** | NACK-based Selective Retransmission | ACK-based End-to-End Handshake | ACKs generate 20M reverse packets/sec, saturating network; NACK transmits only on detected missing gaps. |
| **Memory Concurrency** | LMAX Disruptor Lock-Free Ring Buffer | Java `BlockingQueue` / Mutex Locks | Mutex locks waste 2–5μs per thread context switch; Disruptor processes 50M ops/sec using lock-free CAS primitives. |
| **Network Stack** | Kernel-Bypass (Solarflare EF_VI / DPDK) | Standard Linux POSIX Sockets | Bypasses kernel packet parsing and memory copies; drops packet delivery latency from 25μs to $< 2\mu\text{s}$. |
| **Retail Distribution** | Conflation Engine (100ms Sliding Window) | Raw WebSocket Tick Streaming | Raw streaming to 1M users consumes 100+ Gbps bandwidth; 100ms conflation slashes traffic by 98% with zero visual difference. |
| **Deduplication Strategy** | Sliding Bitmask Window (8 KB RAM) | Hash Set of Seen Sequence IDs | Hash sets require dynamic memory allocation and trigger GC pauses; fixed bitmask operates in $O(1)$ with zero allocations. |
| **Historical Storage** | Air-Gapped ClickHouse Passive Consumer | Synchronous DB Write from Processor | Keeps live memory trading pipelines isolated; disk I/O never interferes with real-time distribution. |

