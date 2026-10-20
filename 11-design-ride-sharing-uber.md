# System Design 11: Design Ride Sharing Application (Uber / Lyft / Ola / Rapido)

> **Source**: [Interview with Bunny - Episode 11](https://www.youtube.com/watch?v=twUz_hWw2T8)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/11](https://www.interviewwithbunny.com/systemdesign/11)  
> **Topic**: Architecting a Scalable, Fault-Tolerant, Hyperlocal Real-Time Ride-Hailing Platform (Uber, Lyft, Ola)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Clarifying Questions & Core Scope
- **Geographic distribution**: Global platform partitioned into localized geofences/cities.
- **Matching criteria**: Proximity-based sequential or batch dispatch to the nearest available driver within a 5–10 km radius.
- **Vehicle classifications**: Support multiple categories (Bike, Auto, Economy Sedan, Premium SUV) with discrete rate cards.
- **Pricing model**: Upfront guaranteed fare estimation with dynamic surge multiplier based on real-time localized supply-demand imbalance.
- **Tracking & communication**: Continuous sub-second bi-directional GPS tracking over WebSockets during active trips.

---

### 1.2 Functional Requirements (FR)
1. **Fare Estimation**: Riders enter pickup and destination coordinates $(\text{lat}, \text{lon})$ and receive upfront estimated fares across vehicle tiers (Bike, Sedan, SUV) along with route distance and ETA.
2. **Ride Request**: A rider can select a vehicle tier and confirm a ride request based on a valid quotation token (`request_id`).
3. **Driver Proximity Matching & Dispatch**: The system identifies candidate idle drivers within a 5–10 km radius using spatial indexing and sequentially or conditionally offers the trip.
4. **Driver Acceptance & Rejection**: A notified driver has a strict 10-second countdown window to accept or decline. If rejected or timed out, the system waterfall-dispatches to the next candidate.
5. **Real-Time Live Location Tracking**: Both rider and driver can track each other's live location on an interactive map with sub-second latency from dispatch until trip completion.
6. **Trip Lifecycle Management**: Drivers can start the ride (with OTP verification) and terminate the ride at the destination point.
7. **Billing & Mutual Rating**: Upon trip termination, the system computes the final fare (accounting for route deviations or wait times), processes payment via payment gateways, and enables mutual 1-to-5 star ratings and reviews.

---

### 1.3 Non-Functional Requirements (NFR)
1. **CAP Theorem Posture (Partition Tolerance + Availability vs. Consistency)**:
   - **Browse & Estimation Paths (AP)**: High availability ($99.999\%$). Viewing estimates, browsing vehicle types, and reading trip history must never be blocked by regional partitions.
   - **Driver Dispatch & Assignment (CP)**: Strict consistency. A driver must **never** be concurrently double-booked or assigned to multiple active trips simultaneously.
2. **Latency Budgets**:
   - Upfront fare calculation: $< 200\text{ ms}$.
   - Driver matching loop: Total matching SLA $\le 60\text{ seconds}$ (sequential dispatch to top $K=5$ candidates, 10s timeout each).
   - Live location updates: End-to-end GPS latency $< 1\text{ second}$ over persistent WebSockets.
3. **High Throughput & Ingestion Resilience**:
   - Geospatial ingestion from hundreds of thousands of concurrent active drivers transmitting GPS pings every 3–5 seconds without saturating persistent disk I/O.
4. **Auditability & Traceability**:
   - Complete route coordinates logged durably for post-trip dispute resolution, route deviations, and refund recalculations.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Scale & User Base
- **Monthly Active Users (MAU)**: $100\text{ Million}$.
- **Daily Active Riders (DAU)**: $20\text{ Million}$.
- **Daily Active Drivers**: $2\text{ Million}$ globally ($500,000$ concurrent active drivers during peak hours).
- **Daily Completed Trips**: $15\text{ Million trips/day}$.
- **Peak Trip Request QPS**:
  $$\text{Peak Request QPS} = \frac{15,000,000 \times 2 \text{ (rush multiplier)}}{86,400\text{ s}} \approx 350 \text{ ride requests/sec}$$

#### Location Ingestion Throughput
- $500,000$ active drivers pinging GPS coordinates every $4\text{ seconds}$:
  $$\text{Location Ingestion QPS} = \frac{500,000}{4} = 125,000\text{ pings/sec}$$
- At $125,000\text{ writes/sec}$, relational databases (PostgreSQL/MySQL) writing to disk would instantly choke IOPS and lock table indexes. 
- **Architectural Mandate**: Location ingestion must terminate in an **in-memory spatial store (Redis Geospatial / H3)** backed by an asynchronous event stream (**Apache Kafka**).

#### Bandwidth & Network Throughput
- Location ping payload size:
  - `driver_id` (16 bytes UUID) + `lat` (8 bytes double) + `lon` (8 bytes double) + `bearing` (4 bytes) + `timestamp` (8 bytes) + overhead $\approx 100\text{ bytes}$.
- **Ingress Bandwidth**:
  $$125,000\text{ pings/sec} \times 100\text{ bytes} = 12.5\text{ MB/sec} = 100\text{ Mbps}$$
- **Egress Bandwidth (Broadcasting to active riders tracking cabs)**:
  - Assume $200,000$ active matched trips concurrently streaming driver coordinates every 1 second:
  $$200,000 \times 100\text{ bytes} = 20\text{ MB/sec} = 160\text{ Mbps}$$

#### Storage Growth (5-Year Projection)
- **Trip Metadata**: $15\text{ Million trips/day} \times 2\text{ KB/record} = 30\text{ GB/day} \approx 11\text{ TB/year} \to 55\text{ TB in 5 years}$.
- **Trip Route Telemetry Logs (Kafka $\to$ S3 Glacier)**:
  - Average trip duration: $20\text{ minutes} = 1,200\text{ seconds} \to 300\text{ pings/trip}$.
  - $15\text{M trips/day} \times 300 \text{ pings} \times 100 \text{ bytes} \approx 450\text{ GB/day} \approx 164\text{ TB/year} \to 820\text{ TB in 5 years}$.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

```
+-----------------------------------------------------------------------------------+
|                                  CORE DOMAIN ENTITIES                             |
+-----------------------------------------------------------------------------------+

   +-----------------------+              1 : N             +-----------------------+
   |         Rider         |------------------------------->|      FareEstimate     |
   +-----------------------+                                +-----------------------+
   | rider_id (PK)         |                                | estimate_id (PK)      |
   | name, email, phone    |                                | rider_id (FK)         |
   | rating (FLOAT)        |                                | pickup_lat, pickup_lon|
   | created_at            |                                | drop_lat, drop_lon    |
   +-----------------------+                                | vehicle_tier          |
               | 1                                          | base_fare, surge_mult |
               |                                            | total_estimated_fare  |
               | 1 : N                                      | expires_at (TTL 5 min)|
               v                                            +-----------------------+
   +-----------------------+                                            | 1
   |         Ride          |<-------------------------------------------+
   +-----------------------+                                (redeemed by request)
   | ride_id (PK, UUID)    |
   | estimate_id (FK)      |              1 : N             +-----------------------+
   | rider_id (FK)         |------------------------------->|    RideLocationPing   |
   | driver_id (FK, Null)  |                                +-----------------------+
   | vehicle_id (FK)       |                                | ping_id (BIGSERIAL PK)|
   | pickup_address, geo   |                                | ride_id (FK)          |
   | drop_address, geo     |                                | driver_id (FK)        |
   | status (ENUM)         |                                | lat, lon, bearing     |
   | otp_code (VARCHAR)    |                                | recorded_at           |
   | started_at, ended_at  |                                +-----------------------+
   +-----------------------+
          | 1           | 1
          |             |
     1:1  |             | 1:1
          v             v
+------------------+  +--------------------+               +------------------------+
|     Payment      |  |     TripRating     |               |         Driver         |
+------------------+  +--------------------+               +------------------------+
| payment_id (PK)  |  | rating_id (PK)     |               | driver_id (PK, UUID)   |
| ride_id (FK)     |  | ride_id (FK)       |    N : 1      | name, phone, license_no|
| amount, currency |  | sender_id (FK)     |-------------->| status (IDLE/BUSY/OFF) |
| method, status   |  | receiver_id (FK)   |               | current_lat, lon       |
| transaction_ref  |  | score (1-5), review|               | overall_rating (FLOAT) |
+------------------+  +--------------------+               +------------------------+
                                                                       | 1
                                                                       | 1:N
                                                                       v
                                                           +------------------------+
                                                           |        Vehicle         |
                                                           +------------------------+
                                                           | vehicle_id (PK)        |
                                                           | driver_id (FK)         |
                                                           | tier (BIKE/SEDAN/XL)   |
                                                           | plate_number, model    |
                                                           +------------------------+
```

---

## 3. Step 3: API & Interface Design

### 3.1 Rider REST Endpoints

#### 3.1.1 Upfront Fare Estimation
Calculate distance, ETA, and base + surge prices for multiple tiers.

```http
POST /api/v1/rides/estimate
Content-Type: application/json
Authorization: Bearer <rider_jwt_token>

{
  "pickup_location": { "latitude": 12.9352, "longitude": 77.6245 },
  "drop_location": { "latitude": 12.9716, "longitude": 77.5946 }
}
```

**Response (`200 OK`)**:
```json
{
  "estimate_id": "est_7fa91c2b-4390-4c12",
  "route_distance_km": 8.4,
  "route_duration_minutes": 26,
  "expires_in_seconds": 300,
  "tier_estimates": [
    {
      "tier": "BIKE",
      "fare_amount": 95.00,
      "currency": "INR",
      "surge_multiplier": 1.0,
      "eta_arrival_mins": 3
    },
    {
      "tier": "SEDAN",
      "fare_amount": 240.00,
      "currency": "INR",
      "surge_multiplier": 1.25,
      "eta_arrival_mins": 5
    },
    {
      "tier": "SUV_XL",
      "fare_amount": 380.00,
      "currency": "INR",
      "surge_multiplier": 1.25,
      "eta_arrival_mins": 8
    }
  ]
}
```

---

#### 3.1.2 Request a Ride
Confirms the ride based on the estimation quotation.

```http
POST /api/v1/rides/request
Content-Type: application/json
Authorization: Bearer <rider_jwt_token>
X-Idempotency-Key: req_idem_90192830192

{
  "estimate_id": "est_7fa91c2b-4390-4c12",
  "selected_tier": "SEDAN",
  "payment_method_id": "pm_card_99812"
}
```

**Response (`202 Accepted`)**:
```json
{
  "ride_id": "ride_44c10a12-88ef-410a",
  "status": "SEARCHING_FOR_DRIVER",
  "timeout_seconds": 60,
  "created_at": "2026-10-18T18:15:30Z"
}
```

---

#### 3.1.3 Cancel Ride Request
```http
POST /api/v1/rides/{ride_id}/cancel
Authorization: Bearer <rider_jwt_token>

{
  "reason": "Driver took too long to move"
}
```
**Response (`200 OK`)**:
```json
{
  "ride_id": "ride_44c10a12-88ef-410a",
  "status": "CANCELLED",
  "cancellation_fee": 0.00
}
```

---

### 3.2 Driver REST & WebSocket Endpoints

#### 3.2.1 Driver Dispatch Response (Accept / Deny)
When an offer is pushed to the driver device via Push Notification or persistent WebSocket, the driver responds within 10 seconds.

```http
POST /api/v1/drivers/rides/{ride_id}/respond
Authorization: Bearer <driver_jwt_token>

{
  "action": "ACCEPT" // or "DECLINE"
}
```

**Response (`200 OK`)**:
```json
{
  "ride_id": "ride_44c10a12-88ef-410a",
  "status": "CONFIRMED",
  "pickup_location": { "lat": 12.9352, "lon": 77.6245 },
  "drop_location": { "lat": 12.9716, "lon": 77.5946 },
  "rider_details": {
    "name": "Alex M.",
    "phone": "+919876543210",
    "rating": 4.88
  },
  "expected_payout": 192.00
}
```

---

#### 3.2.2 Trip Lifecycle Transitions (Start & End)
```http
POST /api/v1/drivers/rides/{ride_id}/start
Authorization: Bearer <driver_jwt_token>

{
  "otp": "4921",
  "current_location": { "latitude": 12.9354, "longitude": 77.6247 }
}
```
**Response (`200 OK`)**: `{"status": "IN_PROGRESS", "started_at": "2026-10-18T18:22:10Z"}`

```http
POST /api/v1/drivers/rides/{ride_id}/end
Authorization: Bearer <driver_jwt_token>

{
  "current_location": { "latitude": 12.9718, "longitude": 77.5949 }
}
```
**Response (`200 OK`)**:
```json
{
  "ride_id": "ride_44c10a12-88ef-410a",
  "status": "COMPLETED",
  "final_fare": 240.00,
  "payment_status": "CHARGED_SUCCESS"
}
```

---

### 3.3 Real-Time WebSocket Interface

#### Driver GPS Ingestion Stream
```http
GET /ws/v1/driver/location
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer <driver_jwt_token>
```
*Client-to-Server Ping Frame (every 3 seconds)*:
```json
{
  "driver_id": "drv_9921",
  "latitude": 12.93531,
  "longitude": 77.62462,
  "bearing": 42.5,
  "speed_mps": 8.4,
  "status": "IDLE", // IDLE, ACCEPTED, ON_TRIP
  "timestamp": 1729275330100
}
```

#### Rider Live Ride Tracking Stream
```http
GET /ws/v1/rides/{ride_id}/track
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer <rider_jwt_token>
```
*Server-to-Client Broadcast Frame (every 1 second)*:
```json
{
  "ride_id": "ride_44c10a12-88ef-410a",
  "driver_location": { "lat": 12.93605, "lon": 77.62410 },
  "bearing": 45.0,
  "eta_minutes": 3,
  "distance_remaining_km": 0.8,
  "trip_status": "DRIVER_ARRIVING"
}
```

---

## 4. Step 4: High-Level Architecture (HLD)

```
                                  +---------------------------------------+
                                  |         DNS / Anycast Routing         |
                                  +---------------------------------------+
                                                      │
                         ┌────────────────────────────┴───────────────────────────┐
                         ▼                                                        ▼
         +───────────────────────────────+                        +───────────────────────────────+
         |     HTTP API Gateway & LB     |                        |   WebSocket Gateway Cluster   |
         | (Auth, Rate Limit, REST APIs) |                        | (Sticky Sessions, TLS Term.)  |
         +───────────────────────────────+                        +───────────────────────────────+
                         │                                                        │
          ┌──────────────┼──────────────┬──────────────┐                          │
          ▼              ▼              ▼              ▼                          ▼
  +──────────────+ +───────────+ +─────────────+ +───────────+          +────────────────────────+
  | Ride Service | | Matching  | |  Rating &   | | Payment   |          | Location Update Server |
  | (Fares/Est.) | |  Engine   | | Dispute Svc | |  Service  |          | (Active Driver Ingest) |
  +──────────────+ +───────────+ +─────────────+ +───────────+          +────────────────────────+
          │              │              │              │                          │
          │              ├──────────────┘              │                          │
          │              │                             │                          ▼
          │              ▼                             ▼                 +──────────────────+
          │     +────────────────+             +──────────────+          |  Redis Cluster   |
          │     | Apache         |             | Payment Gate |          | (Geospatial H3/  |
          │     | ZooKeeper      |             | (Stripe/UPI) |          |  Active Drivers) |
          │     | (Ephemeral     |             +──────────────+          +──────────────────+
          │     |  Driver Locks) |                                                │
          │     +────────────────+                                                │ (Forward active)
          │                                                                       ▼
          │     +───────────────────────────────────────────────────────────────────────+
          │     |                         Apache Kafka Event Bus                        |
          │     |   [driver-telemetry]      [ride-events]      [fare-request-stream]   |
          │     +───────────────────────────────────────────────────────────────────────+
          │                        │                    │                     │
          │                        ▼                    ▼                     ▼
          │             +────────────────────+  +───────────────+    +──────────────────+
          │             | Trip Telemetry Sinks| | Notification  |    | Surge Pricing    |
          │             | (Route Reconciler) |  | Svc (FCM/APNs)|    | Stream Calculator|
          │             +────────────────────+  +───────────────+    +──────────────────+
          │                        │                    │                     │
          ▼                        ▼                    ▼                     ▼
  +──────────────+      +────────────────────+  +───────────────+    +──────────────────+
  |   Ride DB    |      | Location Audit DB  |  | Push Provider |    | Surge State DB   |
  | (PostgreSQL  |      | (Cassandra / S3)   |  | (Apple / FCM) |    | (Hexagon Demand) |
  |  ACID Store) |      +────────────────────+  +───────────────+    +──────────────────+
  +──────────────+
```

### Architectural Component Walkthrough
1. **API Gateway & HTTP Load Balancer**: Handles stateless REST traffic (Authentication, HMAC signing, Rate Limiting, Request ID routing).
2. **Dedicated WebSocket Gateway Tier**: Decoupled from HTTP workers to handle persistent TCP connections. Maintains stateful socket maps with connection stickiness using internal Redis connection registries (`driver_id -> gateway_instance_id`).
3. **Ride Service**: Orchestrates fare estimation, interacts with Google/Apple Maps Distance Matrix APIs, and manages the ride lifecycle state machine.
4. **Surge Calculator Engine**: Asynchronously consumes trip search queries from Kafka and calculates geographic supply-demand ratios per hexagonal cell (Uber H3 Index).
5. **Driver Matching Service**: Implements the proximity search query against the in-memory **Redis Geospatial Index** and coordinates driver locking via **Apache ZooKeeper**.
6. **Location Update Server**: Ingests high-frequency GPS coordinate pings directly into RAM (Redis) and publishes them to Kafka `driver-telemetry` without touching relational databases.
7. **Kafka Event Stream**: Decouples write-heavy GPS streams from read pipelines, notifying tracking clients, feeding fraud detection, and saving historical routes to cold storage (S3).

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Proximity Search & Geospatial Indexing: Geohash vs. Uber H3 vs. QuadTree

```
       UBER H3 HEXAGONAL GRID SYSTEM            GEOHASH RECTANGULAR CELL BOUNDARY
                _                                    +--------------------+
              /   \                                  |   Neighbor NW      |
             /     \                                 |                    |
       +----<       >----+                           +----------+---------+
       |     \     /     |                           | Sub-cell | Center  |
       |      \ _ /      |                           | 8 char   | (Dist.) |
       +----<       >----+                           +----------+---------+
             \     /                                 | Edge distortion at |
              \ _ /                                  | high latitudes     |
       Equal neighbor distances (6)                  +--------------------+
```

#### Why Bunny Chooses Geohashing / Uber H3
- **QuadTree**: Kept entirely in-memory. Node splitting and rebalancing on every GPS move from $500,000$ drivers creates severe write-lock contention across concurrent threads.
- **Geohashing (Base32)**: Divides the Earth into rectangular bounding boxes. However, diagonal neighbors have distances $\sqrt{2}\times$ greater than direct horizontal/vertical neighbors, introducing distance distortions.
- **Uber H3 Hexagonal Grid (Production Standard)**:
  - Every hexagon has exactly **6 equidistant neighbors**.
  - Cell radius is uniform in all directions, making radial radius queries ($R = 5\text{ km}$) seamless.
  - H3 indexes are compact 64-bit unsigned integers (`uint64`), allowing rapid bitwise operations and efficient storage in Redis Sorted Sets or Hashes.

#### Redis Geospatial Implementation
Redis implements geospatial indexing internally using a **Sorted Set (ZSET)** where the score is a 52-bit integer representing the **Geohash** of the $(\text{longitude}, \text{latitude})$ coordinates:

```bash
# Ingest active driver location ping into Redis Geospatial Index:
GEOADD active_drivers:tier_sedan 77.62462 12.93531 "drv_9921"

# Query top nearest available drivers within 5 km of pickup point:
GEOSEARCH active_drivers:tier_sedan FROMLONLAT 77.6245 12.9352 BYRADIUS 5 km ASC WITHDIST WITHCOORD COUNT 10
```

---

### 5.2 Dynamic Surge Pricing Computation Engine

Surge pricing balances marketplace liquidity during supply crunches.

```
+--------------------+       Write Query Event       +-------------------------+
|   Rider App /      |------------------------------>|  Kafka Topic:           |
|   Ride Service     |                               |  [fare-request-stream]  |
+--------------------+                               +-------------------------+
                                                                  │
                                                                  ▼
                                                     +-------------------------+
                                                     | Flink / Kafka Streams   |
                                                     | Sliding Window (5 mins) |
                                                     +-------------------------+
                                                                  │
                                      Aggregates Demand           │ Correlates Active
                                      Per H3 Hexagon Cell         │ Idle Drivers
                                                                  ▼
+--------------------+   Fetches Multiplier    +-------------------------------+
|    Ride Service    |<------------------------|  Surge Multiplier Cache       |
|    Fare Engine     |                         |  (Redis Hash: h3_cell -> 1.8x)|
+--------------------+                         +-------------------------------+
```

#### Mathematical Formulation
$$\text{Surge Multiplier } (M) = 1.0 + \alpha \cdot \max\left(0, \frac{\text{Demand } (D) - k \cdot \text{Supply } (S)}{S + \epsilon}\right)$$
- $D$: Number of ride quotation requests generated within the H3 cell in the past 5 minutes.
- $S$: Number of available `IDLE` drivers inside the H3 cell and adjacent level-1 neighbor rings.
- $\alpha$: Elasticity tuning coefficient ($0.2 \le \alpha \le 0.5$).
- Cap bounds: Multiplier is clamped between $1.0\times$ and $3.5\times$ to avoid extreme price gouging.

---

### 5.3 Driver Allocation & Distributed Locking: ZooKeeper Ephemeral Nodes vs. Redis Redlock

#### The Race Condition Dilemma
In high-density zones (e.g., airports, stadium exits), multiple riders request rides simultaneously. Without locking, matching instances $S_1$ and $S_2$ could concurrently pull the same idle driver $D_1$ and issue conflicting dispatch offers. If both are accepted, the driver is double-booked.

```
       Matching Service Instance 1                      Matching Service Instance 2
                 │                                                │
                 │ 1. Tries to lock drv_9921                      │ 1. Tries to lock drv_9921
                 ▼                                                ▼
     +──────────────────────────────────────────────────────────────────────────+
     |                       Apache ZooKeeper Ensemble                          |
     |                     Persistent Node: /uber/driver_locks                  |
     +──────────────────────────────────────────────────────────────────────────+
                 │                                                │
      Creates Ephemeral Node:                                     │
      /uber/driver_locks/drv_9921                                 │ NodeExistsException!
      ==> SUCCESS (Lock Acquired)                                 ==> Fails. Pick next driver!
                 │                                                │
                 ▼                                                ▼
     Notifies drv_9921 via Push                         Skips drv_9921, locks drv_8830
```

#### ZooKeeper Ephemeral Node Mechanism
- **Creation**: The service attempts to create an ephemeral znode `/uber/driver_locks/{driver_id}`.
- **Mutual Exclusion**: ZooKeeper guarantees atomic linearizable writes across the Zab quorum. If the znode already exists, a `NodeExistsException` is returned.
- **Crash Safety**: Ephemeral znodes are tied directly to the client’s ZooKeeper heartbeat session. If Matching Service $S_1$ crashes midway, ZooKeeper detects the heartbeat timeout and automatically tears down the znode, preventing orphaned locks.
- **Timeout Cleanup**: If the driver fails to respond within 10 seconds, $S_1$ explicitly deletes `/uber/driver_locks/{driver_id}` and falls back to candidate driver 2.

#### Redis Distributed Lock Comparison (Redlock / Lua Script)
Alternatively, Redis can be used with an atomic Lua script:
```lua
-- Acquire lock on driver with 10-second TTL
if redis.call("SET", KEYS[1], ARGV[1], "NX", "PX", 10000) then
    return 1
else
    return 0
end
```
> **Bunny's Trade-off Recommendation**: ZooKeeper provides strict CP guarantees under network partitions and avoids Redlock clock-drift issues. However, Redis Lua script locks have lower latency and are simpler to operate if the team is already heavily invested in Redis for geospatial indexing.

---

### 5.4 Sequential Driver Matching Waterfall Algorithm

To satisfy the **$\le 60\text{ second}$ Latency Budget**, the system limits matching to candidate size $K = 5$:

```
Rider Requests Ride
        │
        ▼
Find Top 5 Nearest Drivers via Redis GEOSEARCH
        │
        ▼
   Set Index i = 1
        │
  ┌────►│
  │     ▼
  │  Attempt ZooKeeper Lock for Driver[i]
  │     │
  │     ├─ Lock Busy? ───────────────► Increment i ──────┐
  │     │                                                │
  │     └─ Lock Acquired                                 │
  │             │                                        │
  │             ▼                                        │
  │        Send Push Notification (10s Countdown)        │
  │             │                                        │
  │             ├─ Accepted? ─► Convert to Confirmed     │
  │             │               Trip; Release Lock; EXIT │
  │             │                                        │
  │             ├─ Declined / Timed Out (10s)            │
  │             │       │                                │
  │             │       ▼                                │
  │             │   Delete ZooKeeper Lock                │
  │             │   Increment i                          │
  │             │       │                                │
  │             │       ▼                                │
  │             └───────┤                                │
  │                     ▼                                │
  │              Is i <= K (5)?                          │
  │                     ├─ YES ──────────────────────────┘
  │                     │
  │                     └─ NO (All 5 rejected/busy)
  │                             │
  │                             ▼
  └────────────────── Return: "No Drivers Found"
```

---

### 5.5 Database Schemas & Partitioning Strategy

#### PostgreSQL (ACID Relational Core for Rides & State)

```sql
-- Ride Master Table
CREATE TABLE rides (
    ride_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rider_id UUID NOT NULL REFERENCES riders(rider_id),
    driver_id UUID REFERENCES drivers(driver_id),
    vehicle_tier VARCHAR(20) NOT NULL,
    pickup_latitude NUMERIC(10, 7) NOT NULL,
    pickup_longitude NUMERIC(10, 7) NOT NULL,
    pickup_address TEXT,
    drop_latitude NUMERIC(10, 7) NOT NULL,
    drop_longitude NUMERIC(10, 7) NOT NULL,
    drop_address TEXT,
    status VARCHAR(30) NOT NULL DEFAULT 'REQUESTED',
    otp_code VARCHAR(6),
    estimated_fare NUMERIC(10, 2) NOT NULL,
    actual_fare NUMERIC(10, 2),
    surge_multiplier NUMERIC(3, 2) DEFAULT 1.00,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ
);

CREATE INDEX idx_rides_rider ON rides (rider_id, created_at DESC);
CREATE INDEX idx_rides_driver ON rides (driver_id, created_at DESC);
CREATE INDEX idx_rides_status ON rides (status) WHERE status IN ('REQUESTED', 'ACCEPTED', 'IN_PROGRESS');

-- Estimated Fare Quotation Cache Table
CREATE TABLE fare_estimates (
    estimate_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rider_id UUID NOT NULL,
    pickup_point GEOGRAPHY(Point, 4326) NOT NULL,
    drop_point GEOGRAPHY(Point, 4326) NOT NULL,
    base_fare NUMERIC(10, 2) NOT NULL,
    surge_multiplier NUMERIC(3, 2) NOT NULL,
    total_fare NUMERIC(10, 2) NOT NULL,
    vehicle_tier VARCHAR(20) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_estimates_expires ON fare_estimates (expires_at);

-- Driver Master Metadata Table
CREATE TABLE drivers (
    driver_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    license_number VARCHAR(50) UNIQUE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'OFFLINE', -- OFFLINE, IDLE, ON_TRIP
    rating NUMERIC(3, 2) DEFAULT 5.00,
    total_trips INTEGER DEFAULT 0,
    current_latitude NUMERIC(10, 7),
    current_longitude NUMERIC(10, 7),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_drivers_status ON drivers (status);

-- Granular Rating Table
CREATE TABLE trip_ratings (
    rating_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ride_id UUID NOT NULL REFERENCES rides(ride_id),
    sender_id UUID NOT NULL,
    receiver_id UUID NOT NULL,
    role VARCHAR(10) NOT NULL, -- 'RIDER_TO_DRIVER' or 'DRIVER_TO_RIDER'
    rating_score SMALLINT CHECK (rating_score BETWEEN 1 AND 5),
    feedback_text TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_ratings_receiver ON trip_ratings (receiver_id);
```

#### Driver Telemetry Route Storage: Apache Cassandra / ScyllaDB
For storing millions of GPS breadcrumb trails generated during trips:

```cql
CREATE KEYSPACE uber_telemetry WITH replication = {
    'class': 'NetworkTopologyStrategy', 
    'us-east': 3, 
    'us-west': 3
};

CREATE TABLE uber_telemetry.ride_breadcrumbs (
    ride_id uuid,
    recorded_at timestamp,
    latitude double,
    longitude double,
    bearing float,
    speed_mps float,
    PRIMARY KEY (ride_id, recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at ASC);
```
- **Partition Key**: `ride_id` groups all coordinates for a single trip on the same storage node.
- **Clustering Column**: `recorded_at ASC` stores breadcrumbs sequentially on disk, enabling instantaneous range scans for route replay.

---

### 5.6 Estimated vs. Actual Fare Reconciliation & Fraud Auditing

```
                           +---------------------------+
                           |  Driver Ends Ride in App  |
                           +---------------------------+
                                         │
                                         ▼
                           +---------------------------+
                           |    Ride Service Ingests   |
                           |   Drop GPS Coordinates    |
                           +---------------------------+
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
+---------------------------------+             +---------------------------------+
| Calculate Traveled Route Length |             |  Fetch Upfront Fare Estimation  |
| (Sum distances of breadcrumbs   |             |  (fare_estimates table record:  |
|  fetched from Cassandra)        |             |   quoted base + surge)          |
+---------------------------------+             +---------------------------------+
                 │                                               │
                 └───────────────────────┬───────────────────────┘
                                         ▼
                 +-----------------------------------------------+
                 | Has Route Distance Deviated by > 15% or Wait  |
                 | Duration Exceeded Normal Limits?              |
                 +-----------------------------------------------+
                                  │             │
                             NO   │             │  YES
            ┌─────────────────────┘             └────────────────────┐
            ▼                                                        ▼
+-----------------------+                                +-----------------------+
| Charge Exactly Upfront|                                | Recalculate Fare with |
| Estimated Quoted Fare |                                | Actual Road Tolls,    |
+-----------------------+                                | Distance & Wait Time  |
                                                         +-----------------------+
                                                                     │
                                                                     ▼
                                                         +-----------------------+
                                                         | Flag to Audit/Dispute |
                                                         | Engine for Review     |
                                                         +-----------------------+
```

1. **Upfront Fare Guarantee**: If the actual traversed path remains within a tolerance threshold (e.g., within $15\%$ of original routing and no significant detours), the customer is charged the exact upfront quotation.
2. **Dynamic Recalculation**: If traffic jams or deliberate driver detours cause massive deviation, the fare service reconciles the meter using recorded Cassandra coordinates and posts an itemized receipt.
3. **Dispute & Refund Processing**: If a rider reports driver manipulation or route inflation, support agents query the Cassandra breadcrumb ledger against the optimal Google Maps polyline to issue automated refunds.

---

### 5.7 Rating Aggregation Pipeline

To prevent read-time lock contention from aggregating ratings on every driver lookup:
1. Each completed ride rating is inserted as an append-only row into `trip_ratings`.
2. A Kafka event `rating-submitted-event` is emitted.
3. An **Aggregator Worker Service** consumes the stream and maintains an in-memory decay or exponential moving average:
   $$\text{New Rating} = \frac{\sum_{i=1}^{N} \text{score}_i \cdot e^{-\lambda(t_{\text{now}} - t_i)}}{\sum_{i=1}^{N} e^{-\lambda(t_{\text{now}} - t_i)}}$$
4. The worker periodically flushes the updated average rating into the `drivers` table and the Redis active driver cache.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you prevent double assignment of a driver to multiple ride requests?

Use Zookeeper distributed locking with ephemeral nodes: Scenario: Driver D1 available at location X, simultaneously 3 ride requests R1, R2, R3 created nearby. Driver Matching Service instances (DMS1, DMS2, DMS3) each select D1 as best match for their respective requests. Lock acquisition:

- **Point 1**: DMS1 tries to create /locks/drivers/D1/request_R1_seq0001 (ephemeral, sequential node), Zookeeper assigns sequence 0001,

- **Point 2**: DMS2 tries to create /locks/drivers/D1/request_R2_seq0002, Zookeeper assigns sequence 0002,

- **DMS3 tries to create /locks/drivers/D1/request_R3_seq0003, sequence 0003. Checking lock ownership**:
  Each DMS gets children of /locks/drivers/D1: [request_R1_seq0001, request_R2_seq0002, request_R3_seq0003]. DMS1 sees its node (seq0001) is lowest → lock acquired → proceeds with assignment. DMS2 and DMS3 see seq0001 < their sequences → lock NOT acquired → must try next driver. DMS1 proceeds:

- **Mark D1 as BUSY in Redis**:
  SET driver:D1:status 'BUSY' EX 900,

- **Remove from available drivers**:
  GEOREM drivers:available D1,

- **Update database**:
  UPDATE drivers SET status='BUSY', current_ride_id={R1},

- **Send push notification to driver for accept/decline via Kafka → Notification Service. DMS2 and DMS3 retry**:
  

- **Point 1**: Delete their Zookeeper nodes (didn't get lock),

- **Point 2**: Select next closest driver (D2, D3) from candidate list,

- **Attempt lock acquisition for new drivers. Driver D1 responds**:
  If accepts → DELETE /locks/drivers/D1/request_R1_seq0001 (lock released, no longer needed), CREATE trip in DB, notify rider. If declines or timeout (30 sec) →

- **Point 1**: Zookeeper session expires → ephemeral node auto-deleted,

- **D1 marked AVAILABLE again**:
  SET driver:D1:status 'AVAILABLE', GEOADD drivers:available {lon} {lat} D1,

- **System can retry D1 for other requests. Why Zookeeper?**:
  

- **Point 1**: Strong consistency (CP system) - guarantees only one lock holder,

- **Point 2**: Ephemeral nodes - auto-cleanup on failure/timeout,

- **Point 3**: Atomic operations - sequential node creation with ordering,

- **Session management - built-in detection of client crashes. Alternative approach**:
  Redis distributed lock with Redlock algorithm:

- **SETNX lock**:
  driver:D1 {request_id} EX 30,

- **Point 2**: If returns 1 → lock acquired,

- **If returns 0 → already locked. But Redis approach requires**:
  (a) Manual TTL management, (b) No automatic ordering of concurrent lock attempts, (c) Risk of TTL expiring before lock released if service slow. Zookeeper provides stronger guarantees for critical use case (preventing revenue loss from double assignments, poor UX).

---

### Q2: How does real-time location tracking work with WebSockets?

WebSocket enables bidirectional persistent connection for sub-second location updates: Architecture: Driver app maintains WebSocket connection to Location Update Service (WebSocket server), rider app subscribes to driver location updates via separate WebSocket. Driver side:

- **Driver goes online → app requests location permission → establishes WebSocket**:
  WS /v1/driver/location with JWT token,

- **Location Update Service authenticates**:
  Validates JWT, checks driver approved, creates session {session_id, driver_id, connected_at},

- **GPS tracking starts**:
  Android/iOS location API provides updates every 3-5 seconds with {lat, lon, accuracy, speed, timestamp},

- **App sends via WebSocket**:
  {driver_id, lat: 37.7849, lon: -122.4194, accuracy: 10m, speed: 25 km/h, timestamp},

- **Location Update Service processes**:
  (a) Validates driver in active trip or available, (b) Updates Redis geospatial: GEOADD drivers:available {lon} {lat} {driver_id} if driver available, (c) Sets location cache: SET location:{driver_id} '{lat, lon, timestamp}' EX 60 (1 min TTL), (d) Publishes to Redis Pub/Sub: PUBLISH driver_location:{driver_id} '{lat, lon, timestamp}' for real-time subscribers, (e) Sends to Kafka: 'driver.location_updated' event for persistence → Trip Update Consumer writes to location_history table. Rider side:

- **Rider matched with driver → app subscribes**:
  WS /v1/trip/{trip_id}/track with rider auth token,

- **WebSocket Gateway authenticates**:
  Validates rider_id owns this trip_id,

- **Gateway subscribes to Redis Pub/Sub**:
  SUBSCRIBE driver_location:{driver_id},

- **When driver location published → Gateway forwards to rider via WebSocket**:
  {type: 'driver_location', lat, lon, timestamp, eta_min: 4, distance_m: 500},

- **Rider app updates**:
  (a) Moves driver marker on map smoothly (interpolation), (b) Recalculates ETA: distance_remaining / avg_speed, (c) Shows status: 'Driver is 500m away, arriving in 4 min'. During trip: Both rider and driver see each other's locations (bidirectional), rider sees: 'Your trip in progress', driver sees: 'Passenger onboard'. ETA calculation:

- **Every 30 seconds, Location Update Service queries Google Maps Directions API**:
  {origin: {driver_current_location}, destination: {pickup_location}, mode: 'driving'},

- **Response**:
  {distance_remaining: 2500m, duration: 8 min, route_polyline},

- **Broadcast ETA update to rider**:
  {eta_min: 8, distance_m: 2500}. Geofencing:

- **When driver within 50m of pickup**:
  calculate distance(driver_location, pickup_location) using Haversine formula,

- **If distance < 50m → trigger event**:
  {type: 'driver_arrived', driver_id, trip_id},

- **Update trip status**:
  UPDATE trips SET status='DRIVER_ARRIVED',

- **Notify rider**:
  Push notification + WebSocket message 'Your driver has arrived!',

- **Rider app shows**:
  Vibrate phone, play arrival sound, display 'Driver is here' banner. Connection management:

- **Heartbeat**:
  Client sends ping every 30 seconds, server responds with pong + {server_time}, detects stale connections,

- **Reconnect**:
  If WebSocket closes (network loss, app backgrounded), client auto-reconnects with exponential backoff (1s, 2s, 4s, 8s, max 30s),

- **Resume**:
  On reconnect, request last known state: {last_received_timestamp}, server sends missed updates. Bandwidth optimization:

- **Point 1**: Send location only if moved >10 meters (skip updates when stationary),

- **Point 2**: Reduce frequency if speed < 5 km/h (1 update per 10 sec vs 3 sec),

- **Compress updates**:
  Binary protocol (Protocol Buffers) instead of JSON (50% size reduction),

- **Batch**:
  Multiple location updates in single WebSocket frame if queued. Scaling:

- **Point 1**: Each WebSocket server handles 10K connections (1 connection per active driver + riders),

- **Point 2**: For 100K active trips (100K drivers + 100K riders = 200K connections), need 20 instances,

- **Point 3**: Load balancer with sticky sessions (IP hash) ensures rider stays on same instance for trip duration,

- **Point 4**: Redis Pub/Sub scales horizontally (multiple subscribers),

- **Kafka buffers location history for offline processing. Why WebSocket vs HTTP polling?**:
  HTTP polling: Client sends GET /location/{driver_id} every 3 seconds → 100K riders × 20 requests/min = 2M requests/min = 33K req/sec. WebSocket: Persistent connection → send update only when location changes → 100K drivers × 20 updates/min = 2M updates/min but NO HTTP overhead (no headers, no handshake) → 10x less bandwidth, <100ms latency vs 1-3 sec polling delay.

---

### Q3: How do you calculate surge pricing dynamically?

Surge pricing balances supply and demand through real-time monitoring and dynamic multipliers: Geohash grid system: Divide geographic area into 5km × 5km grid cells using geohash (6-character precision), each cell independently tracks supply/demand, example: Downtown SF = 9q8yy, Financial District = 9q8yw. Real-time monitoring (runs every 60 seconds): Background job (Surge Calculator Service) for EACH geohash:

- **Count available drivers**:
  Redis query: GEORADIUS drivers:available {cell_center_lon} {cell_center_lat} 5 km → count results, example: Downtown SF has 15 available drivers,

- **Count pending requests**:
  Database query: SELECT COUNT(*) FROM ride_requests WHERE status='PENDING' AND ST_DWithin(pickup_location, ST_MakePoint({cell_center_lon}, {cell_center_lat}), 5000) AND created_at > now() - INTERVAL '10 min' → example: 45 pending requests in last 10 minutes. Calculate demand/supply ratio: demand_ratio = pending_requests / available_drivers = 45 / 15 = 3.0. Determine surge multiplier based on ratio:

- **Point 1**: demand_ratio < 1.2 → no surge (multiplier = 1.0), plenty of drivers,

- **Point 2**: 1.2 <= demand_ratio < 2.0 → low surge (multiplier = 1.2×), slight imbalance,

- **Point 3**: 2.0 <= demand_ratio < 3.0 → medium surge (multiplier = 1.5×), moderate shortage,

- **Point 4**: 3.0 <= demand_ratio < 5.0 → high surge (multiplier = 1.8×), significant shortage,

- **demand_ratio >= 5.0 → very high surge (multiplier = 2.0-3.0×), severe shortage, max cap at 3.0×. Example calculation**:
  Downtown SF at 8 PM Friday (high demand):

- **Available drivers**:
  15,

- **Pending requests**:
  45,

- **Point 3**: demand_ratio = 3.0 → high surge,

- **Point 4**: surge_multiplier = 1.8×,

- **Base fare**:
  $15 → Surge fare: $15 × 1.8 = $27. Store in Redis: SET surge_multiplier:9q8yy 1.8 EX 120 (2 min TTL, refresh every 60 sec ensures always fresh). Rider sees surge:

- **Fare estimate API**:
  POST /v1/api/fare/estimate → fetches surge: GET surge_multiplier:{pickup_geohash} from Redis,

- **Response**:
  {estimated_fare: 27.00, base_fare: 15.00, surge_multiplier: 1.8, surge_notice: 'Fares are higher due to increased demand'},

- **Rider app displays**:
  Red/orange heatmap overlay on map showing surge areas, modal: 'High demand! Fares are 1.8× normal. Want to wait for prices to drop?',

- **Rider must explicitly accept surge**:
  Click 'Accept higher fare' button before ride request submitted. Driver incentives during surge:

- **Notify nearby offline drivers**:
  Push notification 'Earn more! High demand in Downtown SF. Surge pricing active (1.8×).' with deep link to 'Go Online' screen,

- **Bonus offers**:
  'Complete 3 rides in surge zone in next hour → $20 bonus' to increase supply. Smoothing algorithm: Prevent sudden jumps:

- **Point 1**: New surge = 0.7 × old_surge + 0.3 × calculated_surge (exponential moving average),

- **Example**:
  Old surge = 1.5×, Calculated = 2.0×, New surge = 0.7×1.5 + 0.3×2.0 = 1.05 + 0.6 = 1.65×,

- **Gradual increase feels fairer to users, prevents gaming (riders waiting for surge to drop). Historical trends**:
  

- **ML model predicts surge**:
  Train on historical data (time of day, day of week, weather, events, holidays),

- **Proactive driver positioning**:
  'High demand expected in Financial District at 5 PM. Head there now for more rides.',

- **Rider notifications**:
  'Avoid surge! Book your ride 30 min earlier tomorrow.'. Anti-gaming measures:

- **Point 1**: Surge applies only to NEW requests (pre-scheduled rides locked at original price),

- **Point 2**: Rider must accept surge before request (can't game by requesting then canceling),

- **Point 3**: Max cap 3.0× prevents price gouging during emergencies,

- **Transparent pricing**:
  Show exact breakdown (base + surge) not just total. Edge cases:

- **Special events**:
  Concert ends → 10K people requesting rides simultaneously → surge spikes to 3.0× but capped, queue system activates (first-come-first-served),

- **Low supply at night**:
  Only 5 drivers available citywide → surge high but also trigger driver incentives aggressively,

- **Weather**:
  Rain starts → demand spikes 50% → surge increases, notify off-duty drivers. Monitoring:

- **Dashboard**:
  Real-time heatmap of surge across city,

- **Alerts**:
  If surge > 2.5× for > 30 min in area → notify operations team to investigate (event? accident? driver shortage?),

- **A/B testing**:
  Test different surge algorithms (linear vs exponential) measure impact on: (a) Driver supply response, (b) Rider conversion rate, (c) Total revenue, (d) User satisfaction. Production metrics: Uber surge pricing increases driver supply by 30-40% during high demand, typical surge duration 15-45 minutes, surge reduces rider wait time by balancing supply/demand, revenue increase during surge but must balance with user retention.

---

### Q4: How do you handle geospatial queries efficiently for finding nearby drivers?

Redis Geospatial commands with indexing + PostGIS for persistent storage: Redis Geospatial indexing:

- **Data structure**:
  GEOADD drivers:available {longitude} {latitude} {driver_id} stores driver locations in sorted set with geohash encoding,

- **Under the hood**:
  Redis uses geohash (52-bit integer encoding of lat/lon) stored in sorted set, enables range queries via ZRANGE operations,

- **Example**:
  GEOADD drivers:available -122.4194 37.7749 driver_123 adds driver to San Francisco location. Query nearby drivers: GEORADIUS drivers:available -122.4194 37.7749 5 km WITHDIST ASC COUNT 10 →

- **Point 1**: Finds all drivers within 5km radius of point,

- **Point 2**: Returns drivers sorted by distance ascending,

- **Point 3**: Limits to 10 nearest drivers,

- **Includes distance in results**:
  [(driver_123, 0.8 km), (driver_456, 1.2 km), ...]. Performance:

- **Time complexity**:
  O(N + log(M)) where N = number of results returned, M = total items in sorted set,

- **For 1 million drivers, finding 10 nearest within 5km**:
  ~10ms,

- **Point 3**: Geohash encoding enables quick pruning (only scan nearby cells),

- **Memory**:
  Each driver entry ~100 bytes (driver_id + geohash), 1M drivers = ~100 MB. Precision levels:

- **Geohash character length determines precision**:
  5 chars ≈ 5km × 5km cell, 6 chars ≈ 1.2km × 600m cell, 7 chars ≈ 150m × 150m cell,

- **Use 6-char precision for driver locations (balance between accuracy and performance). Filtering by vehicle type**:
  

- **After GEORADIUS, filter results**:
  FOR EACH driver_id in results: SISMEMBER drivers:sedan {driver_id},

- **Alternative**:
  Maintain separate indexes per vehicle type: GEOADD drivers:available:sedan {lon} {lat} {driver_id}, GEOADD drivers:available:suv {lon} {lat} {driver_id},

- **Query specific type**:
  GEORADIUS drivers:available:sedan -122.4194 37.7749 5 km. Updating driver locations:

- **WebSocket receives location ping**:
  {driver_id, lat, lon},

- **Update geospatial index**:
  GEOADD drivers:available {lon} {lat} {driver_id} (atomic operation, replaces old location),

- **Time**:
  ~1ms per update,

- **With 100K drivers updating every 5 seconds = 20K updates/sec = 20K GEOADD operations/sec, Redis handles 100K ops/sec per instance. Removing drivers**:
  

- **Driver goes offline or accepts ride**:
  GEOREM drivers:available {driver_id},

- **Point 2**: Atomic removal from sorted set,

- **O(log(N)) time complexity. Persistent storage with PostGIS**:
  

- **Point 1**: PostgreSQL with PostGIS extension for geographic data types,

- **Schema**:
  CREATE TABLE driver_locations (driver_id uuid PRIMARY KEY, location geography(Point, 4326), updated_at timestamp); CREATE INDEX idx_driver_location_gist ON driver_locations USING GIST (location);,

- **Query**:
  SELECT driver_id, ST_Distance(location, ST_MakePoint(-122.4194, 37.7749)::geography) as distance FROM driver_locations WHERE ST_DWithin(location, ST_MakePoint(-122.4194, 37.7749)::geography, 5000) ORDER BY distance LIMIT 10;,

- **Point 4**: GIST index enables efficient spatial queries,

- **Performance**:
  10-50ms for 1M drivers (slower than Redis but provides persistence and complex queries). Hybrid approach (production):

- **Point 1**: Redis for real-time queries (sub-10ms latency, ephemeral data with 1-min TTL),

- **Point 2**: PostGIS for persistent storage + historical analysis,

- **Background sync**:
  Location Update Service writes to both: Redis (immediate) + Kafka → Consumer → PostgreSQL (async, 1-5 sec lag). Geographic sharding:

- **Divide world into regions**:
  US-West, US-East, EU, APAC, each region has own Redis instance,

- **Point 2**: Route requests to nearest region (latency optimization),

- **Cross-region queries rare (rider in SF won't see driver in NYC). Alternative data structures**:
  

- **QuadTree**:
  Recursively partition 2D space into 4 quadrants, efficient for sparse data,

- **R-Tree**:
  Balanced tree for multidimensional data, better for overlapping regions,

- **S2 Geometry Library (used by Uber)**:
  Google's library for spherical geometry, handles edge cases (poles, antimeridian), better accuracy for long distances. Edge cases:

- **Antimeridian crossing (±180° longitude)**:
  Geohash handles correctly, but custom logic needed for queries spanning antimeridian,

- **Poles (90° latitude)**:
  Few drivers at poles, but geohash degenerates, use special handling or different projection,

- **Very sparse areas**:
  If no drivers within 5km, increase radius to 10km, 20km (exponential backoff) until drivers found or max radius reached. Caching query results:

- **Frequent queries (e.g., airport, train station)**:
  Cache top 10 drivers for location: SET nearby_drivers:{geohash} '[driver_ids]' EX 10 (10 sec TTL),

- **Point 2**: Invalidate on driver movement,

- **Reduces load on Redis geospatial queries for hotspots. Monitoring**:
  

- **Point 1**: Track GEORADIUS query time (p50, p99),

- **Point 2**: Alert if p99 > 50ms (indicates performance degradation),

- **Point 3**: Monitor memory usage (sorted set size),

- **Track cache hit rate for hotspot locations. Production scale**:
  Uber processes millions of geospatial queries per second globally, Redis Cluster with sharding across 100+ nodes, sub-10ms latency for 99th percentile, handles 10M+ driver location updates per minute, PostGIS used for analytics and fraud detection (abnormal GPS patterns).

---

### Q5: How do you design the notification system for real-time updates to riders and drivers?

Multi-channel notification system with FCM/APN for push + WebSocket for in-app: Architecture: Notification Service consumes Kafka events (ride.matched, trip.started, driver.ride_offered) → routes to appropriate channel (push notification or WebSocket). Push notifications (FCM/APN):

- **Device token registration**:
  On app login/install → rider/driver registers device token: POST /v1/api/devices with {user_id, device_token, platform: 'ios'/'android'},

- **Store**:
  INSERT INTO device_tokens (user_id, device_token, platform, active: true, registered_at),

- **Update on token refresh (iOS rotates tokens periodically). Notification flow**:
  

- **Event published**:
  Ride Service → Kafka 'ride.matched' topic: {trip_id, rider_id, driver_id, driver_name, vehicle_model, license_plate, eta_min: 5},

- **Notification Service consumes**:
  Kafka consumer group 'notification-service' processes event,

- **Fetch device tokens**:
  SELECT device_token, platform FROM device_tokens WHERE user_id={rider_id} AND active=true (user may have multiple devices: iPhone + iPad),

- **Build payload**:
  {title: 'Driver Assigned!', body: '{driver_name} is on the way in a {vehicle_model}. ETA: 5 min', data: {trip_id, driver_id, action: 'open_trip_screen', deep_link: 'myapp://trip/{trip_id}'}},

- **Send via FCM (Android)**:
  fcm.send({registration_ids: [device_tokens], notification: {title, body}, data: {...}, priority: 'high', time_to_live: 600}),

- **Send via APN (iOS)**:
  apn.send({device_token, payload: {aps: {alert: {title, body}, sound: 'default', badge: 1}, custom_data: {...}}}). Notification types:

- **Ride matched → rider**:
  'Driver {name} accepted! ETA 5 min',

- **Ride matched → driver**:
  'New ride request from {rider_name}. Pickup at {address}',

- **Driver arrived → rider**:
  'Your driver has arrived!',

- **Trip started → rider**:
  'Your trip has started. Enjoy your ride!',

- **Trip completed → rider & driver**:
  'Trip completed. Fare: $25.50',

- **Payment processed → rider**:
  'Payment of $25.50 charged to card ending in 1234',

- **Surge active → nearby riders**:
  'High demand in your area. Fares are 1.5× higher.'. Delivery guarantees:

- **Point 1**: FCM/APN best-effort delivery (not guaranteed),

- **FCM retry**:
  Up to 4 weeks for devices offline,

- **APN retry**:
  Up to 1 day for devices offline,

- **Fallback**:
  If push fails, send SMS for critical notifications (driver assigned, payment failed). WebSocket for in-app real-time:

- **Rider in app**:
  Maintains WebSocket connection to API Gateway,

- **Gateway subscribes**:
  Redis Pub/Sub channel user:{rider_id}:notifications,

- **Notification Service publishes**:
  PUBLISH user:{rider_id}:notifications '{type: ride_matched, data: {...}}',

- **Gateway forwards to rider via WebSocket**:
  Immediate delivery (<100ms latency),

- **Rider app handles**:
  {type: 'ride_matched'} → navigate to trip screen, update UI, show driver card. Multi-device synchronization:

- **Point 1**: User has iPhone + iPad both logged in,

- **Point 2**: Both devices have WebSocket connections,

- **Notification sent to both**:
  push notification + WebSocket message,

- **When one device acknowledges (user taps notification), mark as read**:
  SET notification:{notification_id}:read true,

- **Other device receives 'notification_read' WebSocket message → dismiss duplicate. Notification preferences**:
  

- **User settings**:
  Allow push for {ride_updates: true, promotions: false, surge_alerts: true},

- **Respect preferences**:
  Notification Service checks: SELECT preferences FROM users WHERE user_id={user_id}, skip if category disabled,

- **Regulatory compliance**:
  GDPR/opt-in requirements for marketing notifications. Rich notifications:

- **iOS actionable notifications**:
  Buttons on notification (Accept/Decline for driver),

- **Android heads-up display**:
  Full-screen overlay for critical notifications (driver assigned),

- **Images**:
  Include driver photo in ride matched notification,

- **Maps**:
  Show pickup location map thumbnail. Notification analytics:

- **Track delivery rate**:
  sent / delivered (FCM provides delivery receipts),

- **Track open rate**:
  notifications delivered / notifications opened (app reports when user taps),

- **Track action rate**:
  notifications opened / action taken (e.g., accepted ride),

- **A/B test**:
  Different notification copy, timing, images → optimize engagement. Handling failures:

- **Invalid token (user uninstalled app)**:
  FCM/APN returns 'invalid token' error → UPDATE device_tokens SET active=false WHERE device_token={token},

- **Transient errors (network timeout)**:
  Retry with exponential backoff (1s, 2s, 4s, max 3 retries),

- **Service unavailable (FCM/APN down)**:
  Queue in Kafka (retention 7 days), replay when service restored,

- **Critical notifications (payment failed)**:
  If push fails → send SMS via Twilio as backup. Batching:

- **Bulk notifications (surge alert to 10K users in area)**:
  Batch send 500 tokens per API call,

- **FCM multicast**:
  Single API call with array of 1000 tokens,

- **Rate limiting**:
  FCM: 1 million messages/min, APN: no rate limit but throttle sends to avoid overwhelming,

- **Async processing**:
  Notification Service worker pool (50 threads) sends in parallel. Priority and TTL:

- **High priority**:
  ride_matched, driver_arrived (immediate delivery, wake device from sleep),

- **Normal priority**:
  trip_completed, rating_reminder (can wait if device offline),

- **Time-to-live**:
  ride_matched TTL=10 min (stale after 10 min, driver moved on), promotion TTL=24 hours. Silent notifications:

- **Background data sync**:
  Send silent push to update trip status without alerting user,

- **iOS**:
  content-available: 1 flag triggers background app refresh,

- **Android**:
  data-only message (no notification shown),

- **Use case**:
  Update driver location, sync ride history, refresh surge map. Localization:

- **Notification text in user's language**:
  Fetch: SELECT language FROM users WHERE user_id={user_id},

- **Translate**:
  title_es: '¡Conductor asignado!', title_hi: 'ड्राइवर नियुक्त!',

- **FCM/APN support localization keys**:
  Send key + parameters, device renders in user's language. Cost optimization:

- **FCM**:
  Free,

- **APN**:
  Free (but requires Apple Developer account $99/year),

- **SMS fallback**:
  $0.01-0.05 per SMS → use sparingly for critical notifications only,

- **Reduce volume**:
  Don't send promotional push during ride (user focused on trip), batch non-urgent notifications. Production scale: Uber sends 100M+ push notifications daily, delivery rate: 95-99% (devices reachable), open rate: 30-50% depending on notification type, latency: push delivery p50: 1-2 seconds, p99: 5-10 seconds (includes FCM/APN processing), WebSocket: <100ms in-app delivery.

---

## 7. Architectural Trade-Off Analysis Matrix

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Active Driver Spatial Indexing** | **Redis Geospatial (H3 / Geohash ZSET)** | Disk-based PostGIS (`ST_DWithin`) | PostGIS requires disk seeks and index tree traversals. 125,000 pings/sec would overload connection pools. Redis processes commands in RAM in $< 1\text{ ms}$. |
| **Driver Lock for Dispatch** | **Apache ZooKeeper Ephemeral Nodes** | Redis Redlock | ZooKeeper provides strict CP guarantees, linearizable leader election, and automatic node reclamation via session actuator heartbeats, preventing split-brain double bookings. |
| **Driver Communication Protocol** | **Persistent WebSocket Gateway** | HTTP Long Polling | Long polling re-negotiates TCP/TLS handshakes repeatedly, wasting mobile radio battery and bandwidth. WebSockets maintain a full-duplex lightweight socket. |
| **Trip GPS Breadcrumb Storage** | **Apache Cassandra / ScyllaDB** | Relational Database (PostgreSQL) | GPS trails are write-intensive append-only timeseries. Cassandra provides linear horizontal write scaling with clustering keys partitioned by `ride_id`. |
| **Fare Estimation Consistency** | **Eventual Consistency with Snapshot TTL** | Real-Time Dynamic Re-evaluation | Freezing the estimate for 5 minutes with a quotation token (`estimate_id`) guarantees price transparency and frictionless UX, avoiding checkout price fluctuations. |
| **Driver Dispatch Strategy** | **Sequential Waterfall ($K=5$, 10s timeout)** | Broadcast (Flash Offer to all nearby) | Broadcast leads to driver race conditions, phantom touches, driver dissatisfaction, and notification stampedes. Sequential gives each driver a fair, deterministic window. |
