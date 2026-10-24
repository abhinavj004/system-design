# System Design 02: Design Ticket Booking System (BookMyShow / Ticketmaster)

> **Source**: [Interview with Bunny - Episode 2](https://www.youtube.com/watch?v=uamf0wRMx5o)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/02](https://www.interviewwithbunny.com/systemdesign/02)  
> **Topic**: Designing a Scalable, Distributed Ticket Booking System (BookMyShow, Ticketmaster, District)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Search & Discovery**: Users can search for movies, concerts, or live events by title, genre, language, city/location, and date.
2. **View Event & Seat Map**: Users can view event details, show timings across venues/theatres, and the real-time seat layout with availability status (`AVAILABLE`, `LOCKED`, `BOOKED`).
3. **Seat Selection & Temporary Lock**: Users can select specific seats and place a temporary hold (e.g., 10-minute reservation window) while they complete the payment.
4. **Payment & Booking Confirmation**: Users can complete payment within the 10-minute window. Upon success, seats are permanently booked and an e-ticket with a QR code is generated.
5. **Automatic Lock Release**: If the user fails to complete payment or abandons the session within 10 minutes, the locked seats must automatically revert to `AVAILABLE` without manual intervention.

---

### 1.2 Non-Functional Requirements (NFR)
1. **CAP Theorem Split (Hybrid Consistency Model)**:
   - **Search & Catalog Path (AP - High Availability)**: Event browsing, movie descriptions, and theatre schedules must prioritize ultra-high availability and low read latency. Minor eventual consistency is acceptable.
   - **Booking & Seat Locking Path (CP - Strict Consistency)**: Zero double-booking. Under no circumstance can two users book the same seat for the same show.
2. **Extreme Read-Heavy System**: Read-to-write ratio is $\approx 100:1$ (millions browsing movie shows vs. thousands completing bookings).
3. **Low Latency**: Search results $< 50\text{ms}$; seat map rendering $< 100\text{ms}$; seat lock $< 200\text{ms}$.
4. **Traffic Surge Resilience**: The system must withstand catastrophic traffic spikes during high-demand events (e.g., Coldplay concerts, World Cup finals, Avengers movie drops).

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)
- **Active Users**: 100 Million Monthly Active Users (MAU).
- **Daily Events & Shows**:
  - 500 cities $\times$ 20 theatres/city $\times$ 5 screens/theatre $\times$ 4 shows/day $\approx 200,000 \text{ shows/day}$.
  - Average capacity: 200 seats/screen $\implies 40 \text{ Million total seats/day}$.
- **Read Throughput**:
  - Peak search & browse traffic $\approx 50,000 \text{ QPS}$.
- **Write Throughput (Bookings)**:
  - Average booking rate $\approx 500 \text{ bookings/sec}$ (spiking to $5,000+\text{ QPS}$ during mega-event ticket drops).
- **Architecture Takeaway**:
  - Separate read-heavy browsing microservices from high-contention booking transactional services.
  - Scale search horizontally using dedicated search indexes (**Elasticsearch**).

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+       +--------------------+       +--------------------+
|        User        |       |    Venue/Theatre   |       |    Movie / Event   |
+--------------------+       +--------------------+       +--------------------+
| id (UUID)          |       | id (UUID)          |       | id (UUID)          |
| name, email, phone |       | name, city, address|       | title, description |
+--------------------+       | total_screens (INT)|       | duration, language |
                             +--------------------+       +--------------------+
                                      |                             |
                                      | 1                           | 1
                                      | *                           | *
                             +-------------------------------------------------+
                             |                   Show / Screen                 |
                             +-------------------------------------------------+
                             | id (UUID), venue_id (FK), event_id (FK)         |
                             | start_time, end_time, screen_number             |
                             +-------------------------------------------------+
                                      | 1                           | 1
                                      | *                           | *
+--------------------+       +--------------------+       +--------------------+
|      Booking       | 1   * |     ShowSeat       | *   1 |      SeatLayout    |
+--------------------+<----->+--------------------+<----->+--------------------+
| id (UUID)          |       | id (UUID)          |       | id (UUID)          |
| user_id (FK)       |       | show_id (FK)       |       | screen_id (FK)     |
| show_id (FK)       |       | seat_id (FK)       |       | row (VARCHAR)      |
| total_amount       |       | status (ENUM)      |       | seat_number (INT)  |
| booking_status     |       | locked_at (TIME)   |       | seat_type (VIP/REG)|
+--------------------+       | lock_expires_at    |       +--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Search Events & Movies
```http
GET /api/v1/events/search?query=Oppenheimer&city=Mumbai&date=2026-10-20
Response: 200 OK
{
  "total_results": 1,
  "events": [
    {
      "event_id": "evt_101",
      "title": "Oppenheimer",
      "genre": "Biography, Drama",
      "duration_mins": 180,
      "theatres_available": 42
    }
  ]
}
```

---

### 3.2 Get Shows & Real-Time Seat Map
```http
GET /api/v1/shows/{show_id}/seats
Response: 200 OK
{
  "show_id": "show_9981",
  "venue_name": "PVR Icon, Lower Parel",
  "screen": "Screen 3 (IMAX)",
  "seat_layout": [
    { "seat_id": "s_A1", "row": "A", "num": 1, "tier": "VIP", "price": 450.00, "status": "AVAILABLE" },
    { "seat_id": "s_A2", "row": "A", "num": 2, "tier": "VIP", "price": 450.00, "status": "LOCKED" },
    { "seat_id": "s_A3", "row": "A", "num": 3, "tier": "VIP", "price": 450.00, "status": "BOOKED" }
  ]
}
```

---

### 3.3 Temporary Seat Lock (10-Minute Reservation)
```http
POST /api/v1/shows/{show_id}/seats/lock
Authorization: Bearer <jwt>
Content-Type: application/json

Request Body:
{
  "seat_ids": ["s_L12", "s_L13"]
}

Response: 200 OK
{
  "lock_id": "lck_7712",
  "seats": ["s_L12", "s_L13"],
  "status": "LOCKED",
  "lock_expires_at": "2026-10-18T18:15:00Z", // Exactly 10 minutes from now
  "payment_redirect_url": "https://payment.bookmyshow.com/checkout/lck_7712"
}
```
*Error Responses*:
- `409 Conflict`: `{"error": "SEAT_ALREADY_LOCKED", "message": "One or more selected seats are currently held by another user."}`

---

### 3.4 Confirm Booking (Post-Payment)
```http
POST /api/v1/bookings/confirm
Authorization: Bearer <jwt>

Request Body:
{
  "lock_id": "lck_7712",
  "payment_transaction_id": "txn_stripe_88192"
}

Response: 201 Created
{
  "booking_id": "bkg_4451",
  "status": "CONFIRMED",
  "seats": ["L12", "L13"],
  "qr_code_url": "https://assets.bookmyshow.com/tickets/qr_4451.png"
}
```

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/TicketBookingApplication%20(1).png?updatedAt=1746560493385)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 02](https://www.interviewwithbunny.com/systemdesign/02))*

### Component Interaction Topology
```
[ Mobile App / Web Client ]
             │
             ▼
[ CDN / Cloudflare ] (Caches movie banners, trailer media, static CSS/JS)
             │
             ▼
┌───────────────────────────────────────────────────────────────┐
│                  API Gateway & Load Balancer                  │
│       (JWT Authentication, Rate Limiting, Route Splitting)    │
└───────────────────────────────────────────────────────────────┘
          │                                  │
  (Read/Browse Traffic)             (Booking/Lock Traffic)
          │                                  │
          ▼                                  ▼
┌──────────────────┐               ┌──────────────────┐
│  Search Service  │               │ Booking Service  │
│  & Event Service │               │  (Orchestrator)  │
└──────────────────┘               └──────────────────┘
     │        │                              │
     │        ▼                              ▼
     │  ┌───────────────┐          ┌───────────────────────┐
     │  │ Elasticsearch │          │     Redis Cluster     │
     │  │ (Fuzzy Search)│          │ (Atomic Seat Locking, │
     │  └───────────────┘          │    10-Min TTL Hold)   │
     ▼                             └───────────────────────┘
┌───────────────────────┐                    │
│   Apache Cassandra    │                    │ (On Payment Success)
│ (Event Metadata, Shows│                    ▼
│   Theatres, Screens)  │          ┌───────────────────────┐
└───────────────────────┘          │ PostgreSQL / MySQL DB │
           ▲                       │ (ACID Transactions &  │
           │                       │  Confirmed Bookings)  │
           │                       └───────────────────────┘
           │ (Near Real-time Sync)           │
           │                                 ▼
           └───────── [ CDC Pipeline (Debezium / Kafka) ]
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Dual Database Strategy (Polyglot Persistence)
Bunny justifies why a single database cannot handle both search discovery and seat reservations:

1. **Read-Heavy Catalog & Show Metadata $\to$ Apache Cassandra + Elasticsearch**:
   - Movie summaries, actor lists, theatre locations, and past show schedules are write-rarely, read-constantly.
   - **Cassandra** provides linear horizontal write scaling and masterless high availability (AP).
   - **Elasticsearch** handles faceted queries (e.g., *"Show IMAX screenings of Oppenheimer in South Mumbai after 7:00 PM"*).
2. **Transactional Seat Booking $\to$ Relational DB (PostgreSQL / MySQL)**:
   - Financial orders, tickets, and confirmed bookings require strict ACID transactions and relational constraints (`UNIQUE(show_id, seat_id)`).

---

### 5.2 The Seat Locking Mechanism: DB Cron vs. Redis TTL
At `[38:15]`, Bunny walks through why traditional database-driven seat locking fails:

#### Why a Database Cron Job Fails:
- User selects seat $\to$ DB updates `status = 'LOCKED', locked_until = NOW() + 10 MIN`.
- A background Cron Job runs every 60 seconds:
  ```sql
  UPDATE show_seats 
  SET status = 'AVAILABLE' 
  WHERE status = 'LOCKED' AND locked_until < NOW();
  ```
- **The Problems**:
  1. **Latency lag**: If a user cancels after 2 minutes, other users must wait until the full 10-minute timeout + cron interval to see it available.
  2. **Severe DB churn**: Under 200,000 active shows, full-table polling every minute causes heavy I/O lock contention and degrades checkout transactions.

#### The Optimal Solution: Distributed In-Memory Lock with Redis
Instead of touching disk, the `Booking Service` leverages **Redis atomic primitives**:
```redis
# Atomically acquire lock on seat if not already locked
SET show:{show_id}:seat:{seat_id} {user_id} NX EX 600
```
- `NX`: Set only if key does not exist (prevents race condition between concurrent users).
- `EX 600`: Automatic **10-minute TTL expiration**.
- **Execution Flow**:
  1. When User 1 clicks Seat L12 & L13, Redis sets the keys atomically. Returns `OK`.
  2. If User 2 clicks Seat L12 one millisecond later, Redis returns `(nil)`. The system immediately returns `409 Conflict` to User 2.
  3. If User 1 abandons checkout, Redis automatically purges the keys at second 600. The seat becomes instantly available without any cron job or database write!

---

### 5.3 Near Real-Time Seat Map Updates (CDC + Kafka Pipeline)
How do hundreds of users viewing the seat grid see newly locked or booked seats without polling the database?
1. When a seat status changes (either locked in Redis or confirmed in PostgreSQL), an event is streamed via a **Change Data Capture (CDC)** engine (e.g., Debezium) into an **Apache Kafka** topic: `seat-status-events`.
2. A WebSocket Gateway subscribes to the Kafka topic.
3. Connected browser clients viewing that specific `show_id` receive a lightweight push notification:
   ```json
   { "event": "SEAT_STATUS_CHANGED", "show_id": "show_9981", "seat_id": "s_L12", "status": "LOCKED" }
   ```
4. The client's UI immediately turns seat L12 grey/unavailable in real time.

---

### 5.4 High-Demand Spike Handling (Virtual Waiting Room / Queue)
*The Scenario*: A Coldplay or blockbuster ticket drop causes 500,000 users to click "Book" simultaneously for 50,000 available seats.

**The Solution: Virtual Queue / Waiting Room**
```
[ Incoming 500k QPS Spike ] ──▶ [ Virtual Waiting Room (Redis Sorted Set) ]
                                              │ (Drip traffic at 500 QPS)
                                              ▼
                                 [ Booking Service & Redis Lock ]
```
1. Users entering the booking flow are placed into a **FIFO Waiting Queue** managed in a Redis Sorted Set (ranked by arrival timestamp).
2. The waiting room dispenses signed cryptographic queue passes at a calibrated rate (e.g., 500 users/second) matching the database's safe throughput.
3. When inventory hits zero, the remaining queued users are notified immediately with a *"Sold Out"* banner, shielding core booking and payment microservices from catastrophic crash.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you prevent double booking when 100 users try to book the same last seat?

Multi-layer approach with distributed locks + database pessimistic locking:

- **Redis SETNX lock BEFORE DB check**:
  FOR EACH seat: SETNX lock:seat:{show_id}:{seat_id} {hold_id} EX 30, if any returns 0 (lock exists) → abort immediately with 'Seats being held', this provides fast rejection (<10ms) before expensive DB operations.

- **If all locks acquired**:
  Query DB with pessimistic lock: SELECT seat_id, status FROM seats WHERE show_id={show_id} AND seat_id IN (...) FOR UPDATE, this row-level exclusive lock prevents concurrent transactions from reading same rows.

- **Point 3**: Validate all seats have status='AVAILABLE', if not → release Redis locks, return 'Seats not available'.

- **Create hold atomically**:
  BEGIN TRANSACTION; INSERT INTO seat_holds (...); UPDATE seats SET status='HELD', held_by={hold_id} WHERE seat_id IN (...); COMMIT;.

- **Release Redis locks**:
  DEL lock:seat:... for each seat. Why both mechanisms?: Redis provides fast concurrent rejection (100 users → 1 proceeds, 99 rejected in <10ms), PostgreSQL FOR UPDATE ensures ACID within transaction + prevents phantom reads, Redis lock TTL=30s auto-releases if service crashes (no deadlock). Example timeline: T=0ms: 100 users send request, T=5ms: User A acquires all Redis locks, Users B-Z fail Redis lock (rejected), T=50ms: User A completes DB transaction and hold creation, T=55ms: User A releases Redis locks, T=60ms: Users who retry can now attempt (but seats status='HELD' in DB). Result: Zero double bookings, fast rejection for losers, automatic recovery from crashes.

---

### Q2: Why use Redis holds with TTL instead of just database expiry?

Redis TTL provides automatic cleanup with zero overhead:

- **Performance**:
  Redis handles 100K+ ops/sec for seat availability checks vs PostgreSQL ~10K ops/sec, during ticket sale spikes (popular movie release), millions of users checking seat maps simultaneously - Redis serves from memory in <1ms vs DB 10-50ms.

- **Auto-expiry**:
  SET hold:{hold_id} {...} EX 900 automatically deletes after 15 minutes, no background job needed for Redis cleanup, contrast with DB: requires periodic sweeper job running every 1 min to find and expire holds.

- **Fast distributed lock**:
  SETNX provides atomic check-and-set in <1ms across multiple API servers, prevents race conditions during concurrent bookings.

- **Eventual consistency acceptable**:
  Redis hold expires at 15:00:00, DB sweeper runs at 15:01:30 and updates seats.status='AVAILABLE', 1-2 min lag acceptable since user sees 'Hold expired' immediately from Redis.

- **DB is still source of truth**:
  Redis serves as fast check layer, DB has authoritative state with ACID guarantees, payment confirmation always checks DB not Redis.

- **Failure handling**:
  If Redis crashes, system falls back to DB-only mode (slower but functional), if DB crashes, bookings stop but existing holds in Redis still prevent conflicts. Trade-off: Redis holds can drift from DB during network partitions, sweeper job reconciles within minutes, but final booking confirmation always validates against DB with FOR UPDATE lock. Storage: 1 hold = ~500 bytes in Redis, 10K concurrent holds = 5MB (tiny), vs DB transaction logs growing continuously. The combination provides speed (Redis) + durability (DB) + automatic cleanup (TTL).

---

### Q3: How does the CDC pipeline work for syncing Event DB to Elasticsearch?

Change Data Capture ensures search index stays fresh:

- **Source**:
  Cassandra database stores events, venues, showtimes as source of truth, admin creates/updates events via Event Service which writes to Cassandra.

- **CDC mechanism**:
  Cassandra commitlog captures all INSERT/UPDATE/DELETE operations, Debezium connector or custom CDC service tails commitlog and converts to change events, publishes to Kafka topic 'event.changes' with schema: {operation: 'insert'/'update'/'delete', table: 'events', key: event_id, data: {full_event_object}, timestamp}.

- **Kafka as buffer**:
  Provides durability and replay capability, multiple consumers can process same events, handles backpressure if Elasticsearch indexer slows down.

- **Elasticsearch indexer consumer**:
  Subscribes to 'event.changes' topic with consumer group 'es-indexer', processes events: if operation='insert' or 'update': es.index(index='events', id=event.event_id, body=event.data), if operation='delete': es.delete(index='events', id=event.event_id), uses bulk API: batches 100 events, indexes every 2 seconds for efficiency.

- **Error handling**:
  Failed indexing → retry with exponential backoff, persistent failures → push to dead letter queue (DLQ) for manual investigation, monitor consumer lag: alert if lag >1000 events.

- **Consistency**:
  Eventual consistency with typical lag 1-2 seconds (Cassandra write → Kafka publish → ES index), acceptable since search doesn't need real-time (user can wait 2s to see new movie in search).

- **Initial load**:
  On first setup, bulk export all events from Cassandra: cqlsh COPY events TO 'events.csv', bulk import to ES: curl -X POST 'localhost:9200/_bulk' with all events, then CDC maintains sync going forward.

- **Schema evolution**:
  When adding new fields to events table, update ES index mapping, reindex from Cassandra with new fields, CDC picks up changes going forward. Performance: Cassandra handles 10K writes/sec, Kafka buffers peaks, ES indexes at 5K docs/sec sustained, system can handle high event creation rate during bulk imports. Monitoring: Track Kafka consumer lag, ES indexing rate, replication lag from Cassandra, alert if lag increases beyond threshold.

---

### Q4: How do you handle payment failures and refunds?

Payment failure and refund flows with idempotency:

- **Payment timeout**:
  User confirms booking → Payment Gateway processes → webhook delayed/lost, Booking Service sets timeout: hold_id expires in 15 minutes, background job checks: SELECT hold_id, user_id FROM seat_holds WHERE status='PENDING' AND created_at < NOW() - INTERVAL '15 minutes', if payment not received → release seats: UPDATE seats SET status='AVAILABLE' WHERE held_by={hold_id}, publish 'payment.timeout' event → Notification Service sends 'Payment failed' email.

- **Payment declined**:
  Stripe webhook: POST /webhooks/payment with {status: 'failed', error: 'card_declined'}, Booking Service: (a) Idempotency check (handle duplicate webhooks), (b) Release hold: UPDATE seat_holds SET status='EXPIRED', UPDATE seats SET status='AVAILABLE', (c) Publish 'payment.failed' → notify user with reason and retry option.

- **Refund request**:
  User cancels booking: DELETE /v1/booking/{booking_id}, Service validates: (a) Booking belongs to user, (b) Cancellation policy: if show_time - NOW() > 24 hours → 100% refund, 2-24 hours → 50% refund, <2 hours → no refund, (c) Prevent duplicate cancellation: if status='CANCELLED' → return existing refund_id.

- **Process refund atomically**:
  BEGIN TRANSACTION; UPDATE bookings SET status='CANCELLED', refund_amount={calculated}, cancelled_at=NOW(); UPDATE seats SET status='AVAILABLE', booked_by=NULL, booking_id=NULL; INSERT INTO refunds (refund_id, booking_id, amount, status='PENDING'); COMMIT;

- **Async refund via Payment Gateway**:
  Call Stripe API: stripe.refunds.create({payment_intent_id, amount}), Stripe processes (takes 5-7 business days), webhook confirms: POST /webhooks/refund with {status: 'succeeded'} → UPDATE refunds SET status='COMPLETED'.

- **Idempotency everywhere**:
  Use booking_id as idempotency_key for refund API, prevents double refund if user clicks 'Cancel' multiple times, UNIQUE constraint on refunds(booking_id) ensures single refund per booking.

- **Partial refunds**:
  If cancellation policy gives 50% refund, create payment intent for 50% of original amount, track: refunds.refund_type='partial', original_amount, refunded_amount.

- **Webhook failures**:
  Stripe retries webhook 3 times with exponential backoff, if all fail → poll Stripe API: stripe.refunds.retrieve({refund_id}) to check status, reconciliation job runs daily: compares bookings DB with Stripe records, flags discrepancies for manual review. Edge cases: Concurrent cancellation + showtime started → cancellation rejected with 'Show already started', User cancels during refund processing → idempotency prevents duplicate, refund shows 'Already processing'.

---

### Q5: How do you handle popular event launches (e.g., Marvel movie premiere)?

Spike traffic handling with queue-based seat allocation:

- **Pre-sale preparation**:
  Admin creates event + showtimes 1 week before sale, pre-generates all seats: INSERT INTO seats for all shows (10K+ seats per cinema), pre-warm caches: load seat maps into Redis for all showtimes, scale infrastructure: auto-scale API servers 5x (50 → 250 instances), increase DB connection pool limits.

- **Virtual waiting room**:
  Before sale time (e.g., sale opens at 12:00 PM), users join waiting room at 11:50 AM, Cloudflare Waiting Room or custom queue manages 1M+ users, at 12:00 PM, release users in batches (1000 users/sec), each user gets token with expiry.

- **Seat reservation queue**:
  Instead of synchronous seat hold, POST /v1/booking/reserve publishes to Kafka 'seat_reservation.requests' topic: {request_id, show_id, seat_ids, user_id, timestamp}, consumer group (50 workers) processes queue at steady rate (100 reservations/sec), user sees 'In queue, position #523, estimated wait 5 min'.

- **Optimistic locking with retry**:
  Worker attempts reservation: (a) Acquire Redis lock: SETNX lock:seat:{show_id}:{seat_id}, (b) If conflict → requeue with different seats (suggest alternative nearby seats), (c) If success → create hold in DB, send confirmation to user via WebSocket: {hold_id, seats, expires_at}.

- **Rate limiting**:
  Per user: 1 hold attempt per 30 seconds (prevents spamming), Per IP: 10 requests/sec (blocks bots), CAPTCHA required before reservation (filters automated scripts).

- **Database optimization**:
  Read replicas for seat availability queries (90% read, 10% write), Primary handles only seat hold writes (strongly consistent), Sharding: shard seats table by show_id (each show on separate shard), Index optimization: INDEX on (show_id, status) for fast availability queries.

- **Cache strategy**:
  Seat map cached with very short TTL (5-10 seconds) during sale, Redis Cluster (10 nodes) handles 500K ops/sec, Cache-aside pattern: check Redis → if miss → query DB → cache for 5 sec.

- **Graceful degradation**:
  If system overloaded, disable non-critical features: turn off real-time seat map updates (show count instead), disable recommendations and personalization, show static 'High demand' message instead of exact availability.

- **Monitoring**:
  Real-time dashboard: concurrent users, queue depth, booking rate, error rate, alerts: if queue depth >10K or error rate >5%, auto-scale triggers: CPU >70% → add 50 more API servers within 2 min.

- **Post-spike cleanup**:
  After initial rush (first 10 minutes), gradually reduce infrastructure, process expired holds (sweeper job), reconcile Redis vs DB state. Real example: Avengers Endgame premiere - 2M users online, 100K bookings in first 10 minutes, queue-based system handled 1000 reservations/sec peak, 99.5% success rate (no crashes), average wait time 3 minutes. Cost: Spike infrastructure (5x normal) for 2 hours costs $500, but prevents lost revenue of $50K+ from crashes.

---

### Q6: How does seat map rendering work for large venues (1000+ seats)?

Efficient seat map rendering with caching and pagination:

- **Data structure**:
  Seats stored flat in DB: [(show_id, seat_id='A01', row='A', number=1, type='regular', status='AVAILABLE'), ...], UI needs 2D grid: [ ['A01', 'A02', 'A03', ...], ['B01', 'B02', ...] ] with status for each.

- **Seat map generation**:
  API call: GET /v1/showtime/{show_id}/seats, Service checks Redis cache: GET seat_map:{show_id}, if miss: (a) Query DB: SELECT seat_id, row, number, type, status FROM seats WHERE show_id={show_id} ORDER BY row, number, (b) Build grid: Group by row, sort by number, create 2D array with seat objects, (c) Cache in Redis: SET seat_map:{show_id} {json_grid} EX 30 (30 sec TTL), Response: { rows: [{ row: 'A', seats: [{seat_id, number, type, status, price}, ...] }, ...], summary: {available: 150, held: 20, booked: 80} }.

- **Client-side rendering**:
  React/Vue component receives grid, renders using virtual scrolling (only visible rows in DOM), Color coding: green=available, yellow=held, red=booked, grey=wheelchair/blocked, User hovers → tooltip with price and seat details, Click to select (multi-select with Shift+Click).

- **Real-time updates**:
  WebSocket connection: ws://api/showtime/{show_id}/seats, Server publishes seat status changes: {seat_id: 'A01', status: 'HELD'} when another user holds, Client updates UI without full reload, Throttled updates: batch changes every 2-3 seconds (don't send individual seat updates), Optimistic UI: mark selected seats as 'SELECTING' immediately, revert if hold fails.

- **Large venue optimization**:
  For 1000+ seats, paginate seat map: Load by section: balcony, mezzanine, orchestra separately, User selects section first → load only that section's seats (200-300 seats), Alternative: Send only seat summary initially: {type: 'regular', available: 150}, Show full seat picker only after user clicks 'Select seats'.

- **Mobile optimization**:
  Mobile app: simpler grid view with zoom/pan, Touch-friendly: larger tap targets (40px vs 20px), Progressive loading: load first 5 rows immediately, lazy load rest on scroll.

- **Cache invalidation**:
  After any hold/booking/cancellation: DEL seat_map:{show_id} in Redis, Next request regenerates fresh map from DB, During high traffic: increase TTL to 60 seconds (acceptable staleness during rush).

- **Seat type filtering**:
  User filters: 'Show only Regular seats', Backend returns filtered grid: WHERE type='regular', Frontend hides unavailable types (grey out).

- **Accessibility**:
  Screen reader support: announce seat status changes, Keyboard navigation: arrow keys to move between seats, Enter to select, Color-blind mode: use patterns + colors for status (stripes for held, solid for booked). Performance: 1000 seats = ~100KB JSON gzipped = ~20KB, loads in <200ms on 3G, Redis serves 10K requests/sec for seat maps, DB query (if cache miss) takes ~50ms with proper indexing. Real venue example: IMAX theater (450 seats) → seat map loads in 100ms, user selects 2 seats → hold created in 200ms, real-time update to other users within 3 seconds.

---

### Q7: How do you prevent bots from hoarding seats during high-demand shows?

Multi-layer bot prevention and rate limiting:

- **CAPTCHA verification**:
  Require CAPTCHA before seat selection (Google reCAPTCHA v3 with score >0.5), Challenge-response for suspicious behavior (rapid requests, unusual patterns), CAPTCHA bypass for trusted users (verified accounts with booking history).

- **Rate limiting**:
  Per user: 3 hold attempts per 10 minutes (prevents rapid seat cycling), Per IP address: 10 requests per minute (blocks bot farms), Per session: 1 active hold at a time (can't hold multiple shows simultaneously), Token bucket algorithm: burst allowed (5 requests in 1 sec) but sustained rate limited.

- **Device fingerprinting**:
  Collect browser fingerprint (canvas, WebGL, fonts, timezone), Track fingerprint + IP combination, Flag suspicious patterns: same fingerprint from multiple IPs, multiple accounts from same fingerprint.

- **Behavioral analysis**:
  Track user journey: must view event details before booking (can't directly POST /reserve), Minimum interaction time: 5 seconds on seat map before reservation (humans need time to choose), Mouse movement tracking: bots have unnatural movement patterns or no movement.

- **Account verification**:
  Email verification required before booking, Phone verification for high-value events (concerts, sports finals), Payment method pre-authorization (verify valid card before allowing holds).

- **Progressive delays**:
  First hold attempt: immediate, Second attempt (after cancellation): 30 second wait, Third attempt: 60 second wait, exponentially increasing, Penalize accounts that frequently hold and release without payment.

- **Automated bot detection**:
  Machine learning model trained on: successful vs abandoned bookings, time patterns (bots active 24/7, humans concentrated in evenings), API access patterns (bots follow predictable sequences), Flag suspicious accounts: {suspicious_score: 0.85} → require additional verification.

- **Hold limits**:
  Maximum 10 seats per user per show (prevents hoarding entire sections), Maximum 3 concurrent holds across all shows, Expired hold penalty: if hold expires 3 times in a row → 1 hour cooldown before next attempt.

- **Dynamic pricing during detection**:
  If bot activity detected (e.g., 100+ hold attempts from 50 IPs in 5 min), Temporarily increase prices by 10-20% for next batch (bots programmed for specific price points), Enable 'verified user' discount (real users with history get original price).

- **Post-booking validation**:
  After payment, verify: booking pattern normal, user IP matches payment card country, email/phone reachable (send verification code), Suspicious bookings flagged for manual review before ticket issuance.

- **Enforcement**:
  Detected bots: block IP + fingerprint + account for 24 hours, Repeat offenders: permanent ban + void bookings, Collaborate with payment processors: share bot fingerprints across platforms.

- **Honeypot seats**:
  Inject fake 'available' seats visible only to bots (hidden in HTML but not rendered), Bots selecting honeypot seats → immediate flagging and blocking, Helps identify bot scripts without affecting real users. Real example: Taylor Swift concert - Bot attacks detected: 50K reservation attempts from 500 IPs in first minute, Bot prevention activated: CAPTCHA + rate limiting + fingerprinting, Result: 95% bot requests blocked, real fans got tickets, bots wasted resources on fake seats. Trade-off: Legitimate users may experience slight friction (CAPTCHA, wait times), but prevents unfair advantage for bots, essential for high-demand events to ensure fair access.

---

### Q8: What happens if the database goes down during a booking transaction?

Database failure handling with high availability setup:

- **Architecture**:
  Primary DB (us-east-1) for all writes, Replica 1 (us-east-1, same region) with synchronous replication (lag <1 second), Replica 2 (us-west-2, cross-region) with asynchronous replication (lag <10 seconds), Health checks every 30 seconds: TCP connect + simple query (SELECT 1).

- **Failure detection**:
  If primary fails to respond to 3 consecutive health checks (90 seconds), Connection pool exhaustion (all connections timing out), Application sees connection errors or transaction rollbacks.

- **Automatic failover**:
  Orchestrator or cloud-managed service (AWS RDS, Google Cloud SQL) detects failure, Promotes Replica 1 to primary (2-3 minutes for promotion), DNS updated to point to new primary endpoint, Application reconnects automatically (connection pool retry logic).

- **During transaction failure**:
  User submits: POST /v1/booking/confirm with {hold_id, payment}, (a) Booking Service attempts: BEGIN TRANSACTION, (b) Database dies mid-transaction → connection lost, (c) Transaction automatically rolls back (no partial state), (d) Service catches exception: DatabaseConnectionError, (e) Returns 503 Service Unavailable with { error: 'Temporary service disruption, please retry', retry_after: 60 }.

- **Idempotency protection**:
  Client retries after 60 seconds with same hold_id and idempotency_key, By this time, failover complete (new primary active), Retry succeeds: SELECT * FROM bookings WHERE hold_id={hold_id}, if exists (somehow completed before failure) → return existing booking_id, if not exists → process normally with INSERT INTO bookings.

- **Redis holds during outage**:
  Seat holds in Redis continue to function, Users can still select seats and create holds (Redis remains available), Holds stored but can't be confirmed (DB required for payment), Hold TTL (15 min) automatically releases seats if DB not recovered.

- **Read queries during outage**:
  Seat availability checks: routed to read replicas (unaffected), Search, browse, view events: continue normally (served by Elasticsearch + replicas), Only bookings/payments blocked (write operations require primary).

- **Data consistency after failover**:
  Synchronous replica promoted → zero data loss (up to last committed transaction), Asynchronous replica promoted → potential loss of last 1-10 seconds of transactions, Reconciliation: check Redis holds vs DB bookings, cleanup orphaned holds.

- **Payment handling**:
  Payment Gateway already processed payment → user charged but booking not created, Solution: reconciliation job runs after recovery, finds orphaned payments: SELECT p.payment_id FROM payments p LEFT JOIN bookings b ON p.hold_id = b.hold_id WHERE b.booking_id IS NULL, creates bookings retroactively or refunds if seats no longer available.

- **User communication**:
  Real-time status: display 'System temporarily unavailable, retrying...' in UI, Notifications: email users with pending holds about temporary disruption, Compensation: offer discount or free booking fee for affected users.

- **Recovery validation**:
  After failover, verify data integrity: check seat status consistency, verify no double bookings occurred, reconcile Redis vs DB state, manual review of transactions during failure window.

- **Preventive measures**:
  Database connection pooling: gracefully handle connection losses with retries, Circuit breaker pattern: temporarily block booking attempts if failure rate >10%, Graceful degradation: show 'View only' mode during DB issues (search works, bookings disabled). Real scenario: Database primary fails at 8:00 PM (peak time), Failover triggers at 8:01:30 PM, replica promoted at 8:03 PM, Bookings resume at 8:04 PM, Impact: 3-4 minutes of booking unavailability, 20 users mid-transaction affected (all retry successfully), zero double bookings, zero lost payments (idempotency + reconciliation). Cost: Multi-region setup adds 30-40% infrastructure cost, insurance against revenue loss (1 hour outage = $100K+ lost bookings).

---

### Q9: How do you implement QR code generation for ticket verification at venue entry?

QR code generation and venue verification system:

- **QR code content**:
  Contains encrypted booking_id + show_id + user_id + timestamp, Encryption: AES-256 with secret key (only backend and venue scanners have key), Format: encrypted_data = encrypt(json_string), QR contains: {enc: base64(encrypted_data), v: 'v1'} where 'v' is version for future schema changes.

- **Generation on booking confirmation**:
  After successful payment and booking creation, Generate QR code: import qrcode, data = json.dumps({booking_id, show_id, user_id, timestamp}), encrypted = aes_encrypt(data, secret_key), qr_img = qrcode.make(encrypted), upload to S3: s3.upload_file(qr_img, bucket, f'qr_codes/{booking_id}.png'), store URL in DB: UPDATE bookings SET qr_code_url={s3_url} WHERE booking_id={booking_id}.

- **Delivery to user**:
  Email: attach QR code image + booking details, Mobile app: display QR code in 'My Tickets' section (works offline - QR stored locally), SMS: send link to web page with QR code.

- **Venue entry flow**:
  User shows QR code at entry gate, Venue staff scans with mobile app or handheld scanner, Scanner sends: POST /v1/verify/qr with {qr_data, venue_id, timestamp}, Backend decrypts: booking_data = aes_decrypt(qr_data, secret_key), Validates: (a) Decryption successful (invalid QR returns error), (b) Booking exists: SELECT * FROM bookings WHERE booking_id={booking_id}, (c) Booking confirmed: status='CONFIRMED', (d) Show hasn't happened yet: showtime > NOW(), (e) Not already checked in: checked_in_at IS NULL, (f) Correct venue: showtime.venue_id matches venue_id from request.

- **Check-in atomically**:
  BEGIN TRANSACTION; UPDATE bookings SET checked_in_at=NOW(), checked_in_venue={venue_id} WHERE booking_id={booking_id} AND checked_in_at IS NULL; if affected_rows == 0: ROLLBACK; return 'Already checked in or invalid'; COMMIT; Response: {valid: true, user_name, show_name, seat_ids, booking_id}.

- **Duplicate scan prevention**:
  UNIQUE constraint on bookings(booking_id, checked_in_at) ensures single check-in, Attempting to scan again (e.g., user tries to enter twice) → query returns checked_in_at != NULL → reject with 'Already checked in at {time}', Fraud detection: if same QR scanned at multiple venues → flag for security review.

- **Offline verification (optional)**:
  For venues with poor connectivity, embed full booking details in QR: {booking_id, user_name, show_id, seat_ids, show_time, signature}, Signature: HMAC-SHA256(booking_data, secret_key) verifies authenticity, Venue scanner validates signature offline, syncs check-ins when connectivity restored.

- **QR code security features**:
  Expiry: QR expires 2 hours after showtime (prevents reuse), One-time use: becomes invalid after successful scan, Tamper detection: any modification invalidates HMAC signature, Rate limiting: max 5 scan attempts per QR per minute (prevents brute force).

- **Multiple tickets per booking**:
  If user books 3 seats, single QR code for entire booking, Scanner shows: 'Booking for 3 seats: L12, L13, L14', Entry coordinator allows 3 people through gate, Alternative: generate separate QR per seat (more granular tracking).

- **Lost ticket handling**:
  User forgot phone or deleted email, Venue staff can look up by: name + phone + booking_id (last 4 digits), Backend validates identity: phone number matches booking, generates temporary QR code valid for 10 minutes: temp_qr = generate_temp(booking_id, expires_at=NOW()+10min), user can enter with temp QR.

- **Refund/cancellation handling**:
  If booking cancelled after QR generated, mark QR as invalid: UPDATE bookings SET qr_code_valid=false, Scanner rejects: 'This ticket has been cancelled and refunded'.

- **Analytics**:
  Track check-in metrics: on-time vs late arrivals, no-show rate (booked but not checked in), popular entry gates (load balancing), fraud attempts (invalid QRs, duplicate scans). Real example: Concert with 10K attendees, Average check-in time: 3 seconds per person, Peak entry rate: 500 people/minute (30 seconds before showtime), Zero fraud incidents (secure QR + check-in validation), 98% check-in rate (2% no-shows). Additional features: Push notification: 'Your show starts in 1 hour, have your QR ready', QR code styling: embed venue logo, colorful design (QR still scans), Wallet integration: add QR to Apple Wallet / Google Pay for quick access.

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Catalog Storage** | Apache Cassandra + Elasticsearch | Monolithic Relational DB | Highly available read scaling; sub-50ms fuzzy full-text search across thousands of theatres. |
| **Booking Storage** | PostgreSQL / MySQL | NoSQL (MongoDB / DynamoDB) | Enforces strict ACID relational integrity for financial orders and confirmed seat tickets. |
| **Seat Locking Engine** | In-Memory Redis with `NX EX 600` | Periodic Database Cron Job Polling | Eliminates heavy DB table scan churn; provides instantaneous atomic locking and zero-maintenance TTL expiry. |
| **Seat Map Sync** | CDC (Debezium) + Kafka + WebSockets | Client HTTP Polling every 2s | Reduces server network load by 95%; pushes seat state changes in near real-time. |
| **Surge Mitigation** | Virtual Waiting Room (Token Bucket Queue) | Direct Unthrottled Database Ingress | Protects booking database from thread exhaustion during massive viral ticket drops. |
