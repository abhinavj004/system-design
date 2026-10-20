# System Design 06: Design Food Delivery App (Zomato / Swiggy / Uber Eats)

> **Source**: [Interview with Bunny - Episode 6](https://www.youtube.com/watch?v=rZyAgZuuZiA)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/06](https://www.interviewwithbunny.com/systemdesign/06)  
> **Topic**: Designing a Hyperlocal Real-Time Food Delivery Platform (Zomato, Swiggy, Uber Eats)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Restaurant Discovery & Menu Browsing**: Customers can discover nearby open restaurants within a serviceable radius (e.g., 5–7 km) based on live GPS location, cuisine, price, and ratings.
2. **Order Placement & Payment**: Customers can add items to cart, customize add-ons, and place orders with online payments or Cash on Delivery.
3. **Restaurant Order Management**: Restaurants receive real-time order notifications, accept/reject orders, and update preparation milestones (`ACCEPTED`, `PREPARING`, `READY_FOR_PICKUP`).
4. **Delivery Partner Matching & Dispatch**: The system matches the order with the nearest available delivery driver and routes them to the restaurant.
5. **Real-Time Live Order Tracking**: Customers and restaurants can track the driver’s live GPS coordinates moving on an interactive map in real time.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Low Latency Live Tracking**: Driver GPS updates must reflect on the customer’s map within $< 1\text{ second}$.
2. **High Throughput Geospatial Ingestion**: Must handle 100,000+ active delivery drivers transmitting GPS coordinates every 3 to 5 seconds.
3. **High Availability for Browse Path (99.99%)**: Restaurant menus, photos, and ratings must be continuously available.
4. **Strict State Machine Consistency**: An order must transition deterministically across actors (Customer $\to$ Restaurant $\to$ Driver) without orphaned or duplicated dispatches.
5. **Eventual Consistency for Trip Path History**: Historical routes can be processed and archived asynchronously.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)
- **Active User Base**: 50 Million Monthly Active Users (MAU).
- **Daily Orders**: 2 Million orders/day (Peak order rate: $1,000 \text{ orders/sec}$ during lunch/dinner rush).
- **Active Delivery Drivers**: 100,000 drivers online concurrently during peak hours.
- **GPS Location Ingestion Throughput**:
  - Each driver pings location every 3 seconds:
    $$\text{Location Ingestion QPS} = \frac{100,000 \text{ drivers}}{3 \text{ seconds}} \approx 33,333 \text{ writes/sec}$$
  - **Core Engineering Takeaway**: Writing 33,333 GPS updates/second directly to a relational disk database (PostgreSQL/MySQL) will saturate disk I/O and crash connection pools. Location ingestion must be **100% in-memory (Redis Geospatial)** backed by **Kafka**.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|        User        |        |     Restaurant     |        |      MenuItem      |
+--------------------+        +--------------------+        +--------------------+
| id (UUID)          | 1    * | id (UUID)          | 1    * | id (UUID)          |
| name, email, phone |<------>| name, address      |<------>| restaurant_id (FK) |
| default_lat, lon   |        | latitude, longitude|        | name, description  |
+--------------------+        | is_open (BOOL)     |        | price, category    |
                              | rating (FLOAT)     |        | is_veg, in_stock   |
                              +--------------------+        +--------------------+
                                      | 1
                                      | *
+--------------------+        +--------------------+        +--------------------+
|   DeliveryAgent    | 1    * |       Order        | 1    1 |      Payment       |
+--------------------+<------>+--------------------+<------>+--------------------+
| id (UUID)          |        | id (UUID)          |        | id (UUID)          |
| name, phone, status|        | user_id (FK)       |        | order_id (FK)      |
| current_lat, lon   |        | restaurant_id (FK) |        | amount, method     |
| vehicle_type       |        | driver_id (FK, opt)|        | payment_status     |
+--------------------+        | order_status (ENUM)|        +--------------------+
                              | total_amount       |
                              +--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Nearby Restaurant Search
```http
GET /api/v1/restaurants/nearby?latitude=19.0760&longitude=72.8777&radius_km=5&cuisine=biryani
Response: 200 OK
{
  "total_results": 24,
  "restaurants": [
    {
      "restaurant_id": "rest_persian_darbar",
      "name": "Persian Darbar",
      "distance_km": 2.1,
      "estimated_delivery_mins": 35,
      "rating": 4.5,
      "is_open": true
    }
  ]
}
```

---

### 3.2 Place Order (Customer)
```http
POST /api/v1/orders
Authorization: Bearer <jwt>
X-Idempotency-Key: ord_7712_uuid

Request Body:
{
  "restaurant_id": "rest_persian_darbar",
  "items": [
    { "item_id": "item_mutton_biryani", "quantity": 1 }
  ],
  "delivery_address": { "lat": 19.0820, "lon": 72.8810, "text": "Flat 402, Sunshine Apts" }
}

Response: 201 Created
{
  "order_id": "ord_99812",
  "status": "ORDER_PLACED",
  "total_amount": 450.00
}
```

---

### 3.3 Driver GPS Location Ingestion (Heartbeat)
```http
POST /api/v1/drivers/location
Authorization: Bearer <driver_token>
Content-Type: application/json

Request Body:
{
  "driver_id": "drv_8819",
  "latitude": 19.0795,
  "longitude": 72.8790,
  "status": "AVAILABLE" // or BUSY
}

Response: 204 No Content
```

---

### 3.4 Live Order Tracking (WebSocket Handshake)
```http
GET /ws/v1/orders/{order_id}/track
Upgrade: websocket
Connection: Upgrade

Server Push Payload (Every 2 seconds):
{
  "order_id": "ord_99812",
  "status": "OUT_FOR_DELIVERY",
  "driver_location": { "lat": 19.0805, "lon": 72.8800 },
  "eta_minutes": 8
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Customer App ]           [ Restaurant Partner ]           [ Delivery Driver App ]
       │                             │                                 │
       ▼                             ▼                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    API Gateway & WebSocket Gateway Router                    │
└──────────────────────────────────────────────────────────────────────────────┘
       │                             │                                 │
       ▼                             ▼                                 ▼
┌──────────────┐              ┌──────────────┐                 ┌───────────────┐
│ Restaurant & │              │ Order State  │                 │Location Ingest│
│ Catalog Svc  │              │   Service    │                 │  Kafka Gateway│
└──────────────┘              └──────────────┘                 └───────────────┘
       │                             │                                 │
       ▼                             ▼                                 ▼
┌──────────────┐              ┌──────────────┐                 ┌───────────────┐
│Elasticsearch │              │PostgreSQL DB │                 │ Apache Kafka  │
│(Spatial 5km) │              │(ACID Orders) │                 │(GPS Events)   │
└──────────────┘              └──────────────┘                 └───────────────┘
                                     │                                 │
                                     ▼                                 ▼
                              ┌────────────────────────────────────────────────┐
                              │                 Redis Cluster                  │
                              │  - Driver Geohashes (GEOADD drivers:available) │
                              │  - Active Order Cache                          │
                              └────────────────────────────────────────────────┘
                                     │                                 │
                                     ▼                                 ▼
                              ┌──────────────┐                 ┌───────────────┐
                              │Dispatch Svc  │                 │ WebSocket Svc │
                              │(Matching V1) │                 │(Live Push Map)│
                              └──────────────┘                 └───────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 High-Velocity Driver Location Ingestion (33,000 QPS)
Writing 33,000 location pings/second directly to SQL causes lock escalation and disk saturation.
Bunny details the high-speed ingest pipeline:
1. **Direct Kafka Ingest**: Driver apps push UDP/HTTP location pings to a dedicated lightweight **Location Gateway** that directly appends to an **Apache Kafka** topic: `driver-locations`.
2. **In-Memory Spatial Index (Redis Geospatial)**:
   - A fleet of consumer workers reads from Kafka and executes Redis spatial commands:
     ```redis
     GEOADD drivers:available 72.8790 19.0795 "drv_8819"
     ```
   - Redis internally stores coordinates using **52-bit Geohash integers within a Sorted Set (ZSET)**.
   - Lookups and updates execute in $O(\log N)$ in RAM ($< 1\text{ms}$).

---

### 5.2 Delivery Partner Matching & Dispatch Algorithm
When the restaurant marks an order `PREPARING` and estimates 10 minutes to completion, the system initiates driver dispatch:
1. **Proximity Search**:
   ```redis
   GEORADIUS drivers:available 72.8777 19.0760 3 km WITHDIST WITHCOORD ASC COUNT 5
   ```
2. **Filter & Broadcast**:
   - The Dispatch Service retrieves the top 5 nearest drivers within 3 km.
   - It sends an interactive order request with a 15-second acceptance countdown timer.
3. **Atomic Claim**:
   - When Driver 1 accepts, the system executes an atomic CAS operation:
     ```redis
     SET lock:order_dispatch:ord_99812 "drv_8819" NX EX 30
     ```
   - First driver to accept secures the assignment; other drivers are immediately dismissed.
   - The driver is removed from `drivers:available` and added to `drivers:busy`.

---

### 5.3 Real-Time Live Order Tracking (WebSockets & Pub/Sub)
- When a customer opens the "Track Order" screen, the app establishes a persistent **WebSocket connection** to the **WebSocket Gateway**.
- As the assigned driver moves, their GPS coordinates are published to Redis Pub/Sub: `channel:order:ord_99812`.
- The WebSocket server instance holding the customer’s socket receives the event and streams the updated coordinates down to the client device.
- Battery & Network Optimization: Driver apps reduce ping frequency when stationary ($0 \text{ km/h}$) and increase frequency to every 3 seconds while driving.

---

### 5.4 Multi-Actor Distributed State Machine
An order flows across 3 distinct human actors with strict state transitions:

```
[ CUSTOMER ]               [ RESTAURANT ]              [ DRIVER ]
     │                           │                          │
(Place Order) ──▶ ORDER_PLACED   │                          │
                        │        │                          │
                        ▼        ▼                          │
                  RESTAURANT_ACCEPTED                       │
                        │        │                          │
                        ▼        ▼                          │
                  FOOD_PREPARING ──▶ (Trigger Dispatch)     │
                        │                                   ▼
                        ▼                             DRIVER_ASSIGNED
                  READY_FOR_PICKUP                          │
                        │                                   ▼
                        └───────────────────────────▶ FOOD_PICKED_UP
                                                            │
                                                            ▼
                                                      OUT_FOR_DELIVERY
                                                            │
                                                            ▼
                                                      DELIVERED
```
- Every state transition is written to an append-only event log in **Apache Kafka** (`order-state-transitions`) and persisted transactionally in **PostgreSQL**.
- If a restaurant rejects or times out after 3 minutes, an automated cancellation and refund flow is triggered via Kafka.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you search for nearby restaurants efficiently?

Elasticsearch with geo_distance query:

- **Index restaurants with geo_point mapping**:
  PUT /restaurants { mappings: { properties: { location: { type: 'geo_point' } } } },

- **Query**:
  POST /restaurants/_search { query: { bool: { filter: [ { geo_distance: { distance: '5km', location: { lat: 12.97, lon: 77.59 } } }, { term: { is_active: true } } ] } }, sort: [{ _geo_distance: { location: {...}, order: 'asc' } }], size: 50 },

- **Point 3**: Elasticsearch uses BKD tree internally for spatial index, O(log n) queries,

- **Returns restaurants sorted by distance in ~50ms for millions of docs. Caching**:
  Popular queries cached in Redis search:{lat}:{lng}:{radius} with TTL=5 min, serves 70% from cache <5ms. Alternative: PostgreSQL PostGIS with ST_DWithin for exact distance, Redis GEORADIUS for real-time driver locations. Hybrid: Elasticsearch for search (complex filters), Redis for hot cache, PostgreSQL for source of truth.

---

### Q2: How do you implement cart functionality with fast add/remove operations?

Redis-based cart with session management:

- **Schema**:
  HASH key cart:{user_id} with fields {restaurant_id, items: {item_id: {quantity, price, customizations}}, subtotal, tax, total, updated_at},

- **Add item**:
  HSET cart:{user_id} items:item_123 '{quantity: 2, price: 17.98, customizations: [extra_cheese]}', then recalculate total using HGETALL cart:{user_id}, update subtotal/total fields,

- **Remove item**:
  HDEL cart:{user_id} items:item_123, recalculate total,

- **Get cart**:
  HGETALL cart:{user_id} returns entire cart in <1ms,

- **TTL**:
  EXPIRE cart:{user_id} 86400 (24 hours), auto-cleanup inactive carts. Advantages:

- **Point 1**: Fast operations ~1ms vs 50ms for database,

- **Point 2**: Scales to millions of concurrent users,

- **Point 3**: Reduces DB load,

- **Session state persists across client restarts. Persistence**:
  On order placement, serialize cart to Order.items JSONB column in PostgreSQL for audit trail, then DEL cart:{user_id}. Edge case: Cart from different restaurant → prompt user 'Clear existing cart?' before adding items from new restaurant.

---

### Q3: How do you ensure exactly-once order processing and prevent double charging?

Multi-layer idempotency:

- **Client-side idempotency key**:
  Client generates UUID before submit, includes in request header X-Idempotency-Key: {uuid},

- **Database constraint**:
  Orders table has UNIQUE INDEX on idempotency_key column,

- **Order creation**:
  BEGIN TRANSACTION; SELECT * FROM orders WHERE idempotency_key = {key} FOR UPDATE; If found: ROLLBACK; return existing order_id (HTTP 200, idempotent). If not found: INSERT INTO orders (..., idempotency_key) VALUES (...); COMMIT; return new order_id (HTTP 201),

- **Payment idempotency**:
  Payment gateway (Stripe/Razorpay) uses same idempotency_key, prevents double charging even if retry,

- **Key expiry**:
  Idempotency keys valid for 24 hours, then removed. Example: User clicks 'Place Order' twice within 2s → Request 1 creates order, Request 2 finds existing order by idempotency_key, returns same order_id → user charged once. Race condition: Two concurrent requests with same key → database UNIQUE constraint ensures only one INSERT succeeds, other gets constraint violation → retry with SELECT. Alternative: Distributed lock with Redis SETNX before order creation, but database constraint simpler.

---

### Q4: How do you match drivers to orders efficiently and prevent double assignment?

Geospatial matching with distributed locks:

- **Trigger**:
  Kafka 'order.restaurant_accepted' event consumed by Driver Matching Service,

- **Find candidates**:
  Redis GEORADIUS drivers:online {restaurant_lng} {restaurant_lat} 5 km WITHDIST ASC returns nearby drivers sorted by distance,

- **Filter**:
  status='online' AND available=true AND current_orders=0, vehicle_type suitable,

- **Scoring**:
  score = (1/distance_km) × 0.4 + (rating/5) × 0.3 + acceptance_rate × 0.2 + (idle_time/60) × 0.1, select highest score,

- **Lock acquisition**:
  Redis SETNX lock:driver:{driver_id} {order_id} EX 60, if returns 1 (success) proceed, if 0 (already locked) try next driver,

- **Send request**:
  Push notification to driver 'New order: ₹50 earnings', driver has 30s to accept,

- **Acceptance**:
  Driver accepts → update order.driver_id, status='DRIVER_ASSIGNED', release lock,

- **Rejection/timeout**:
  DEL lock:driver:{driver_id}, cascade to next driver in queue. Double assignment prevention: Lock ensures only one order can claim driver at a time, even if two orders match same driver concurrently. Example: Order O1 and O2 both match Driver D1 → O1 acquires lock first → O2's SETNX fails → O2 tries D2 instead → no conflict. Alternative: Database pessimistic locking with SELECT ... FOR UPDATE on drivers table, but slower than Redis.

---

### Q5: How do you implement real-time order tracking with driver location updates?

WebSocket + Redis Pub/Sub architecture:

- **Driver location updates**:
  Driver app sends POST /v1/drivers/location every 10-20s with {driver_id, lat, lng, timestamp},

- **Update Location Service**:
  (a) Updates Redis GEOADD drivers:online {lng} {lat} {driver_id}, (b) If driver on active delivery, publish to Kafka 'driver.location.update' topic,

- **Location Consumer**:
  Consumes Kafka events, publishes to Redis Pub/Sub channel location:{order_id} with {driver_location, eta, distance_remaining},

- **WebSocket Gateway**:
  User connects WS /v1/orders/{order_id}/track, gateway subscribes to Redis Pub/Sub location:{order_id},

- **Real-time push**:
  When new location published to Redis channel, WebSocket Gateway forwards to user's WebSocket connection in <100ms,

- **Client renders**:
  Update driver marker on map, recalculate ETA, display distance. ETA calculation: Call Google Maps Distance Matrix API with {origin: driver_location, destination: delivery_address, mode: 'driving'}, returns duration considering traffic. Optimization: Batch location updates every 10s instead of every update, send only when distance changed >50m. Scaling: 100K active deliveries × 1 update/10s = 10K updates/sec, Redis Pub/Sub handles 100K msg/sec. Fallback: If WebSocket fails, client falls back to HTTP polling GET /v1/orders/{order_id}/location every 10s. Example: Driver moves from Point A to B → Location Service publishes to Redis → WebSocket Gateway forwards to user → map updates driver position → ETA recalculated '8 min remaining'.

---

### Q6: How do you handle order cancellations and refunds?

Multi-stage cancellation policy:

- **Before restaurant acceptance**:
  User can cancel free, POST /v1/orders/{order_id}/cancel → status='CANCELLED', full refund initiated,

- **After restaurant acceptance**:
  Cancellation fee applies (₹20-50 depending on time), partial refund = total - cancellation_fee,

- **After driver assignment**:
  Higher fee (₹50-100), compensates driver for time,

- **Refund processing**:
  Call Payment Service → payment_gateway.refund(payment_id, amount), status='REFUNDING', webhook confirms → status='REFUNDED',

- **Notification**:
  Push to user 'Order cancelled, ₹X refunded', email with refund receipt. Database updates: UPDATE orders SET status='CANCELLED', cancelled_by='user', cancelled_at=now(), refund_amount={amount}, refund_status='PROCESSING' WHERE order_id={id}. Kafka event: Publish 'order.cancelled' → Restaurant Service notifies restaurant, Driver Service releases driver if assigned. Edge cases:

- **Food already prepared**:
  No refund, but offer credit for future order,

- **Driver already picked up**:
  Cannot cancel, must receive delivery,

- **Payment failed originally**:
  No refund needed, just mark cancelled. Abuse prevention: Track user cancellation_rate, if >30% in last 10 orders, add 5 min waiting period before allowing next order. Example: User cancels 2 min after ordering → restaurant not yet accepted → full refund ₹26.17 → refund appears in wallet/card in 5-7 days.

---

### Q7: What's your strategy for handling peak traffic during lunch/dinner hours?

Multi-tier scaling strategy:

- **Auto-scaling**:
  Kubernetes HPA scales API servers based on CPU/memory (threshold 70%), pre-warm 30 min before peak (11:30 AM, 6:30 PM), scale from 20 to 100 instances,

- **Database connection pooling**:
  Maintain 100 connections per API server (10K total), prevents connection exhaustion, use pgBouncer for connection management,

- **Read replicas**:
  Route read queries (restaurant search, order history) to 5 PostgreSQL replicas, writes to primary only,

- **Redis cluster**:
  Shard cart data across 10 Redis nodes using hash(user_id), each node handles 10K concurrent users,

- **Kafka buffering**:
  Order placement events buffered in Kafka, consumers process at steady rate even if spike in orders, prevents overwhelming downstream services,

- **CDN caching**:
  Restaurant images, menu items cached at edge, reduces origin load by 95%,

- **API rate limiting**:
  100 req/min per user with token bucket algorithm, prevents single user from overwhelming system,

- **Database query optimization**:
  Materialized views for popular queries (top restaurants per area), refresh every 5 min,

- **Async processing**:
  Non-critical tasks (email receipts, analytics) processed async via Kafka, doesn't block order flow. Peak load: 100K orders/hour = 1.67K orders/min = 28 orders/sec. With scaling: 100 API instances × 100 orders/sec capacity = 10K orders/sec headroom. Monitoring: Prometheus + Grafana for real-time metrics, alerts if latency >500ms or error rate >1%, auto-scale triggers.

---

### Q8: How do you ensure data consistency across multiple microservices?

Event-driven architecture with saga pattern:

- **Point 1**: Order Service is orchestrator, manages distributed transaction,

- **Order placement saga**:
  Step 1: Create order in Order DB (PENDING_PAYMENT), Step 2: Call Payment Service → if success proceed, if fail rollback (mark order FAILED), Step 3: Publish 'order.paid' to Kafka, Step 4: Restaurant Service updates inventory → if out of stock, compensating transaction (refund, cancel order), Step 5: Driver Matching assigns driver → if no drivers available, retry with backoff or cancel order.

- **Eventual consistency**:
  Each service has its own database (database-per-service pattern), updates propagated via Kafka events,

- **Compensating transactions**:
  If downstream step fails, previous steps reversed via 'order.cancelled' event → Payment Service refunds, Restaurant Service restores inventory.

- **Idempotency**:
  All event handlers idempotent (use event_id to deduplicate), prevents duplicate processing if event replayed.

- **Outbox pattern**:
  Order Service writes order + event to same database in single transaction, background job publishes events to Kafka → ensures order creation and event publishing are atomic. Example: User orders last pizza → Order created → Payment succeeds → Inventory check fails (out of stock) → Compensating transaction: refund payment, cancel order, notify user 'Item unavailable, refund processed'. Data consistency levels:

- **Strong**:
  Payments, inventory (use ACID transactions),

- **Eventual**:
  Search index, analytics (sync async via Kafka). Trade-off: Strong consistency = slower, limited scalability; Eventual = faster, more scalable, acceptable for most use cases.

---

### Q9: How do you implement search with multiple filters (cuisine, rating, delivery time)?

Elasticsearch with compound boolean query:

- **Index mapping**:
  PUT /restaurants { mappings: { properties: { name: {type: 'text'}, location: {type: 'geo_point'}, cuisine: {type: 'keyword'}, rating: {type: 'float'}, avg_delivery_time: {type: 'integer'}, price_range: {type: 'keyword'}, is_active: {type: 'boolean'} } } },

- **Search query**:
  POST /restaurants/_search { query: { bool: { must: [ { match: { name: 'pizza' } } ], filter: [ { geo_distance: { distance: '5km', location: { lat: 12.97, lon: 77.59 } } }, { terms: { cuisine: ['Italian', 'American'] } }, { range: { rating: { gte: 4.0 } } }, { range: { avg_delivery_time: { lte: 45 } } }, { term: { is_active: true } } ] } }, sort: [{ rating: 'desc' }], aggs: { cuisine_facets: { terms: { field: 'cuisine', size: 20 } }, price_facets: { terms: { field: 'price_range' } } }, size: 50 },

- **Query execution**:
  (a) Full-text match on 'pizza' → 10K candidates, (b) Proximity filter → 2K, (c) Cuisine filter → 500, (d) Rating + delivery time → 100 results,

- **Aggregations**:
  Returns facet counts {Italian: 50, Chinese: 30, Mexican: 20} for UI filters. Sorting:

- **Default**:
  Sort by distance (closest first),

- **Rating**:
  Sort by rating DESC,

- **Delivery time**:
  Sort by avg_delivery_time ASC,

- **Popularity**:
  Sort by total_orders DESC. Pagination: Use from/size for first 10K results, search_after for deep pagination (>10K). Performance: ~50ms p95 for 100M restaurants with 5 filters. Caching: Cache popular filter combinations in Redis with TTL=5 min. Example: User searches 'pizza' near SF, filters: Italian, rating ≥4, delivery ≤45 min → Elasticsearch returns 50 restaurants + facets → user sees results with applied filters + counts for refinement.

---

### Q10: How do you calculate delivery ETA accurately?

Multi-factor ETA prediction:

- **Initial estimate**:
  When order placed, estimate = restaurant_prep_time + travel_time_to_restaurant + travel_time_to_customer,

- **Restaurant prep time**:
  Use historical avg_prep_time from restaurant, adjust for current load (busy hours +10 min),

- **Travel time**:
  Call Google Maps Distance Matrix API with {origin: driver_current_location, destination: restaurant_location, mode: 'driving', departure_time: 'now'}, returns duration considering real-time traffic,

- **Dynamic updates**:
  As order progresses, recalculate ETA every 1-2 min, formula: ETA = (distance_remaining / driver_speed) + traffic_adjustment,

- **Machine learning**:
  Train model on historical data (features: time_of_day, day_of_week, weather, distance, restaurant, driver), predict ETA more accurately than static calculation,

- **Traffic data**:
  Integrate with Google Maps Traffic API, adjust for congestion (red roads +50% time),

- **Driver speed**:
  Calculate from last 5 location updates, driver_speed = distance / time, use for real-time ETA. Display: Show range 'Arrives in 25-35 min' instead of exact '30 min' to manage expectations. Update frequency: Recalculate every 60s during preparation, every 10s during delivery. Accuracy target: ±3 min from actual delivery time. Example: Order placed 12:00 PM → initial ETA: prep_time 20 min + travel 15 min = 35 min (arrive 12:35 PM) → at 12:15 driver assigned, recalc: travel 12 min = arrive 12:32 PM → at 12:25 driver picked up, recalc with actual speed and traffic: 10 min = arrive 12:35 PM → actual delivery 12:33 PM (within 2 min of estimate).

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Location Ingestion** | Kafka Gateway $\to$ Redis Geospatial (`GEOADD`) | Synchronous Relational DB Writes | Absorbs 33,000+ writes/sec entirely in memory without saturating disk I/O. |
| **Proximity Matching** | Redis `GEORADIUS` / Geohash ZSET | PostGIS Geospatial SQL Queries | Delivers sub-millisecond proximity queries to match drivers within a 3km radius. |
| **Live Map Tracking** | WebSocket Gateway + Redis Pub/Sub | Client HTTP Polling every 2 seconds | Eliminates HTTP header overhead; pushes driver GPS coordinates to customer maps with $< 1\text{s}$ latency. |
| **Order State Machine** | PostgreSQL ACID + Kafka Event Sourcing | Monolithic 2PC across microservices | Enforces strict consistency for order payments while allowing asynchronous dispatch and notifications. |
| **Restaurant Search** | Elasticsearch (Fuzzy + 5km Radius) | Relational SQL `WHERE city = ...` | Provides sub-50ms text search across food items combined with spatial delivery radius boundaries. |
