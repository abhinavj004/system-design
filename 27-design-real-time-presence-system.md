# System Design 27: Real-Time Presence System (Online/Offline Status & Last Seen)

> **Topic**: Designing a Massively Scalable Real-Time Presence Platform for High-Scale Messaging (WhatsApp, Slack, Discord, Teams)  
> **Scale**: 500 Million DAU, 50 Million Concurrent Connected WebSockets, Sub-Second Presence Propagation, Distributed "Last Seen" State Engine

---

## 1. Architectural Pattern Lineage & Transferred Learnings

To design this system at staff-level depth, we synthesize architectural patterns proven across our previous master designs:

```
┌───────────────────────────────────────┬───────────────────────────────────────────┬──────────────────────────────────────────┐
│ Subsystem Challenge in Presence       │ Transferred Architectural Pattern         │ Foundational Lineage in Our Master Notes │
├───────────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────┤
│ Real-Time Bi-directional Connection   │ Stateful WebSocket Gateways & Epoll       │ Episode 08 (WhatsApp) & Ep 10 (Docs)     │
│ Heartbeat Ingestion at 1.67M QPS      │ Ephemeral Sliding Window TTL in Memory    │ Episode 11 (Uber Driver GPS Tracking)    │
│ Presence State Flapping / Churn       │ Debounce Buffer & Grace Period Machine    │ Episode 21 (Distributed Rate Limiter)    │
│ The N x M Fanout Broadcast Explosion  │ Dynamic Viewport / Conversation Subscriptions│ Episode 13 (Instagram Hybrid Feed)     │
│ Ephemeral vs Durable State Split      │ Dual Storage (Redis Cache + Cassandra DB) │ Episode 02 (BookMyShow Dual Storage)     │
│ Asynchronous Last Seen Archival       │ Event-Driven Commitlog / Kafka Buffering  │ Episode 16 (Distributed Logging / Splunk)│
│ Channel-Wide Celebrity Hotspots       │ Threshold-based Push-to-Pull Transition   │ Episode 13 (Celebrity Hotspot Handling)  │
└───────────────────────────────────────┴───────────────────────────────────────────┴──────────────────────────────────────────┘
```

---

## 2. Step 1: Requirements Gathering & Estimations

### 2.1 Functional Requirements (FR)
1. **Real-Time Presence Tracking**: When a user opens the app or establishes a network session, their status transitions to `ONLINE` within $< 1\text{ second}$.
2. **Instant Disconnect Detection**: When a user closes the app, kills the process, or loses internet connectivity, their status transitions to `OFFLINE` (after a calibrated grace period).
3. **Last Seen Timestamps**: For any offline user, the system accurately displays their most recent active timestamp (e.g., *"Last seen today at 10:42 PM"*).
4. **Targeted Presence Delivery**: Users must view the real-time presence of their active 1:1 chat contacts and users in visible chat screens.
5. **Privacy Controls**: Users can configure visibility settings (`Everyone`, `My Contacts`, `Nobody`).

### 2.2 Non-Functional Requirements (NFR)
1. **Ultra-Low Propagation Latency**: Status changes must be delivered to interested active clients within $< 500\text{ms}$.
2. **High Concurrency & Scalability**: Support **50 Million concurrent persistent WebSocket connections** without resource starvation.
3. **High Availability & Fault Tolerance (AP over CP)**: A momentary delay or slight inconsistency in presence (e.g., seeing someone online for 5 extra seconds) is entirely acceptable. Network partitions must never crash chat delivery.
4. **Network & Battery Efficiency**: Mobile devices must not drain battery or exceed cellular data caps due to aggressive heartbeat loops.
5. **No Cascading Presence Storms**: Transient network drops (e.g., driving through a tunnel, switching between 5G and Wi-Fi) must not trigger full-mesh broadcast waves across millions of users.

---

### 2.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Traffic & Connection Math
- **Total Registered Users**: 1 Billion users.
- **Daily Active Users (DAU)**: 500 Million users.
- **Peak Concurrent Online Users ($C$)**: $10\% \text{ of DAU} = \mathbf{50 \text{ Million concurrent users}}$.
- **Heartbeat Interval ($T_{\text{hb}}$)**: 30 seconds.
- **Heartbeat Inbound Ingestion QPS**:
  $$\text{Heartbeat QPS} = \frac{50,000,000 \text{ connections}}{30 \text{ seconds}} \approx \mathbf{1,666,667 \text{ req/sec (1.67 Million QPS)}}$$
  > **Crucial Takeaway**: 1.67 Million writes/sec will instantly destroy any relational database. Heartbeat updates **must never write directly to persistent disk storage**; they must reside entirely in distributed memory (Redis / Cluster In-Memory Bitmaps).

#### Fanout Math: The Danger of Naive Broadcasting
- Suppose each user has an average of 200 contacts.
- If we broadcast every status change to all contacts:
  - If 500,000 users connect/disconnect every minute:
    $$\text{Outbound Fanout Messages} = 500,000 \times 200 = 100,000,000 \text{ messages/min} \approx \mathbf{1.67 \text{ Million msg/sec}}$$
- In large enterprise channels (e.g., Slack `#general` with 50,000 users or Discord guild with 100,000 users):
  - 1 user toggling status $\to 100,000$ WebSocket push messages.
  - 100 users toggling $\to \mathbf{10 \text{ Million push frames}}$!
  - **Verdict**: Unrestricted push is mathematically impossible. We **must** use **Dynamic Viewport Subscriptions (Fanout on Active Window Only)**.

#### Memory & Storage Estimations
1. **Ephemeral Presence Cache (Redis)**:
   - Key: `presence:{user_id}` $\to$ Value: `{status: 1, last_active: timestamp, gateway_id: "gw-102"}`
   - Record size: $\approx 64 \text{ bytes}$.
   - Total RAM for 50 Million online users:
     $$\text{RAM} = 50,000,000 \times 64 \text{ bytes} \approx 3.2 \text{ GB}$$
     *(A tiny footprint! Easily fits in a 3-node Redis cluster with master-replica redundancy).*
2. **Durable Last-Seen Database (Cassandra / ScyllaDB)**:
   - Table stores: `user_id` (16 bytes UUID), `last_seen_at` (8 bytes), `device_type` (1 byte), `privacy_flags` (1 byte).
   - Record size: $\approx 50 \text{ bytes}$ (with overhead $\approx 100 \text{ bytes}$).
   - 1 Billion registered users:
     $$\text{Total Storage} = 1,000,000,000 \times 100 \text{ bytes} \approx \mathbf{100 \text{ GB}}$$
     *(Easily stored across a small 3-node Cassandra cluster).*

---

## 3. Step 2: Identification of Core Entities

```
+-----------------------------------+             +-----------------------------------+
|       UserSession (Memory)        |             |      PresenceState (Ephemeral)   |
+-----------------------------------+             +-----------------------------------+
| session_id       : UUID           |             | user_id          : UUID (PK)      |
| user_id          : UUID (FK)      | 1         1 | status           : ENUM(ON/OFF)   |
| gateway_id       : VARCHAR(32)    |<----------->| last_heartbeat   : TIMESTAMP      |
| client_version   : VARCHAR(16)    |             | device_platform  : VARCHAR(10)    |
| connected_at     : TIMESTAMP      |             | ttl              : EXPIRE (60s)   |
+-----------------------------------+             +-----------------------------------+
                  │                                                 │
                  │ 1                                               │ 1
                  ▼ *                                               ▼ 1
+-----------------------------------+             +-----------------------------------+
|     ViewportSubscription (Memory) |             |    UserLastSeen (Durable DB)      |
+-----------------------------------+             +-----------------------------------+
| subscriber_id    : UUID           |             | user_id          : UUID (PK)      |
| target_user_id   : UUID           |             | last_seen_at     : TIMESTAMP      |
| context_type     : ENUM(DM, GROUP)|             | privacy_setting  : VARCHAR(16)    |
| created_at       : TIMESTAMP      |             | updated_at       : TIMESTAMP      |
+-----------------------------------+             +-----------------------------------+
```

---

## 4. Step 3: API & Protocol Interface Design

### 4.1 Transport Protocols: Trade-Off Analysis (Polling vs. SSE vs. WebSockets)

The interview prompt explicitly asks to evaluate transport mechanisms for presence:

```
+--------------------------------------------------------------------------------------------------------------------+
|                                      TRANSPORT PROTOCOL COMPARATIVE MATRIX                                         |
+------------------+-----------------------------+-----------------------------+-------------------------------------+
| Protocol         | Mechanism                   | Pros                        | Cons (Why it fails / works)         |
+------------------+-----------------------------+-----------------------------+-------------------------------------+
| Short Polling    | Client sends HTTP GET every | Simple; stateless; works    | Disastrous at scale. 50M users      |
|                  | 5–10 seconds.               | through all firewalls/CDNs. | polling every 5s = 10M QPS of HTTP  |
|                  |                             |                             | headers; destroys mobile battery.   |
+------------------+-----------------------------+-----------------------------+-------------------------------------+
| Server-Sent      | Persistent unidirectional   | Low overhead; native HTTP/2 | Strictly unidirectional (server ->  |
| Events (SSE)     | HTTP text stream.           | multiplexing; auto-reconnect| client). Requires separate HTTP POST|
|                  |                             | built into browsers.        | for heartbeats (doubling sockets).  |
+------------------+-----------------------------+-----------------------------+-------------------------------------+
| WebSockets       | Full-duplex persistent TCP  | Ultra-low framing (2 bytes);| Stateful connections; requires      |
| (Chosen Standard)| connection established via  | bi-directional heartbeats   | connection managers and sticky L4/L7|
|                  | HTTP 101 Upgrade.           | & instant pushes on 1 pipe. | load balancers with TCP tuning.     |
+------------------+-----------------------------+-----------------------------+-------------------------------------+
```

> **Decision**: **WebSockets** is the standard architecture. Since the messaging application already maintains a persistent WebSocket for chat messaging, **presence piggybacks on the existing TCP connection without opening a second socket**, eliminating transport overhead entirely.

---

### 4.2 WebSocket Wire Protocol (Binary Protobuf Frames)

To conserve cellular data across 50 Million connections, text JSON is replaced with compact **Protocol Buffers**:

#### Frame 1: Client Heartbeat (Inbound every 30s)
```protobuf
message ClientHeartbeat {
  string user_id = 1;
  int64 client_timestamp = 2;
  repeated string active_conversation_user_ids = 3; // Viewport: who is on screen right now!
}
```

#### Frame 2: Server Presence Update (Outbound Push)
```protobuf
message PresencePushEvent {
  string target_user_id = 1;
  enum Status {
    OFFLINE = 0;
    ONLINE = 1;
    AWAY = 2;
  }
  Status status = 2;
  int64 last_seen_timestamp = 3; // Provided if OFFLINE
}
```

### 4.3 Public REST APIs

#### 1. Batch Query Presence (Fallback on App Boot / Opening Contact List)
```http
POST /api/v1/presence/batch-query
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request:
{
  "user_ids": ["usr_8812", "usr_9934", "usr_1029"]
}

Response: 200 OK
{
  "statuses": {
    "usr_8812": { "status": "ONLINE", "last_seen": null },
    "usr_9934": { "status": "OFFLINE", "last_seen": 1729291200 },
    "usr_1029": { "status": "OFFLINE", "last_seen": null } // Hidden by user privacy setting
  }
}
```

---

## 5. Step 4: High-Level Architecture (HLD)

```
[ Mobile / Web Clients ]
       │  ▲
       │  │ (Persistent WebSockets: Heartbeats & Presence Frames)
       ▼  │
┌────────────────────────────────────────────────────────────────────────┐
│                   L4 Load Balancers (AWS NLB / Maglev)                 │
│                 (Consistent Hashing / TCP Pass-Through)                │
└────────────────────────────────────────────────────────────────────────┘
       │  ▲
       ▼  │
┌────────────────────────────────────────────────────────────────────────┐
│               WebSocket Gateway Fleet (Envoy / Go / Netty)             │
│   - Terminates TLS/TCP (50k conns/node, ~1,000 servers)                │
│   - Holds In-Memory Connection Map: user_id -> SocketDescriptor        │
│   - Tracks Local Client Viewport Subscriptions                         │
└────────────────────────────────────────────────────────────────────────┘
       │                              ▲
       │ (1) Ingest Heartbeats        │ (6) Dispatched Push Frames
       ▼ (1.67M QPS)                  │
┌────────────────────────────────────────────────────────────────────────┐
│                     Presence Service Cluster                           │
│   - Evaluates Heartbeats & Enforces Debounce Grace Periods             │
│   - In-Memory State Verification via Redis                             │
└────────────────────────────────────────────────────────────────────────┘
       │                                     ▲
       │ (2) Read/Write                      │ (5) Delivers Presence Diff
       ▼                                     │
┌───────────────────────────────────┐        │
│    Redis Presence Cluster         │        │
│ - presence:{uid} -> State (TTL 60s)│        │
│ - user_subs:{uid} -> Set(Followers)│        │
└───────────────────────────────────┘        │
       │                                     │
       │ (3) State Transitions Only          │
       ▼ (Online -> Offline)                 │
┌────────────────────────────────────────────────────────────────────────┐
│                   Message Broker (Apache Kafka)                        │
│             Topic: 'presence-state-transitions'                        │
└────────────────────────────────────────────────────────────────────────┘
       │                                     │
       ├─────────────────────────────────────┘
       │ (4) Fanout Worker Pipeline
       ▼
┌───────────────────────────────────┐         ┌───────────────────────────┐
│     Last-Seen Persistence Worker  │         │  User Privacy Service     │
│       (Batched Async Writes)      │         │   (Filters Blacklists &   │
└───────────────────────────────────┘         │      Contacts Only)       │
                  │                           └───────────────────────────┘
                  ▼
┌───────────────────────────────────┐
│     Cassandra / ScyllaDB          │
│   (Durable Last Seen Records)     │
└───────────────────────────────────┘
```

---

## 6. Step 5: Low-Level Design & Deep Dives

### 6.1 The Heartbeat & Debounce Grace Period State Machine

A critical flaw in naive systems is **Presence Flapping**:
- When a user enters an elevator, passes through a cellular dead zone, or toggles between Wi-Fi and 5G, the TCP socket disconnects and reconnects within 2–5 seconds.
- If the system immediately marks the user `OFFLINE` and notifies all friends, and 3 seconds later marks them `ONLINE` and notifies everyone again, the network experiences a massive **broadcast storm**.

#### The Debounce Grace Period Machine
```
   [ Client Connects ] ──► (1) Socket Established ──► Mark ONLINE immediately in Redis
                                                      Emit Kafka Event: STATUS_ONLINE
                                                      
   [ Client Disconnects ] OR [ Heartbeat Missed ]
             │
             ▼
   (2) DO NOT Mark Offline Immediately!
       Start Grace Period Timer (60 seconds) in Redis:
       SET pending_offline:{user_id} 1 EX 60
             │
             ├─────────────────────────────────────────┐
             │ (If Reconnects within 60s)              │ (If 60s Expires with No Heartbeat)
             ▼                                         ▼
   Cancel Grace Period Timer                 (3) Finalize OFFLINE State:
   Socket re-associated with Gateway             - Set presence:{user_id} = OFFLINE
   Zero presence events emitted!                 - Record current timestamp as last_seen
   (Saved 100% downstream fanout)               - Emit Kafka Event: STATUS_OFFLINE
```

- **Redis Key Lifespans**:
  - `presence:{user_id}`: Stored with a strict **TTL of 60 seconds**.
  - Every incoming heartbeat executes:
    ```redis
    SET presence:usr_4492 '{"status":"ONLINE","platform":"iOS"}' EX 60
    ```
  - If a server crashes, keys naturally expire without needing a distributed cluster scan.

---

### 6.2 Solving the $N \times M$ Fanout Problem: Dynamic Viewport Subscriptions

In messaging applications, user relationships are vast, but **user attention is tiny**.
- A user may have 500 contacts, but only has **1 active chat open on their screen**.
- Pushing presence for all 500 contacts every 30 seconds wastes gigabytes of cellular bandwidth.

#### Dynamic Viewport Architecture:
1. **Active Viewport Registration**:
   - When Alice opens WhatsApp and enters a chat with Bob, the client sends a lightweight frame:
     ```protobuf
     SubscribePresence { target_user_ids: ["usr_Bob"] }
     ```
   - The WebSocket Gateway registers Alice’s connection ID under Bob’s in-memory subscriber list on the Gateway node:
     ```
     gateway_sub_map: usr_Bob -> [ Alice_Socket_FD_42 ]
     ```
2. **Reverse Gateway Routing**:
   - The Gateway registers with Redis Pub/Sub: *"This Gateway instance is interested in updates for `usr_Bob`"*.
   - When Bob’s status changes to `OFFLINE`, a single message is published to the Redis channel `presence:usr_Bob`.
   - Only the specific Gateway holding Alice’s connection receives the event and pushes it to Alice.
3. **Leaving the Screen**:
   - When Alice navigates back to her main inbox list, the client unbinds Bob (`UnsubscribePresence`) and polls batch presence for the top 10 inbox threads once.

---

### 6.3 Handling Millions of WebSocket Connections on Gateway Nodes

Maintaining 50 Million simultaneous connections requires extreme OS and networking optimization:

1. **Linux Epoll & File Descriptor Limits**:
   - Each TCP socket consumes one file descriptor.
   - Configure `/etc/security/limits.conf`:
     ```bash
     * soft nofile 1048576
     * hard nofile 1048576
     ```
2. **Memory per Connection**:
   - In standard Linux, a default TCP socket allocates 16 KB read/write buffers. $50\text{M} \times 32\text{ KB} = 1.6 \text{ TB RAM}$.
   - We tune the TCP kernel window buffers:
     ```bash
     sysctl -w net.ipv4.tcp_rmem="1024 4096 16384"
     sysctl -w net.ipv4.tcp_wmem="1024 4096 16384"
     ```
   - Each socket memory footprint drops to $\approx 4\text{–}6\text{ KB}$.
3. **Gateway Node Sizing**:
   - A modern 64 GB RAM server running an asynchronous I/O engine (Go `epoll`, Netty, or Envoy) safely handles **50,000 to 100,000 active WebSocket connections**.
   - Total Gateway cluster size:
     $$\text{Servers Required} = \frac{50,000,000 \text{ connections}}{50,000 \text{ per node}} = \mathbf{1,000 \text{ Gateway instances}}$$

---

### 6.4 Durable "Last Seen" Storage Architecture (Cassandra / ScyllaDB)

Writing "Last Seen" timestamps directly to disk on every single heartbeat would saturate storage controllers ($1.67\text{M writes/sec}$). 

#### The Asynchronous Last-Seen Pipeline:
1. When a user transitions from `ONLINE` to `OFFLINE` (after the 60-second debounce period expires), the Presence Service emits a `UserWentOfflineEvent` to Kafka:
   ```json
   {
     "user_id": "usr_9918",
     "last_seen_at": 1729291240,
     "device": "Android"
   }
   ```
2. A fleet of **Last-Seen Persistence Workers** consumes from Kafka.
3. Workers batch updates in memory across a 2-second sliding window and execute bulk upserts into **Apache Cassandra**:
   ```sql
   CREATE KEYSPACE presence_store 
   WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east': 3, 'us-west': 3};

   CREATE TABLE presence_store.user_last_seen (
       user_id uuid,
       last_seen_at timestamp,
       device_type text,
       privacy_level text, -- 'PUBLIC', 'CONTACTS_ONLY', 'NOBODY'
       PRIMARY KEY (user_id)
   );
   ```
4. **Why Cassandra?**
   - Pure point lookups by partition key (`user_id`).
   - Constant-time $O(1)$ writes through append-only CommitLogs and SSTables.
   - Masterless replication ensures zero downtime during region failures.

---

## 7. Common Interview Follow-Ups & Cross-Questions (Deep Dive)

### Q1: How do you prevent presence flapping when mobile users switch between Wi-Fi and 5G?
- **The Core Issue**: Handover between cell towers or switching from Wi-Fi to 5G briefly terminates the active TCP socket. Without mitigation, this produces instant offline-online oscillation across friends' devices.
- **The Solution**: 
  - We decouple **Physical Socket Disconnection** from **Logical Offline State**.
  - When the TCP socket closes, the Gateway sets a **Grace Period Key** in Redis: `SET offline_grace:{user_id} true EX 45`.
  - If the client re-establishes a WebSocket connection within 45 seconds using its session token, the Gateway deletes the grace timer. The user remains `ONLINE` continuously from the perspective of external observers.
  - Only if the grace timer expires does a background sweeper publish the `OFFLINE` event.

---

### Q2: How do you solve the celebrity / large group presence fanout problem (e.g., 100,000 users in a Discord or Slack channel)?
- **The Core Issue**: If a channel contains 100,000 users, broadcasting presence updates for all members will trigger $100,000 \times N$ network frames, causing severe CPU spikes on gateways and clients.
- **The Solution (Threshold Hybrid Model)**:
  - **Small Channels ($< 100$ members)**: Standard push model via WebSocket subscriptions.
  - **Large Channels ($> 100$ members)**: 
    1. **Disable Global Push**: The server never broadcasts individual presence changes to the whole channel.
    2. **Viewport-Only Subscription**: The client only subscribes to the 20–30 user IDs currently visible in their member sidebar list. As the user scrolls, the client issues delta subscription frames (`Subscribe [ids...]`, `Unsubscribe [ids...]`).
    3. **Batch Query on Channel Entry**: When entering the channel, fetch a single bitmap of active members via REST.

---

### Q3: How do you scale to 50M concurrent WebSockets without running out of Linux file descriptors or memory?
- **The Core Issue**: The default Linux OS allocates large TCP buffers and caps open file descriptors at 1,024.
- **The Solution**:
  1. Increase `fs.file-max` and `nofile` to $1,048,576$.
  2. Reduce TCP buffer allocation (`net.ipv4.tcp_rmem` and `tcp_wmem`) from 16 KB defaults to 4 KB minimums, shrinking per-socket kernel RAM to $\approx 4\text{ KB}$.
  3. Deploy **1,000 stateless WebSocket Gateway instances**, each handling 50,000 connections.
  4. Use an event-driven non-blocking I/O runtime (Go with `epoll`/`kqueue` or Java Netty) to avoid thread-per-connection thread stack exhaustion ($1\text{ MB per thread} \times 50,000 = 50\text{ GB RAM}$, which would crash standard multi-threaded servers).

---

### Q4: How does the system scale if Redis crashes—is presence state permanently lost?
- **The Core Issue**: Redis stores the real-time presence cache. If nodes crash, what happens to the status of 50M users?
- **The Solution**:
  - **Presence is Inherently Ephemeral**: We do not require long-term persistence in Redis.
  - If a Redis shard fails, replica nodes promote immediately via Redis Sentinel / Cluster Raft consensus.
  - Even in a catastrophic total cache wipeout:
    1. Clients send heartbeats every 30 seconds.
    2. Within 30 seconds of Redis coming back up, active clients naturally self-heal and re-populate the cache via their regular heartbeat loop.
    3. The durable "Last Seen" database (Cassandra) remains untouched because it only records offline transitions.

---

### Q5: How do you implement privacy settings (e.g., "Hide Last Seen" or "Nobody can see my status") efficiently?
- **The Core Issue**: Evaluating privacy rules on every heartbeat or push frame introduces severe CPU latency.
- **The Solution**:
  - Privacy rules are evaluated at the **Edge Gateway / Ingress Filter**:
    1. A user's privacy preference (`EVERYONE`, `CONTACTS_ONLY`, `NOBODY`) is cached in Redis alongside user metadata.
    2. When Alice requests Bob’s status, the Gateway checks Bob’s privacy rule.
    3. If Bob has set `CONTACTS_ONLY`, the system checks a Redis Set `contacts:Bob` for Alice’s ID.
    4. If unauthorized, the Gateway strips the `last_seen` timestamp and returns status as `OFFLINE` (or masked), preventing the subscriber from knowing Bob is online.

---

### Q6: How do you minimize battery and data consumption on mobile clients sending heartbeats?
- **The Core Issue**: Waking up the mobile radio every 5 seconds drains battery life rapidly.
- **The Solution**:
  1. **Coalesced Heartbeats**: Piggyback heartbeats onto active outgoing chat messages. If the user sends a message, that message acts as an implicit heartbeat, resetting the 30-second timer.
  2. **Relaxed Background Interval**: When the mobile app is minimized (background state), stop WebSocket heartbeats entirely. Send a single `PAUSE` frame to mark the client away, or disconnect the socket and rely on Apple APNs / Google FCM push notifications for incoming messages.
  3. **Binary Framing**: Use 12-byte Protobuf payloads instead of 200-byte JSON strings.

---

### Q7: Why not use Kafka for real-time presence fanout instead of Redis Pub/Sub?
- **The Core Issue**: Candidates often propose Kafka for every pub/sub problem.
- **The Solution**:
  - **Kafka is an append-only, disk-backed persistent distributed log**: It is designed for durable event ordering, replayability, and high-throughput streaming. Creating 50 Million dynamic topics or partition-filtering ephemeral presence messages on disk will cause disk I/O bottlenecks.
  - **Redis Pub/Sub is fire-and-forget in-memory message routing**: It has zero disk serialization overhead, sub-millisecond dispatch latency, and consumes no storage when subscribers are disconnected.
  - **Division of Labor**: Redis Pub/Sub handles transient ephemeral fanout; Kafka handles durable state change events (`ONLINE` $\to$ `OFFLINE`) for database archival.

---

### Q8: What happens to last-seen timestamps during a major network partition or datacenter outage?
- **The Core Issue**: If Datacenter US-East is severed from US-West, split-brain can result in conflicting last-seen timestamps.
- **The Solution**:
  - We use **Last-Write-Wins (LWW) with Timestamp Ordering** in Cassandra.
  - When committing `last_seen_at`, records use client-validated server monotonic timestamps (`USING TIMESTAMP <epoch_microseconds>`).
  - During partition recovery, Cassandra naturally resolves conflicting records by keeping the higher timestamp, ensuring the true latest activity is preserved without data corruption.

---

## 8. Architectural Trade-Off Analysis Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Transport Protocol** | WebSockets (Multiplexed on Chat Connection) | HTTP Short Polling / SSE | Reuses existing chat TCP socket; eliminates HTTP header overhead; saves mobile battery life. |
| **Fanout Delivery Model** | Dynamic Viewport Subscriptions (Active Chats Only) | Broadcast to All Contacts ($N \times M$) | Prevents $167\text{M QPS}$ outbound fanout explosion; limits pushes strictly to what is visible on screen. |
| **Ephemeral Presence Cache** | Redis Cluster (In-Memory with 60s TTL) | Relational DB / Disk Tables | Sub-millisecond latency; automatic expiration cleans up dead connections without cron jobs. |
| **Durable Last-Seen Storage**| Apache Cassandra / ScyllaDB | Relational SQL (PostgreSQL) | Linear horizontal write scaling for hundreds of millions of users; $O(1)$ point lookups by `user_id`. |
| **Disconnect Resilience** | 45-Second Debounce Grace Machine | Instant Offline Notification | Prevents presence flapping and network storms caused by momentary 5G/Wi-Fi handovers. |
| **Wire Protocol Format** | Compact Protocol Buffers (Binary) | JSON over Text WebSocket | Shrinks heartbeat frame size by 85%, significantly reducing mobile data plan usage. |
| **Last-Seen Write Pipeline**| Async Kafka Buffer with 2s Batching | Synchronous DB Write on Disconnect | Decouples real-time presence engine from database write latency spikes. |
