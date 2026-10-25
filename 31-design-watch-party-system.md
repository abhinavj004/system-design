# System Design: Real-Time Watch Party System (Teleparty / Netflix Party / Amazon Prime Watch Party)

> **Source Reference**: [FAANG / Streaming Media System Design Interview Questions]  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* & Dodesimo clean standard).  
> **Topic**: Designing a Low-Level & High-Level Real-Time Virtual Room Architecture with Synchronized Video Playback, Network Drift Correction, and Low-Latency Control Broadcast.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: The host creates a room stored in an in-memory **Redis Cache** (`room:{id}`) and receives a unique 6-character code. Participants join via persistent **WebSockets**. The Watch Party server **does NOT stream the video** (video files stream independently from existing CDNs); it only synchronizes lightweight JSON control signals. When the host clicks *Play*, the server schedules a **Future Epoch Timestamp** (`current_time + max_latency + 200ms`) so all clients trigger playback at the exact same real-world millisecond. Ongoing network lag is managed via **Three-Tier Drift Correction**: minor drift ($<50\text{ms}$) is ignored, medium drift ($50\text{ms} - 1\text{s}$) is smoothly corrected by micro-adjusting playback speed ($0.95\times$ or $1.05\times$), and severe drift ($>1\text{s}$) triggers a hard seek.

| Metric / Dimension | Production Target / Value (Consumer Scale) |
| :--- | :--- |
| **Room Concurrency** | **1,000,000 active rooms** (avg 5–10 users/room; peak 1,000/room) |
| **Active WebSocket Connections** | **5,000,000 – 10,000,000 concurrent sockets** |
| **Sync Accuracy SLA** | **$< 50\text{ ms}$ playback drift** across all participants |
| **Control Signal Latency** | $< 100\text{ ms}$ broadcast fan-out via Redis Pub/Sub |
| **Video Data Responsibility** | **0% video traffic** (handled by Netflix/YouTube CDN) |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **The "Streaming Video Through WebSockets" Trap**:
   - *Trap*: *"How does your Watch Party server stream the 4K video frames to all participants?"*
   - *Defense*: The Watch Party server **never touches video chunks**. Video media streams directly from the streaming provider's CDN (e.g., Netflix, S3/CloudFront). Our server only coordinates lightweight JSON control events (`PLAY`, `PAUSE`, `SEEK`, `RATE_CHANGE`), consuming less than $1\text{ KB/sec}$ per user.
2. **The "Instant Event Broadcast" Drift Flaw**:
   - *Trap*: *"Why not just broadcast `PAUSE` to everyone the millisecond the host clicks pause?"*
   - *Defense*: If Alice has a 20ms ping and Bob has a 300ms ping on mobile, Bob receives the pause signal 280ms later and pauses at a later frame. Over a 2-hour movie, participants drift 10–15 seconds apart. Instead, we use **Future Scheduled Execution**: the server tells clients to execute the action at a precise future clock time (`server_epoch + 300ms`), allowing all clients to buffer and trigger simultaneously.
3. **The "One Slow User Pauses the Movie" Dilemma**:
   - *Trap*: *"If Bob's mobile network drops to 2G and starts buffering, should the entire room freeze for everyone?"*
   - *Defense*: No. Halting the room for 50 people because 1 person buffers ruins the experience. The room maintains a **Host-Authoritative Timeline**. Bob's player buffers locally; when his connection recovers, his player automatically catches up using temporary $1.1\times$ playback speed or a hard jump.

> **Golden Architectural Rule**: Never stream video through the sync server. Decouple high-bandwidth media delivery (CDN) from low-bandwidth timeline synchronization (WebSockets + Redis Pub/Sub).

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Create Room**: A host can create a watch party room and generate a short, shareable Room ID (e.g., `PARTY-7X9Q`).
2. **Join Room**: Multiple participants can join the room using the Room ID, automatically syncing to the current video timestamp.
3. **Synchronized Playback Controls**:
   - **Play**: Starts video playback for all participants at the same timestamp.
   - **Pause**: Pauses playback for all participants in real time.
   - **Seek**: Jumps playback to an exact timestamp (e.g., `00:45:10`) for everyone.
   - **Playback Speed**: Changing speed (e.g., $1.25\times$) applies across the entire room.
4. **Drift & Latency Compensation**: The system must detect when a participant falls out of sync due to network jitter and automatically align them without jarring video stutter.
5. **Host Privileges**: The host has master control over playback. (Optional toggle: allow any member to pause/play).

---

### 1.2 Non-Functional Requirements (NFR)
1. **Tight Playback Synchronization**: All participants must remain synchronized within **$< 50\text{ ms}$** of each other (virtually imperceptible to the human eye).
2. **Low Broadcast Latency**: Host actions (`PAUSE`, `SEEK`) must fan out to all room participants in **$< 100\text{ ms}$**.
3. **High Availability (99.99%)**: Room servers must handle node restarts without terminating watch party sessions.
4. **Graceful Degradation**: Participants with poor connections should not degrade or freeze playback for the rest of the room.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Concurrency & Connection Scale
- **Active Watch Party Rooms**: $1,000,000$ concurrent rooms globally.
- **Average Participants per Room**: 5 users $\implies 5,000,000$ active WebSocket connections.
- **Peak Room Size**: Up to 1,000 users for special live influencer watch parties.

#### Network Bandwidth Math (The Control Plane is Tiny!)
- A synchronization message (JSON) is small:
  ```json
  {"action": "SEEK", "timestamp_ms": 2715000, "scheduled_epoch_ms": 1761307200500}
  ```
  Payload size $\approx 200\text{ bytes}$.
- Users interact infrequently: average 1 action every 2 minutes ($0.008\text{ actions/sec/room}$).
- Periodic sync heartbeat: every client pings every 5 seconds ($0.2\text{ pings/sec/client}$).
- **Total Ingress QPS**:
  $$5,000,000\text{ users} \times 0.2\text{ pings/sec} = 1,000,000\text{ requests/sec}$$
- **Bandwidth Consumption**:
  $$1,000,000\text{ msgs/sec} \times 200\text{ bytes} \approx 200\text{ MB/sec} = 1.6\text{ Gbps}$$
- *Conclusion*: A modest cluster of 20 WebSocket gateway servers (each managing 250,000 open connections via Linux `epoll`) handles 100% of global traffic easily.

---

## 2. Step 2: Identification of Core Entities & Low-Level Data Modeling

```
┌─────────────────────────────────┐           1:N           ┌─────────────────────────────────┐
│              Room               │────────────────────────▶│           Participant           │
├─────────────────────────────────┤                         ├─────────────────────────────────┤
│ room_id (PK, String, 6 chars)   │                         │ participant_id (PK, String)     │
│ host_user_id (String)           │                         │ room_id (FK, String)            │
│ video_content_id (String)       │                         │ user_name (String)              │
│ state (PLAYING, PAUSED, SEEKING)│                         │ socket_connection_id (String)   │
│ current_timeline_ms (Long)      │                         │ estimated_rtt_ms (Integer)      │
│ playback_speed (Float)          │                         │ current_player_time_ms (Long)   │
│ last_updated_epoch_ms (Long)    │                         │ status (SYNCED, BUFFERING)      │
└─────────────────────────────────┘                         └─────────────────────────────────┘
```

### Room State Machine
A room transitions through clean playback states:

```
                  ┌───────────────┐
                  │ INITIALIZING  │
                  └───────┬───────┘
                          │ Host loads video
                          ▼
                  ┌───────────────┐
      ┌──────────▶│    PAUSED     │◀──────────┐
      │           └───────┬───────┘           │
      │ Host              │ Host              │ Host
      │ Pauses            │ Plays             │ Pauses
      │                   ▼                   │
      │           ┌───────────────┐           │
      └───────────│    PLAYING    │───────────┘
                  └───────┬───────┘
                          │ Host Seeks
                          ▼
                  ┌───────────────┐
                  │    SEEKING    │──▶ (Returns to PLAYING/PAUSED)
                  └───────────────┘
```

### In-Memory Storage Schema (Redis Hashes)
Rooms are ephemeral and write-heavy. We store live room state directly in **Redis**:
- **Key**: `room:{room_id}` (Redis Hash)
  - `host_id`: `"usr_alice"`
  - `video_id`: `"movie_inception_4k"`
  - `status`: `"PLAYING"`
  - `anchor_media_time_ms`: `1245000` *(time in movie when play started)*
  - `anchor_server_epoch_ms`: `1761307200000` *(real-world clock time when play started)*
  - `speed`: `1.0`
- **Current Position Calculation (Virtual Clock)**:
  At any given real-world millisecond `now_epoch`:
  $$\text{Current Movie Position} = \text{anchor\_media\_time\_ms} + ((\text{now\_epoch} - \text{anchor\_server\_epoch\_ms}) \times \text{speed})$$
  *(This allows anyone to calculate the exact current frame without the server constantly updating the database!)*

---

## 3. Step 3: API & WebSocket Protocol Design

### 3.1 REST API: Room Lifecycle Management
```http
POST /api/v1/rooms
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
  "video_content_id": "vid_netflix_99812",
  "controls_policy": "HOST_ONLY"
}

Response: 201 Created
{
  "room_id": "7X9Q2A",
  "share_url": "https://watchparty.app/join/7X9Q2A",
  "host_id": "usr_alice",
  "state": "PAUSED",
  "current_time_ms": 0
}
```

```http
GET /api/v1/rooms/7X9Q2A
Response: 200 OK
{
  "room_id": "7X9Q2A",
  "video_content_id": "vid_netflix_99812",
  "active_participants": 4,
  "status": "PLAYING",
  "current_time_ms": 341200
}
```

---

### 3.2 WebSocket Real-Time Control Protocol

Once connected via `wss://ws.watchparty.app/ws/rooms/{room_id}?token={jwt}`, messages are exchanged as compact JSON:

#### 1. Host Triggers Play / Pause / Seek (Client ──▶ Server)
```json
{
  "type": "COMMAND",
  "action": "PLAY",
  "media_time_ms": 1245000,
  "client_send_epoch_ms": 1761307200100
}
```

#### 2. Server Broadcasts Synchronized Action (Server ──▶ All Clients)
```json
{
  "type": "SYNC_ACTION",
  "action": "PLAY",
  "media_time_ms": 1245000,
  "scheduled_execution_epoch_ms": 1761307200400,
  "playback_speed": 1.0
}
```
*(Every client calculates `wait_time = scheduled_execution_epoch_ms - current_epoch` and triggers the player at the exact same millisecond).*

#### 3. Periodic Heartbeat & Time Sync (Client ──▶ Server ──▶ Client)
```json
// Client sends ping:
{ "type": "HEARTBEAT", "client_time_ms": 1761307200050 }

// Server immediately responds with server epoch timestamp:
{ 
  "type": "HEARTBEAT_ACK", 
  "client_time_ms": 1761307200050, 
  "server_receive_time_ms": 1761307200080,
  "server_transmit_time_ms": 1761307200085
}
```

---

## 4. Step 4: High-Level Architecture (HLD)

### Component Interaction Topology

```
[ Video Provider CDN (Netflix / S3 / CloudFront) ]
       │
       │ Direct 4K Video Stream (Chunked HLS / DASH)
       ▼
[ Client A (Host) ]                    [ Client B (Viewer) ]           [ Client C (Viewer) ]
       │                                       │                               │
       │ Persistent WebSocket                  │ Persistent WebSocket          │ Persistent WebSocket
       ▼                                       ▼                               ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               WebSocket Gateway Tier                                   │
│            (Terminates TLS, maintains long-lived TCP socket connections)               │
│               [ Gateway Pod 1 ]                           [ Gateway Pod 2 ]            │
└──────────────────────────────────────┬─────────────────────────────────────────────────┘
                                       │
                      Internal Event   │ (Publish action to room channel)
                                       ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              Redis Pub/Sub Message Bus                                 │
│                     Channels: `room:7X9Q2A`, `room:9K2B1P`                             │
└──────────────────────────────────────┬─────────────────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               Room State Cache (Redis)                                 │
│                Key: `room:7X9Q2A` (Status, Timeline Anchor, Speed, Host ID)            │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 4.1 End-to-End Happy Flow Across All Services

To explain the complete system clearly during an interview, walk through this 4-step user lifecycle:

```
[ 1. Create Room ] ──▶ [ 2. Join & Time Sync ] ──▶ [ 3. Host Plays Video ] ──▶ [ 4. Drift Correction ]
  Generates 6-char      Measures Round-Trip         Schedules Future Epoch       Smooth Speed Tune
  ID in Redis           Time via Ping               to align all players         (0.95x or 1.05x)
```

#### Step 1: Host Creates a Room
1. Host clicks *"Create Watch Party"* in their browser.
2. The browser sends `POST /api/v1/rooms` with the movie ID.
3. The API server generates a clean, URL-safe 6-character room code (`7X9Q2A`), saves the initial room state (`status: PAUSED, time: 0`) in **Redis**, and returns the shareable invite link.

#### Step 2: Participants Join & Synchronize Clocks
1. Viewer receives the link and opens the page.
2. Browser opens a WebSocket connection to the **WebSocket Gateway**: `wss://.../rooms/7X9Q2A`.
3. The gateway registers the user in the room's participant list in Redis and subscribes this gateway pod to the Redis Pub/Sub channel `room:7X9Q2A`.
4. The client exchanges 3 quick heartbeat ping/pong packets with the server to measure **Round-Trip Time (RTT)** and align its internal clock with the server's master clock.
5. The server sends the current room state: *"Movie is PAUSED at 00:15:30"*. The client player seeks to `00:15:30` and displays the paused frame.

#### Step 3: Host Hits "Play" (Synchronized Future Execution)
1. The host clicks **Play** at movie timestamp `00:15:30`:
   - Host's browser sends WebSocket command: `{action: "PLAY", media_time: 930000}`.
2. The Gateway receives the event, updates Redis state to `PLAYING`, and calculates a **Scheduled Future Epoch Time**:
   - `scheduled_epoch = current_server_time + 300ms`.
3. The Gateway publishes the event to Redis Pub/Sub channel `room:7X9Q2A`.
4. All Gateway pods holding connections for that room receive the message and push it to their local WebSocket clients.
5. Every client (Host, Alice, Bob) receives the message with 150ms to spare, counts down, and starts video playback at the **exact same millisecond**.

#### Step 4: Continuous Drift Correction
1. Every 5 seconds, client players report their current playback timestamp.
2. If Bob's player lags behind by $120\text{ms}$ due to a temporary Wi-Fi stutter:
   - The system detects the drift.
   - Instead of jarringly freezing the video, Bob's player **silently speeds up to $1.05\times$** for 2.4 seconds until he catches up, then returns to $1.0\times$.

---

### 4.2 The 3 Options for Video Playback Synchronization

When an interviewer asks: *"How do you keep video frames synchronized across different users?"* Evaluate these 3 approaches:

#### Option 1 (Bad): Naive Instant Event Forwarding
- **How it works**: The host hits Play. The server immediately forwards the `PLAY` command to everyone.
- **Why it breaks**: If Alice has a $20\text{ms}$ network delay and Bob has a $300\text{ms}$ network delay, Bob starts playing 280ms later than Alice. When the host pauses, Bob pauses even later. Over a standard movie, participants end up 5–15 seconds out of sync!

#### Option 2 (Okay): Continuous Server Clock Polling
- **How it works**: Every client continuously queries the server every second: *"What timestamp should I be playing right now?"*
- **Why it's flawed**: If a client corrects its timestamp every second, the video visibly stutters, audio pitch warbles, and 1,000,000 rooms flood the server with millions of useless polling requests.

#### Option 3 (Best / Standard): Future Epoch Scheduling + Smooth Drift Tuning
- **How it works**:
  1. **Future Scheduling for Actions**: When an action occurs, the server schedules execution for a fixed future window (e.g. `now + 300ms`), allowing all clients to buffer and start simultaneously.
  2. **Multi-Tier Drift Compensation**: Clients periodically check their drift against the virtual timeline and use smooth speed adjustments ($0.95\times$ or $1.05\times$) to align imperceptibly.

---

### 4.3 Step-by-Step Architectural Evolution (Whiteboard Script)

#### Phase 1: Simple WebSockets on a Single Node
- **Whiteboard**: `Host & Viewers` ──► `Single Node.js WebSocket Server` ──► `In-Memory Room Map`.
- **Explanation**: *"We start with a single WebSocket server. When the host pauses, the server iterates through its connected sockets for that room and sends a pause message."*
- **Bottlenecks**: A single server is a Single Point of Failure (SPOF) and cannot scale beyond 50,000 open connections.

#### Phase 2: Horizontally Scaled WebSockets + Redis Pub/Sub
- **Whiteboard**: Add multiple **WebSocket Gateway Servers** backed by **Redis Pub/Sub**.
- **Explanation**: *"We distribute client connections across an autoscaling pool of gateways. When the host sends an action to Gateway 1, it publishes to Redis channel `room:{id}`. Gateway 2, holding connections for other viewers, receives the event and pushes it down to its clients."*

#### Phase 3: Solving Network Jitter via Future Scheduled Epochs
- **Whiteboard**: Add **Server Epoch Timestamps** to the control protocol.
- **Explanation**: *"To prevent participants on slow networks from starting late, the server timestamps commands with a future execution clock (`now + 300ms`)."*

#### Phase 4: Dynamic Playback Rate Drift Correction
- **Whiteboard**: Add a client-side **Sync Engine** that micro-adjusts player speed ($0.95\times - 1.05\times$).
- **Explanation**: *"Instead of hard seeks that cause buffering spinners, minor playback drift is resolved smoothly by temporarily altering player speed."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: Clock Synchronization in Simple Terms (NTP Algorithm)

How do we know what time it is on Bob's computer versus the server?

Bob's device clock might be set 2 seconds fast or 5 seconds slow. We cannot rely on the client's local system clock. We must calculate the **Clock Offset ($\theta$)** and **Round-Trip Time ($RTT$)**:

```
Client (Sends Ping)        Server (Receives & Replies)
       │                                │
  t0 ──┼─────── Ping Request ──────────▶│ t1 (Server receives)
       │                                │ t2 (Server transmits response)
  t3 ◀─┼─────── Pong Response ──────────┤
       │                                │
```

1. Client sends ping at client time $t_0$.
2. Server receives ping at server time $t_1$, and sends reply at $t_2$.
3. Client receives reply at client time $t_3$.
4. **Calculations**:
   $$\text{Round-Trip Time } (RTT) = (t_3 - t_0) - (t_2 - t_1)$$
   $$\text{Clock Offset } (\theta) = \frac{(t_1 - t_0) + (t_2 - t_3)}{2}$$
5. **Result**: For any local timestamp, the client knows:
   $$\text{True Server Time} = \text{Client Local Time} + \theta$$
   Now, all participants are synchronized to the exact same master time scale!

---

### 5.2 Deep Dive 2: The Three-Tier Drift Correction Engine

During movie playback, participants gradually drift apart due to minor CPU throttling, tab backgrounding, or packet delays. The client-side sync engine evaluates drift every 5 seconds:

$$\text{Drift} = \text{Client Video Timestamp} - \text{Expected Server Timestamp}$$

```
┌─────────────────────────────────┬───────────────────────────────┬────────────────────────────────────────────────────────┐
│ Measured Drift Magnitude        │ Severity Tier                 │ Action Taken by Player                                 │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ |Drift| < 50 ms                 │ Tier 1: Imperceptible         │ Do Nothing. Human eye cannot notice 50ms variance.     │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ 50 ms ≤ |Drift| ≤ 1,000 ms (1s) │ Tier 2: Mild Drift            │ Smooth Speed Tuning (Rate Adjust):                     │
│                                 │                               │ • If lagging: Play at 1.05x speed until synced.        │
│                                 │                               │ • If ahead: Play at 0.95x speed until synced.          │
│                                 │                               │ (Audio pitch is maintained; zero video buffering).     │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ |Drift| > 1,000 ms (> 1s)       │ Tier 3: Severe Disconnect     │ Hard Seek: Force player to jump directly to the target │
│                                 │                               │ timestamp and display a brief buffering spinner.       │
└─────────────────────────────────┴───────────────────────────────┴────────────────────────────────────────────────────────┘
```

---

### 5.3 Deep Dive 3: Handling Buffering & Slow Network Participants

What happens if Bob's internet drops and his video starts buffering?

#### The Two Strategies Evaluated:
1. **Option A: Freeze the Entire Room for Everyone (Anti-Pattern)**:
   - When Bob buffers, he sends `{status: "BUFFERING"}`. The server pauses the movie for all 50 people until Bob finishes buffering.
   - *Why it's awful*: A single user on a flaky 3G connection ruins the entire movie night for 49 other people.
2. **Option B: Host-Authoritative Uninterrupted Playback (The FAANG Standard)**:
   - The movie continues playing for everyone else.
   - Bob's client pauses locally and shows a local buffering indicator.
   - When Bob's video buffer fills up, his client calculates where the host is right now and executes a **Tier 3 Hard Seek** to jump straight to the host's live position.
   - *Exception*: If the **Host** buffers, the server automatically pauses the room for everyone, because the host is the master timeline anchor.

---

### 5.4 Deep Dive 4: Multi-Server WebSocket Scaling via Redis Pub/Sub

When 1,000,000 watch party rooms are active, how do multiple gateway servers coordinate?

```
[ Host Alice on Gateway 1 ] ──▶ (Publishes to Redis Channel `room:7X9Q2A`)
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      ▼                                               ▼
         [ Gateway 1 (Local Sockets) ]                   [ Gateway 2 (Local Sockets) ]
                      │                                               │
                      ▼                                               ▼
              [ Alice (Host) ]                                 [ Bob & Charlie ]
```

1. **Room-Based Pub/Sub Channels**:
   - Each room maps to a dedicated Redis channel: `room:{room_id}`.
2. **Selective Subscription**:
   - Gateway Pod 2 only subscribes to `room:7X9Q2A` if at least one participant connected to Gateway Pod 2 belongs to that room.
   - When all users in that room on Gateway Pod 2 disconnect, it unsubscribes from that channel.
   - Keeps network traffic strictly contained to servers actively hosting participants.

---

### 5.5 Deep Dive 5: Host Disconnection & Host Migration Protocol

What happens if the Host's laptop battery dies or their Wi-Fi disconnects?

1. **Heartbeat Timeout**:
   - The WebSocket gateway monitors client heartbeats. If no ping is received from the Host for **10 seconds**, the host socket is marked dead.
2. **Grace Period (30 Seconds)**:
   - The room stays alive in `PLAYING` state for 30 seconds to allow the host to reconnect (e.g. refreshing browser or switching Wi-Fi).
3. **Automatic Host Migration**:
   - If the host does not return after 30 seconds:
     - The server queries the room's participant list in Redis.
     - Selects the next participant with the **earliest join timestamp**.
     - Updates `host_id` in Redis and broadcasts `{type: "HOST_CHANGED", new_host_id: "usr_bob"}`.
     - Bob now gains the play/pause/seek controls.

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Media Delivery** | Streaming Provider CDN (Netflix / S3) | Streaming video through WebSocket server | Streaming 4K video through sync servers requires petabits of bandwidth. Only control signals ($<1\text{KB/s}$) pass through WebSockets. |
| **Action Synchronization**| Future Scheduled Epoch Execution | Immediate Event Broadcast | Immediate broadcast causes slow network users to lag behind. Future scheduling ensures all clients execute at the exact same millisecond. |
| **Drift Correction** | 3-Tier: Smooth Speed Tuning ($0.95\times-1.05\times$) | Constant Hard Seeks | Hard seeks cause video freezing and audio pops. Smooth speed tuning eliminates drift without visible interruption. |
| **Buffering Policy** | Host-Authoritative (Room keeps playing) | Pause entire room for slow users | Freezing the room for all 50 viewers whenever 1 user buffers ruins the user experience. |
| **Socket Coordination** | Redis Pub/Sub partitioned by Room ID | Single Monolithic WebSocket Gateway | Redis Pub/Sub allows horizontal scaling of millions of WebSockets across dozens of stateless server pods. |
| **Room State Storage** | In-Memory Redis Hash with Virtual Clock | Persisting every second to Relational DB | Virtual clock calculates movie position dynamically using epoch offsets, eliminating 1,000,000 DB writes per second. |

---

### Top 4 Interview Follow-Up Questions (Rapid-Fire Defenses)

### Q1: How does a new participant who joins 45 minutes into the movie get in sync?
- **Answer**: When the user opens the WebSocket connection, the server reads the room's Redis hash (`anchor_media_time_ms`, `anchor_server_epoch_ms`, `status`). It calculates the current movie timestamp using the virtual clock formula and returns a `{type: "INITIAL_SYNC", media_time_ms: 2715000, status: "PLAYING"}` payload. The player seeks to 45:15 and starts playing immediately.

### Q2: What happens if the host seeks backward by 30 seconds?
- **Answer**: The host's client sends `{action: "SEEK", media_time_ms: 1215000}`. The server updates the anchor timestamp in Redis and broadcasts `{action: "SEEK", media_time_ms: 1215000, scheduled_epoch: now + 300ms}`. All clients pause briefly, seek their video players to the new timestamp, and resume playing together at the scheduled epoch.

### Q3: How do you prevent malicious participants from pausing or seeking the video?
- **Answer**: The API Gateway verifies the user's JWT token against the room's `host_id` stored in Redis. If a non-host user sends a `PLAY`, `PAUSE`, or `SEEK` command, the gateway discards the message and returns `HTTP 403 Forbidden` unless the room configuration explicitly enables *"Collaborative Control"*.

### Q4: Does audio pitch distort when you speed up playback to 1.05x during drift correction?
- **Answer**: Modern HTML5 video players and mobile media frameworks (like Exoplayer and AVPlayer) include native **time-stretching algorithms (pitch correction)**. Minor speed adjustments between $0.95\times$ and $1.05\times$ preserve the original audio pitch completely, making the drift correction 100% inaudible to the human ear.
