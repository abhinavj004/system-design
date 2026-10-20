# System Design 05: Design Hotel Booking App (Airbnb / Booking.com)

> **Source**: [Interview with Bunny - Episode 5](https://www.youtube.com/watch?v=mH2Ye3_vErw)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/05](https://www.interviewwithbunny.com/systemdesign/05)  
> **Topic**: Designing a Global Hotel & Vacation Rental Booking System (Airbnb, Booking.com)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Proximity & Faceted Search**: Users can search for hotels/properties by geographical location (city, GPS coordinates, radius), date range (`check_in`, `check_out`), guest count, price, star rating, and amenities (Wi-Fi, pool, pet-friendly).
2. **Hotel & Room Details**: Users can view property descriptions, room categories (Deluxe, Executive, Suite), high-resolution photos, reviews, and dynamic nightly rates.
3. **Real-Time Room Availability**: The system dynamically displays whether a specific room type is available for the entire requested date range.
4. **Room Reservation & Booking**: Users can place a temporary 10-minute hold on a room and complete payment to confirm the reservation.
5. **Booking Management & Cancellation**: Users can view their upcoming/past reservations and cancel bookings (with automated refund processing and inventory restoration).

---

### 1.2 Non-Functional Requirements (NFR)
1. **CAP Theorem Split (Hybrid Architecture)**:
   - **Search & Browse Path (AP - High Availability)**: Search queries and listing views must prioritize $< 100\text{ms}$ latency and 99.99% availability.
   - **Booking & Inventory Path (CP - Strict Consistency)**: **Zero Double-Booking**. A room cannot be booked by two different guests for overlapping date ranges.
2. **High Read-to-Write Ratio**: Read-heavy traffic ($\approx 100:1$ ratio of search/browse requests to completed bookings).
3. **Geospatial Latency**: Proximity search within a 10–50 km radius must execute in $< 50\text{ms}$.
4. **Idempotency & Concurrency**: Network retries or parallel booking requests must never double-book rooms or double-charge payment instruments.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)
- **Scale**: 5 Million Hotel Properties / Listings globally.
- **Average Rooms per Property**: 20 rooms per property $\implies 100 \text{ Million total bookable rooms}$.
- **Daily Search QPS**:
  - 10 Million DAU $\times$ 10 searches/day $\approx 100 \text{ Million searches/day} \approx 1,200 \text{ QPS}$ (Peaking at $10,000 \text{ QPS}$).
- **Daily Booking QPS**:
  - 500,000 bookings/day $\approx 6 \text{ bookings/sec}$ (Peaking at $100 \text{ bookings/sec}$).
- **Inventory Storage (365-Day Projection)**:
  - Tracking availability per room type per day for 2 years:
  - 10M room types $\times$ 730 days $\approx 7.3 \text{ Billion inventory rows}$.
  - At 50 bytes/row $\approx 365 \text{ GB}$ (very manageable on partitioned relational databases).

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|        User        |        |   Hotel/Property   |        |      RoomType      |
+--------------------+        +--------------------+        +--------------------+
| id (UUID)          | 1    * | id (UUID)          | 1    * | id (UUID)          |
| name, email, phone |<------>| name, description  |<------>| hotel_id (FK)      |
+--------------------+        | latitude, longitude|        | name (Suite/Deluxe)|
                              | address, city      |        | base_price (DECIMAL|
                              | rating, amenities  |        | total_inventory(INT|
                              +--------------------+        +--------------------+
                                                                     | 1
                                                                     | *
+--------------------+        +--------------------+        +--------------------+
|      Booking       | 1    * |     DailyRoom      | *    1 |   RoomInventory    |
+--------------------+<------>|     Inventory      |<------>|     (Aggregate)    |
| id (UUID)          |        +--------------------+        +--------------------+
| user_id (FK)       |        | id (UUID)          |        | room_type_id (FK)  |
| hotel_id (FK)      |        | room_type_id (FK)  |        | date (DATE)        |
| room_type_id (FK)  |        | date (DATE)        |        | total_rooms (INT)  |
| check_in (DATE)    |        | total_rooms (INT)  |        | reserved_rooms(INT)|
| check_out (DATE)   |        | booked_rooms (INT) |        +--------------------+
| status (CONFIRMED) |        +--------------------+
+--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Proximity & Filtered Search
```http
GET /api/v1/hotels/search?latitude=48.8584&longitude=2.2945&radius_km=10&check_in=2026-06-10&check_out=2026-06-14&guests=2&min_price=150&amenities=wifi,pool
Response: 200 OK
{
  "total_results": 184,
  "hotels": [
    {
      "hotel_id": "htl_paris_eiffel_01",
      "name": "Hotel Eiffel View Paris",
      "distance_km": 1.2,
      "rating": 4.7,
      "price_per_night": 220.00,
      "thumbnail_url": "https://cdn.airbnb.com/hotels/eiffel_01.jpg",
      "available_room_types": ["Deluxe Queen", "Executive Suite"]
    }
  ]
}
```

---

### 3.2 Get Available Rooms for a Hotel
```http
GET /api/v1/hotels/{hotel_id}/rooms?check_in=2026-06-10&check_out=2026-06-14
Response: 200 OK
{
  "hotel_id": "htl_paris_eiffel_01",
  "rooms": [
    {
      "room_type_id": "rt_deluxe_queen",
      "name": "Deluxe Queen Room",
      "price_per_night": 220.00,
      "total_price_4_nights": 880.00,
      "available_count": 3
    }
  ]
}
```

---

### 3.3 Temporary Room Reservation (10-Minute Lock)
```http
POST /api/v1/bookings/reserve
Authorization: Bearer <jwt>
Content-Type: application/json

Request Body:
{
  "hotel_id": "htl_paris_eiffel_01",
  "room_type_id": "rt_deluxe_queen",
  "check_in": "2026-06-10",
  "check_out": "2026-06-14",
  "num_rooms": 1
}

Response: 201 Created
{
  "reservation_id": "res_88192",
  "expires_at": "2026-10-18T18:15:00Z", // 10 minutes TTL
  "total_amount": 880.00,
  "payment_redirect_url": "https://pay.airbnb.com/checkout/res_88192"
}
```

---

### 3.4 Confirm Booking
```http
POST /api/v1/bookings/{reservation_id}/confirm
Authorization: Bearer <jwt>

Request Body:
{
  "payment_intent_id": "pi_stripe_77192a"
}

Response: 200 OK
{
  "booking_id": "bkg_00192",
  "status": "CONFIRMED",
  "hotel_name": "Hotel Eiffel View Paris",
  "check_in": "2026-06-10",
  "check_out": "2026-06-14"
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Mobile / Web Clients ]
           │
           ▼
[ Global CDN (CloudFront) ] (Caches hotel photos, room descriptions, reviews)
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
└─────────────────────────────────────────────────────────────┘
          │                                           │
  (Search & Browse Traffic)                   (Booking Traffic)
          │                                           │
          ▼                                           ▼
┌──────────────────┐                         ┌──────────────────┐
│  Search Service  │                         │ Booking Service  │
│ (Proximity Engine│                         │  (Orchestrator)  │
└──────────────────┘                         └──────────────────┘
     │        │                                       │
     │        ▼                                       ▼
     │  ┌───────────────┐                   ┌───────────────────┐
     │  │ Elasticsearch │                   │   Redis Cluster   │
     │  │(GeoPoint +    │                   │(10-Min Distributed│
     │  │ Facet Filters)│                   │    Room Locks)    │
     │  └───────────────┘                   └───────────────────┘
     ▼                                                │
┌───────────────────────┐                             ▼
│   Apache Cassandra    │                   ┌───────────────────┐
│(Hotel Metadata, Photos│                   │ PostgreSQL / MySQL│
│  Amenities, Reviews)  │                   │(ACID Reservations,│
└───────────────────────┘                   │  Daily Inventory) │
           ▲                                └───────────────────┘
           │                                          │
           │ (Near Real-time Inventory Update)        │
           └───────── [ CDC Pipeline (Debezium / Kafka) ]
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Proximity Search: Why Elasticsearch Beats Raw QuadTree/Geohash
At `[44:09]`, Bunny clarifies why hotel search cannot rely on spatial indices alone:
- **The Problem with Raw Geospatial Indexing (QuadTree / Geohash)**:
  - QuadTree or Uber H3 are fantastic at answering: *"Find points within 5 km of (lat, lon)"*.
  - But hotel queries are **heavily faceted**:
    > *"Find hotels within 10 km of Central Paris that have Free Breakfast, Swimming Pool, Rating $\ge 4.5$, and Price $<\$250$."*
  - Performing a spatial query in one database and joining with complex attributes in another causes high latency and massive network overhead.
- **The Solution: Elasticsearch with `geo_point`**:
  - Elasticsearch indexes geographical coordinates as `geo_point` types alongside standard string, boolean, and numeric fields.
  - A single Elasticsearch compound query combines:
    1. **`geo_distance` filter**: Filters documents within `radius_km`.
    2. **Faceted filters**: Rating, price range, amenities.
    3. **Full-text matching**: Hotel or neighbourhood names.
  - Returns ranked results in $< 30\text{ms}$.

---

### 5.2 Room Availability Data Model: The Daily Inventory Table
*Why the naive approach fails*:
Querying overlapping date ranges on the main `bookings` table:
```sql
-- NAIVE APPROACH (Slow, causes table locks and scans)
SELECT COUNT(*) FROM bookings 
WHERE hotel_id = :h_id AND room_type_id = :r_id
  AND check_in < :user_checkout AND check_out > :user_checkin;
```
Under millions of bookings, this query requires full-table index scans and cannot guarantee concurrency without heavy table locks.

#### Bunny's Solution: Pre-Aggregated Daily Inventory Table
Instead of querying raw bookings, maintain a **`room_inventory`** table with 1 row per room type per day:

```sql
CREATE TABLE room_inventory (
    room_type_id UUID NOT NULL,
    date DATE NOT NULL,
    total_rooms INT NOT NULL,
    booked_rooms INT NOT NULL DEFAULT 0,
    PRIMARY KEY (room_type_id, date),
    CONSTRAINT chk_inventory CHECK (total_rooms >= booked_rooms)
);
```

#### Atomic Booking Execution across Date Ranges:
When a guest books a room from **June 10 to June 13 (3 nights)**:
```sql
UPDATE room_inventory
SET booked_rooms = booked_rooms + 1
WHERE room_type_id = :room_type_id
  AND date IN ('2026-06-10', '2026-06-11', '2026-06-12')
  AND (total_rooms - booked_rooms) >= 1;
```
- If the query returns **3 rows updated**, all nights are successfully secured!
- If it returns $< 3$ rows, at least one night is sold out $\implies$ transaction aborts immediately with zero partial bookings.

---

### 5.3 Concurrency Control: Redis Distributed Lock (10-Min Hold)
To prevent two guests from simultaneously attempting to book the same room type during the checkout payment step:
1. When the guest clicks "Reserve", the `Booking Service` creates a distributed lock in **Redis**:
   ```redis
   SET lock:hotel:{h_id}:room:{r_id}:date:{date} {user_id} NX EX 600
   ```
2. If another user attempts to lock overlapping dates, Redis returns `(nil)` $\implies$ returns `409 Conflict`.
3. If the user completes payment within 10 minutes $\to$ the permanent `room_inventory` table in PostgreSQL is incremented, and the Redis lock is released.
4. If the user closes the tab $\to$ Redis automatically expires the key at minute 10, instantly releasing the room without any manual background cleanup.

---

### 5.4 Near Real-Time Availability Sync (CDC Pipeline)
- When inventory in PostgreSQL reaches `booked_rooms == total_rooms`, a **Debezium CDC connector** streams a `RoomSoldOutEvent` to an **Apache Kafka** topic.
- A consumer updates the corresponding property in **Elasticsearch**, ensuring that guests searching for those dates will immediately see the property filtered out or marked *"Sold Out"*.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you prevent double booking (two users booking same room for overlapping dates)?

Multi-layer approach with distributed locks:

- **Redis distributed lock BEFORE availability check**:
  SETNX lock:hotel:{hotel_id}:room:{room_id}:{check_in}:{check_out} {booking_attempt_id} EX 30, lock key includes FULL date range,

- **Point 2**: If SETNX returns 0 (lock exists), return 409 Conflict 'Room being booked, try again',

- **Point 3**: If SETNX returns 1 (lock acquired), proceed to database,

- **Double-check availability with pessimistic database lock**:
  SELECT status FROM room_availability WHERE room_id={room_id} AND date BETWEEN {check_in} AND {check_out} FOR UPDATE (row-level lock),

- **Point 5**: If ANY date shows status != 'Available', release Redis lock, return 'Not available',

- **If all available**:
  BEGIN TRANSACTION; INSERT booking; UPDATE room_availability SET status='Booked', booking_id={id} WHERE room_id={room_id} AND date BETWEEN {dates}; COMMIT;,

- **Release Redis lock**:
  DEL lock:...,

- **Lock auto-expires after 30s if service crashes, prevents deadlock. Why both locks?**:
  Redis is fast (1-2ms) for early rejection, PostgreSQL FOR UPDATE ensures ACID guarantees within transaction. Example race: User A and B both try booking room_123 for Feb 1-5 → User A's SETNX succeeds → User B's SETNX fails immediately → User B sees 'being booked' → User A completes booking → User B can retry and see 'not available' → no double booking.

---

### Q2: How do you implement hotel search with location, dates, and price filters efficiently?

Elasticsearch with geo_distance + compound filters:

- **User searches**:
  location='Paris', checkIn='2025-03-01', checkOut='2025-03-05', priceMax=200, guests=2,

- **Point 2**: Geocode location to (48.8566, 2.3522) using Google Geocoding API or cached mappings,

- **Query Elasticsearch**:
  POST /hotels/_search { 'query': { 'bool': { 'filter': [ { 'geo_distance': { 'distance': '20km', 'location': { 'lat': 48.8566, 'lon': 2.3522 } } }, { 'range': { 'price_range.min': { 'lte': 200 } } }, { 'terms': { 'amenities': ['WiFi'] } } ] } }, 'sort': [ { '_geo_distance': { 'location': {...}, 'order': 'asc' } } ], 'size': 50 },

- **NOTE**:
  Elasticsearch returns hotels matching location + static filters only, does NOT check date availability (too complex for search index),

- **For availability**:
  Separate API call GET /v1/hotels/{hotel_id}/availability?dates=... after user selects hotel, queries room_availability table,

- **Alternative overcomplicated**:
  Store availability bitmap in ES (not recommended - high write volume, inconsistency risk). Why this approach?: (a) ES is optimized for text + geo search, not dynamic availability, (b) Availability changes frequently (every booking), reindexing ES on every booking = high latency + cost, (c) Better UX: Show all hotels nearby, then check availability on-demand. Caching: Cache popular searches in Redis with TTL=10min, invalidate on hotel metadata changes only. CDC: PostgreSQL hotel updates → Kafka → ES indexer keeps search index fresh within 1-2s.

---

### Q3: How do you ensure payment idempotency (prevent double charging if user clicks pay twice)?

Idempotency at multiple layers:

- **Point 1**: Client generates idempotency_key = booking_id before payment,

- **Payment Service**:
  POST /payments with {booking_id, amount, idempotency_key: {booking_id}},

- **Point 3**: Payment Gateway (Stripe/Razorpay) uses same idempotency_key for createPaymentIntent API,

- **Gateway guarantees**:
  Same idempotency_key within 24h = return same payment_intent_id, no duplicate charge,

- **Database constraint**:
  UNIQUE INDEX on payments(booking_id) ensures single payment record per booking,

- **Webhook handling**:
  POST /webhooks/payment with {booking_id, payment_id, status: 'success'}, (a) Check: SELECT * FROM payments WHERE booking_id = {booking_id}, if exists return 200 OK (already processed), (b) If not: BEGIN TRANSACTION; INSERT INTO payments (payment_id, booking_id, amount, status='SUCCESS'); UPDATE bookings SET status='CONFIRMED', payment_id={payment_id}; COMMIT;, (c) Database constraint prevents race condition: Two concurrent webhooks → first INSERT succeeds, second fails on UNIQUE constraint → handled gracefully.

- **Kafka event published only once due to database constraint. Example**:
  User double-clicks 'Pay' → Request 1 creates payment_intent_123 → Request 2 with same idempotency_key → Gateway returns existing payment_intent_123 → user charged once. Alternative: Use distributed lock SETNX lock:payment:{booking_id} before payment, but gateway idempotency + DB constraint simpler and more reliable.

---

### Q4: How do you handle room availability updates and booking modifications?

Availability updates flow:

- **Hotel admin updates availability**:
  PUT /v1/hotels/{hotel_id}/rooms/{room_id}/availability with {dates: ['2025-03-01', '2025-03-02'], status: 'Maintenance'},

- **Service validates**:
  Cannot change status if date already booked (status='Booked'),

- **Bulk update**:
  UPDATE room_availability SET status={status} WHERE room_id={room_id} AND date IN ({dates}) AND status != 'Booked',

- **Publish Kafka event**:
  'availability.updated' for cache invalidation,

- **Cache invalidation**:
  Clear Redis cached availability for affected hotel/room. Booking modification (date change):

- **User modifies**:
  PUT /v1/bookings/{booking_id} with {new_check_in, new_check_out},

- **Service validates**:
  (a) Check cancellation policy (some bookings non-refundable), (b) Check new dates available,

- **Acquire distributed lock**:
  SETNX lock:hotel:{hotel_id}:room:{room_id}:{new_check_in}:{new_check_out},

- **Check new availability**:
  SELECT status FROM room_availability WHERE room_id={room_id} AND date BETWEEN {new_dates} FOR UPDATE,

- **If available**:
  BEGIN TRANSACTION; (a) Release old dates: UPDATE room_availability SET status='Available', booking_id=NULL WHERE booking_id={booking_id}, (b) Book new dates: UPDATE room_availability SET status='Booked', booking_id={booking_id} WHERE room_id={room_id} AND date BETWEEN {new_dates}, (c) Update booking: UPDATE bookings SET check_in={new_check_in}, check_out={new_check_out}, amount={new_amount} WHERE booking_id={booking_id}; COMMIT;,

- **Point 6**: Release lock,

- **Publish 'booking.modified' event. Price adjustment**:
  If new dates have different pricing, calculate difference and either charge extra or refund. Edge cases:

- **New dates more expensive**:
  Charge difference via payment gateway,

- **New dates cheaper**:
  Issue partial refund,

- **No availability**:
  Return error with alternative dates suggestions.

---

### Q5: How do you implement the review and rating system with aggregation?

Review submission and aggregation:

- **User submits review**:
  POST /v1/reviews with {hotel_id, booking_id, rating: 4.5, comment: 'Great location!', images: [file1, file2]},

- **Validation**:
  (a) Check booking exists and belongs to user: SELECT * FROM bookings WHERE booking_id={booking_id} AND user_id={user_id}, (b) Check booking completed (check_out < today), can't review before stay, (c) Check no existing review: SELECT * FROM reviews WHERE booking_id={booking_id}, only one review per booking,

- **Upload images to S3**:
  Multi-part upload → s3://reviews/{review_id}/{image_id}.jpg, get CDN URLs,

- **Insert review**:
  INSERT INTO reviews (review_id, hotel_id, user_id, booking_id, rating, comment, images[], created_at, helpful_count: 0),

- **Background aggregation job (async)**:
  (a) Calculate new avg: UPDATE hotels SET avg_rating = (SELECT AVG(rating) FROM reviews WHERE hotel_id={hotel_id}), review_count = (SELECT COUNT(*) FROM reviews WHERE hotel_id={hotel_id}) WHERE hotel_id={hotel_id}, (b) Can use materialized view for better performance: CREATE MATERIALIZED VIEW hotel_ratings AS SELECT hotel_id, AVG(rating), COUNT(*) FROM reviews GROUP BY hotel_id, refresh every 10 min,

- **CDC pipeline**:
  hotel.avg_rating update → Kafka → Elasticsearch indexer updates search index,

- **Moderation**:
  ML service (AWS Comprehend / Cloud NLP) scans review text for: profanity, spam, fake reviews, flags for manual review if confidence >0.8,

- **Helpful votes**:
  Users can vote reviews helpful: POST /v1/reviews/{review_id}/helpful, increments helpful_count for ranking. Display: GET /v1/hotels/{hotel_id}/reviews?sortBy=helpful&page=1&size=20, sorted by helpful_count DESC or created_at DESC. Alternative aggregation: Use Apache Flink for real-time rating updates, but materialized view + periodic refresh simpler and sufficient.

---

### Q6: How do you handle time zones and date calculations for global hotels?

Time zone handling strategy:

- **Database storage**:
  ALL timestamps stored in UTC in PostgreSQL (timestamp without timezone type),

- **Hotels table**:
  Add timezone column: hotel_timezone varchar

- **Point 50**: (e.g., 'America/New_York', 'Europe/London') using IANA tz database,

- **Check-in/check-out dates**:
  Stored as DATE type (not timestamp), represents local date at hotel location,

- **Business logic**:
  When user searches, dates are in hotel's local timezone: User searches 'Paris hotels for March 1-5' → interpreted as Paris local dates, not user's timezone,

- **Display to user**:
  Backend returns dates + hotel_timezone: {check_in: '2025-03-01', check_out: '2025-03-05', timezone: 'Europe/Paris'}, client displays in local terms: 'Checking in March 1, 2025 (Paris time)',

- **Availability calculation**:
  room_availability.date is in hotel local date, no timezone conversion needed for availability checks,

- **Payment deadlines**:
  Stored as UTC timestamp: booking.payment_deadline = booking.created_at + 15 min (UTC), displayed to user in their local timezone,

- **Notifications**:
  Email/SMS include explicit timezone: 'Check-in: March 1, 2025 at 3:00 PM CET',

- **Edge case - DST**:
  Use proper timezone library (moment-timezone / date-fns-tz) not just UTC offset, handles daylight saving automatically. Example: User in California (UTC-8) books London hotel (UTC+0) for March 1-5 → Backend stores: {check_in: DATE '2025-03-01', check_out: DATE '2025-03-05', hotel_timezone: 'Europe/London', created_at: TIMESTAMP '2025-01-15 20:30:00 UTC'} → User sees: 'Booking confirmed for March 1-5, 2025 (London time), made on Jan 15, 2025 12:30 PM PST'. Alternative: Store all dates with timezone as TIMESTAMPTZ, but DATE simpler for booking dates as time-of-day not relevant for check-in dates.

---

### Q7: How do you implement cancellation policies and refund processing?

Multi-tier cancellation policy:

- **Policy rules stored per hotel/room**:
  cancellation_policy table {hotel_id, room_id, policy_type: 'flexible'/'moderate'/'strict', rules: jsonb},

- **Flexible**:
  { free_cancellation_hours: 24, partial_refund_percent: 50 until 12h, no_refund_hours: 0 }, Moderate: { 72h free, 24h 50%, <24h no refund }, Strict: { 7 days free, <7 days no refund },

- **Booking creation**:
  Copy policy to bookings.cancellation_policy jsonb (snapshot at booking time, protects user if policy changes later),

- **User cancels**:
  PUT /v1/bookings/{booking_id}/cancel with {reason: 'user_requested'},

- **Calculate refund**:
  (a) Fetch booking.cancellation_policy and booking.check_in, (b) Calculate hours_until_checkin = (check_in - now()) in hours, (c) Apply policy rules: if hours_until_checkin >= 24: refund_amount = booking.amount (100%), elif hours_until_checkin >= 12: refund_amount = booking.amount * 0.5 (50%), else: refund_amount = 0,

- **Process cancellation**:
  BEGIN TRANSACTION; (a) UPDATE bookings SET status='CANCELLED', cancelled_at=now(), refund_amount={refund_amount}, refund_status='PROCESSING', (b) UPDATE room_availability SET status='Available', booking_id=NULL WHERE booking_id={booking_id}, (c) If refund_amount > 0: INSERT INTO refunds (refund_id, booking_id, amount, status='PENDING'); COMMIT;,

- **Async refund**:
  Background job calls Payment Gateway: stripe.refunds.create({payment_intent_id, amount}),

- **Webhook confirms**:
  POST /webhooks/refund with {refund_id, status: 'succeeded'} → UPDATE refunds SET status='COMPLETED', bookings.refund_status='REFUNDED',

- **Notification**:
  Email user 'Booking cancelled, refund of ${amount} will appear in 5-7 business days',

- **Publish Kafka 'booking.cancelled' event. Partial refunds**:
  Some policies allow partial cancellation (e.g., 3-night booking, cancel 1 night) → more complex, need to track per-night cancellations. Abuse prevention: Track user cancellation_rate = cancelled_bookings / total_bookings, if >30% in last 10 bookings, flag account for review or increase cancellation fee. Example: User books on Jan 1 for March 1 check-in (flexible policy) → Cancels Feb 28 (1 day before) → 24 hours remaining → 100% refund → refund processed → room becomes available.

---

### Q8: How do you scale the search service to handle 100M searches per day?

Multi-tier scaling strategy:

- **Elasticsearch cluster**:
  10 data nodes, 3 master nodes, shard hotel index by region (US, EU, Asia), each region has 5 primary shards + 1 replica,

- **Geo-based routing**:
  API Gateway routes search requests to nearest ES cluster: User in US → us-east-1 cluster, EU → eu-west-1 cluster, reduces latency from 500ms to 50ms,

- **Caching layer**:
  Redis caches popular searches: Key: hash(location, checkIn, checkOut, filters) → Value: JSON search results, TTL: 10 min, Cache hit rate: ~60-70% (many users search same popular destinations), Cache size: 100GB Redis cluster (10 nodes) stores ~10M cached searches,

- **Query optimization**:
  (a) Use filter context (not query context) for filters: { 'bool': { 'filter': [...] } } → faster, cacheable, (b) Disable scoring for geo queries: 'track_scores': false, (c) Fetch only required fields: '_source': ['hotel_id', 'name', 'location', 'rating'],

- **API rate limiting**:
  Per-user: 100 searches/min, Per-IP: 1000 searches/min (protects against scraping bots),

- **Database read replicas**:
  PostgreSQL primary + 5 read replicas in each region, search service queries replicas for hotel metadata after ES returns IDs,

- **CDN for images**:
  Hotel images served from CloudFront edge locations, <50ms latency globally, 95% cache hit rate,

- **Load balancing**:
  100 search service instances behind ALB, auto-scaling based on CPU >70%,

- **Async indexing**:
  Hotel updates → Kafka → ES indexer consumer (separate service), doesn't block write operations, eventual consistency (1-2s delay acceptable for search),

- **Connection pooling**:
  Each search service instance maintains 50 ES connections (HTTP keep-alive),

- **Monitoring**:
  Prometheus + Grafana track: search latency p50/p95/p99, cache hit rate, ES cluster health, alert if p95 >500ms. Load: 100M searches/day = 1,157 searches/sec average, peak 3x = 3,500 searches/sec, Scaling math: 100 instances × 50 req/sec capacity = 5,000 req/sec headroom. Cost: ES cluster $5K/month, Redis $1K/month, API servers $3K/month = $9K/month for search infrastructure.

---

### Q9: How do you handle overbooking scenarios and inventory management?

Overbooking prevention with inventory tracking:

- **Room inventory model**:
  Hotels have multiple rooms of same type (e.g., 20 Deluxe rooms), rooms table: {hotel_id, room_type, total_count: 20}, individual_rooms table: {room_id, hotel_id, room_type, room_number: '101'},

- **Availability tracking**:
  Two approaches: (a) Aggregate: room_availability_aggregate {hotel_id, room_type, date, available_count, booked_count}, faster queries but complex updates, (b) Individual: room_availability {room_id, date, status, booking_id}, simpler but more rows.

- **Booking validation**:
  (a) User books 'Deluxe room' (not specific room_id initially), (b) Query: SELECT COUNT(*) as booked FROM room_availability WHERE hotel_id={hotel_id} AND room_type='Deluxe' AND date BETWEEN {dates} AND status='Booked', (c) Check: booked_count < total_deluxe_rooms for ALL dates in range, (d) Acquire distributed lock: SETNX lock:hotel:{hotel_id}:room_type:Deluxe:{dates}, (e) Assign specific room: SELECT room_id FROM rooms WHERE hotel_id={hotel_id} AND room_type='Deluxe' AND room_id NOT IN (SELECT room_id FROM room_availability WHERE date BETWEEN {dates} AND status='Booked') LIMIT 1 FOR UPDATE, (f) Update: UPDATE room_availability SET status='Booked', booking_id={booking_id} WHERE room_id={assigned_room_id} AND date BETWEEN {dates}.

- **Controlled overbooking**:
  Some hotels deliberately overbook by 5-10% (like airlines) to account for cancellations, overselling_limit table: {hotel_id, room_type, max_overbook_percent: 0.05}, validation: allow booking if booked_count <= total_count * 1.05, risk mitigation: if actually overbooked on check-in day, hotel upgrades guest to better room or relocates to partner hotel (hotel's responsibility).

- **Maintenance blocking**:
  Hotel can block rooms: room_availability.status = 'Maintenance' → excluded from available_count.

- **Edge case - race condition**:
  Two bookings at exact same time for last available room → distributed lock + database FOR UPDATE ensures only one succeeds. Example: Hotel has 20 Deluxe rooms, March 1: 18 booked, 2 available → User A and B both try booking → User A acquires lock → checks availability (2 available) → assigns room_123 → updates to 19 booked → releases lock → User B acquires lock → checks availability (1 available) → assigns room_124 → updates to 20 booked → releases lock → Third user tries → no rooms available.

---

### Q10: What's your strategy for disaster recovery and data consistency?

Multi-region DR with RPO/RTO targets:

- **Primary region**:
  us-east-1, Secondary: us-west-2, Tertiary: eu-west-1,

- **Database replication**:
  PostgreSQL primary in us-east-1, streaming replication to us-west-2 (warm standby) and eu-west-1 (read replica), Replication lag: <5 seconds under normal load,

- **RPO (Recovery Point Objective)**:
  5 seconds (maximum data loss acceptable), achieved via synchronous replication to secondary,

- **RTO (Recovery Time Objective)**:
  5 minutes (time to restore service), automated failover via Route53 health checks,

- **Elasticsearch**:
  Multi-cluster setup, each region has independent ES cluster, synced via CDC pipeline (PostgreSQL → Kafka → ES in all regions), eventual consistency acceptable for search,

- **Redis**:
  Redis Sentinel for automatic failover (master-slave replication), persistence via AOF (Append-Only File) for data durability,

- **S3 cross-region replication**:
  Hotel images replicated to backup bucket in secondary region automatically,

- **Kafka**:
  Multi-region setup with MirrorMaker replicates events to secondary Kafka cluster,

- **Failover procedure**:
  (a) Route53 health check detects primary region down (failed health checks for 30s), (b) Automatically updates DNS to point to secondary region, (c) Secondary PostgreSQL promoted to primary (pg_ctl promote), (d) API Gateway in secondary region starts accepting traffic, (e) Total downtime: ~2-5 minutes for DNS propagation,

- **Booking consistency during failover**:
  In-flight bookings during failover may fail → retry mechanism + idempotency ensures no double booking, Users see temporary error → retry automatically succeeds in new region,

- **Testing**:
  Monthly DR drills, simulate primary region failure, verify RTO <5 min, identify gaps,

- **Backups**:
  Daily full backup + hourly incremental backups to S3 Glacier, retention: 30 days, quarterly disaster recovery from backup test. Example: Primary region (us-east-1) suffers outage at 10:00 AM → Route53 detects failure at 10:00:30 → Initiates failover → us-west-2 promoted to primary at 10:02 → DNS updated at 10:03 → Users routed to new region at 10:05 → Service restored, data loss <5 sec (last replicated at 09:59:55). Cost: Multi-region setup adds ~40% infrastructure cost but provides business continuity.

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Search Engine** | Elasticsearch with `geo_point` | Raw Spatial Index (QuadTree / Geohash) | Combines spatial radius filtering with complex faceted filters (ratings, price, amenities) in a single sub-30ms query. |
| **Inventory Modeling** | Daily Pre-Aggregated Table (`room_inventory`) | Dynamic Overlapping Date Query on Bookings | Replaces expensive range scans with atomic $O(1)$ point row updates per day. |
| **Concurrency Control** | Redis `SET NX EX 600` + DB Engine `CHECK` Constraint | Pessimistic DB Row Locking (`FOR UPDATE`) | Offloads high-contention temporary holds to RAM; DB constraint guarantees zero double-booking at disk level. |
| **Catalog Storage** | Apache Cassandra + S3/CDN | Relational SQL (PostgreSQL) | Linearly scales read throughput for static hotel descriptions, photos, and amenities. |
| **Inventory Propagation**| CDC Pipeline (Debezium + Kafka) | Direct synchronous multi-DB writes | Decouples transactional checkout from search engine updates, preventing cross-database distributed rollbacks. |
