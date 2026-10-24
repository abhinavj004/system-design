# System Design: Design Yelp (Proximity & Places Search Platform)

> **Source Reference**: [Dodesimo System Design Notes: Design Yelp](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20Yelp.md)  
> **Methodology**: Modeled using the [Master 5-Step Architectural Blueprint](file:///d:/system%20design%20generation/00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG standard).  
> **Topic**: Designing a Scalable Local Business Discovery, Proximity Search, and Review Platform (Yelp, Google Places, Foursquare)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Proximity & Category Search**: Users can search for local businesses by keyword/name, category (Restaurants, Plumbers, Auto Repair), and geographical proximity (current GPS coordinates, city, or specified radius).
2. **Business Profile & Details**: Users can view business information, operating hours, contact details, dynamic tags (e.g., "Outdoor Seating", "Free Wi-Fi"), photo galleries, and aggregate rating score.
3. **Write Reviews & Ratings**: Registered users can submit a review with a mandatory star rating (1 to 5 stars), optional review text, and uploaded photos.
4. **Constraint (One Review per User)**: Each user can only submit **one active review per business**. If they review again, it updates their previous review.
5. **View Paginated Reviews**: Users can view reviews for a business sorted by date, helpfulness, or star rating.
6. **Real-Time Aggregate Ratings**: The system must maintain an accurate average star rating and total review count per business that updates shortly after reviews are posted.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Ultra-Low Latency for Search**: Proximity and keyword searches must return results in $< 50\text{ms}$.
2. **High Availability (99.99%) for Browsing (AP)**:
   - Browsing business profiles and searching places must never be blocked by down-stream service disruptions. Eventual consistency is completely acceptable for review counts and average rating updates.
3. **Read-Heavy Skew**: Typical Yelp traffic features a $200:1$ to $500:1$ Read-to-Write ratio (millions of searchers vs. thousands of review writers).
4. **Static POI Advantage**: Unlike Uber or Swiggy where agents move every 3 seconds, **businesses do not move**. Their GPS coordinates are static, which enables aggressive pre-computation and spatial caching.
5. **Spam & Abuse Protection**: Review bombing and fraudulent fake reviews must be mitigated via asynchronous fraud detection pipelines.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Traffic Estimations
- **Scale**: 100 Million Daily Active Users (DAU); 50 Million registered Businesses globally.
- **Search Queries (Reads)**:
  - Average user performs 5 searches/day:
    $$\text{Total Daily Searches} = 100\text{M} \times 5 = 500 \text{ Million searches/day}$$
    $$\text{Search QPS} = \frac{500,000,000}{86,400} \approx 5,800 \text{ queries/sec} \quad (\text{Peak} \approx 15,000 \text{ QPS})$$
- **Reviews Submitted (Writes)**:
  - 1% of active users write a review per week $\approx 1 \text{ Million reviews/day}$:
    $$\text{Review Write QPS} = \frac{1,000,000}{86,400} \approx 12 \text{ writes/sec} \quad (\text{Peak} \approx 50 \text{ writes/sec})$$

#### Storage Estimations (5-Year Projection)
- **Business Data**:
  - 50 Million businesses $\times$ 2 KB (name, lat, lon, address, tags, hours) $\approx 100 \text{ GB}$.
  - The entire global business catalog can fit in RAM on a few caching nodes!
- **Review Data**:
  - 1 Million reviews/day $\times$ 365 days $\times$ 5 years $\approx 1.825 \text{ Billion reviews}$.
  - Average review size: 500 bytes (text, rating, user_id, timestamps):
    $$\text{Review Text Storage} = 1.825\text{B} \times 500 \text{ bytes} \approx 912 \text{ GB} \approx 1 \text{ TB}$$
- **Photos & Media Storage**:
  - Assume 20% of reviews include 2 photos (average 2 MB per compressed image):
    $$\text{Daily Photo Ingress} = 1\text{M} \times 0.2 \times 2 \times 2\text{ MB} \approx 800 \text{ GB / day} \implies 1.46 \text{ Petabytes over 5 years}$$
  - **Decision**: Images are stored entirely in Object Storage (**AWS S3 / Cloudflare R2**) with global CDN edge caching.

#### Memory / Caching Estimations (80-20 Rule)
- 20% of popular businesses generate 80% of views.
- Active daily cache for hot businesses: $20\% \times 50\text{M} \times 2 \text{ KB} \approx 20 \text{ GB RAM}$.
- Easily handled by a small Redis cluster.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|        User        |        |      Business      |        |      Category      |
+--------------------+        +--------------------+        +--------------------+
| id (UUID)          | 1    * | id (UUID)          | *    * | id (INT)           |
| name (VARCHAR)     |<------>| name (VARCHAR)     |<------>| name (Restaurants) |
| email (VARCHAR)    |        | latitude (DECIMAL) |        | slug (restaurants) |
| profile_photo_url  |        | longitude (DECIMAL)|        +--------------------+
| created_at (TIME)  |        | geohash_6 (VARCHAR)|
+--------------------+        | address (TEXT)     |
                              | phone, website     |
                              | total_rating_sum   |
                              | review_count (INT) |
                              | avg_rating (FLOAT) |
                              +--------------------+
                                      | 1
                                      | *
+--------------------+        +--------------------+        +--------------------+
|    ReviewPhoto     | *    1 |       Review       | 1    1 |    ReviewVote      |
+--------------------+<------>+--------------------+<------>+--------------------+
| id (UUID)          |        | id (UUID)          |        | review_id (FK)     |
| review_id (FK)     |        | user_id (FK)       |        | user_id (FK)       |
| s3_url (TEXT)      |        | business_id (FK)   |        | vote_type (USEFUL, |
| thumbnail_url(TEXT)|        | rating (INT: 1-5)  |        |            FUNNY,  |
| created_at (TIME)  |        | text (TEXT, opt)   |        |            COOL)   |
+--------------------+        | created_at (TIME)  |        +--------------------+
                              | updated_at (TIME)  |
                              | UNIQUE(user, biz)  |
                              +--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Proximity & Category Search
```http
GET /api/v1/businesses/search?query=tacos&lat=37.7749&lon=-122.4194&radius_km=5&category=mexican&min_rating=4.0&page=1&limit=20
Response: 200 OK
{
  "total_hits": 84,
  "page": 1,
  "limit": 20,
  "businesses": [
    {
      "business_id": "biz_la_taqueria_sf",
      "name": "La Taqueria",
      "distance_km": 1.4,
      "avg_rating": 4.6,
      "review_count": 4820,
      "categories": ["Mexican", "Tacos"],
      "price_tier": "$$",
      "thumbnail_url": "https://cdn.yelp.com/biz_photos/la_taq_thumb.jpg",
      "is_open_now": true
    }
  ]
}
```

---

### 3.2 View Business Profile
```http
GET /api/v1/businesses/{business_id}
Response: 200 OK
{
  "business_id": "biz_la_taqueria_sf",
  "name": "La Taqueria",
  "address": "2889 Mission St, San Francisco, CA 94110",
  "coordinates": { "latitude": 37.7508, "longitude": -122.4181 },
  "avg_rating": 4.6,
  "review_count": 4820,
  "rating_breakdown": { "5_star": 3200, "4_star": 1100, "3_star": 300, "2_star": 120, "1_star": 100 },
  "operating_hours": { "Mon-Sat": "11:00-21:00", "Sun": "Closed" },
  "attributes": { "has_parking": false, "accepts_credit_cards": true, "outdoor_seating": true },
  "recent_photos": ["https://cdn.yelp.com/photos/1.jpg", "https://cdn.yelp.com/photos/2.jpg"]
}
```

---

### 3.3 Submit / Update a Review (Strict Idempotency)
```http
POST /api/v1/businesses/{business_id}/reviews
Authorization: Bearer <jwt_token>
Content-Type: application/json
X-Idempotency-Key: idemp_key_002_location

Request Body:
{
  "rating": 5,
  "text": "Best carne asada burrito in the Mission district! Crispy tortilla and authentic salsa.",
  "photo_ids": ["photo_uuid_101", "photo_uuid_102"]
}

Response: 201 Created (or 200 OK if updating existing review)
{
  "review_id": "rev_77192",
  "business_id": "biz_la_taqueria_sf",
  "user_id": "u_8819",
  "rating": 5,
  "created_at": "2026-10-18T18:45:00Z"
}
```
*Status Codes*:
- `201 Created`: First review by this user.
- `200 OK`: Existing review successfully modified.
- `400 Bad Request`: Rating out of bounds ($< 1$ or $> 5$).

---

### 3.4 Get Paginated Reviews
```http
GET /api/v1/businesses/{business_id}/reviews?page=1&limit=10&sort_by=helpful
Response: 200 OK
{
  "business_id": "biz_la_taqueria_sf",
  "total_reviews": 4820,
  "reviews": [
    {
      "review_id": "rev_77192",
      "user": { "name": "Sarah K.", "review_count": 42, "avatar_url": "..." },
      "rating": 5,
      "text": "Best carne asada burrito...",
      "photos": ["https://cdn.yelp.com/photos/101.jpg"],
      "created_at": "2026-10-18T18:45:00Z",
      "votes": { "useful": 18, "funny": 2, "cool": 9 }
    }
  ]
}
```

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/497571165_24020210430896276_2961534887528530562_n.jpg?updatedAt=1747167465961)
*Figure 4.1: Proximity Search & QuadTree Architecture (Source: [Interview with Bunny - Proximity Search Handbook](https://www.interviewwithbunny.com/systemdesign/07))*

### Component Interaction Topology
```
[ Mobile / Web Clients ]
           │
           ▼
[ Global Edge CDN (Cloudflare) ] (Caches business photos, static profiles, landing pages)
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
│         (JWT Auth, Rate Limiter, Reverse Proxy Route)       │
└─────────────────────────────────────────────────────────────┘
          │                                           │
  (Read/Search Traffic)                       (Review Write Traffic)
          │                                           │
          ▼                                           ▼
┌──────────────────┐                         ┌──────────────────┐
│  Search Service  │                         │  Review Service  │
│(Proximity Engine)│                         │ (Submission & DB)│
└──────────────────┘                         └──────────────────┘
     │        │                                       │
     │        ▼                                       ▼
     │  ┌───────────────┐                   ┌───────────────────┐
     │  │ Elasticsearch │                   │   PostgreSQL DB   │
     │  │(Geo BKD-Tree +│                   │(ACID Reviews with │
     │  │Faceted Filters│                   │UNIQUE(user, biz)) │
     │  └───────────────┘                   └───────────────────┘
     ▼                                                │
┌───────────────────────┐                             ▼ (Emit ReviewCreatedEvent)
│     Redis Cluster     │                    ┌───────────────────┐
│ (Hot Business Details │                    │   Apache Kafka    │
│  & Proximity Grids)   │                    └───────────────────┘
└───────────────────────┘                             │
           ▲                                          ├────────────────────────┐
           │                                          ▼                        ▼
           │ (Async Invalidation / Updates)   ┌───────────────┐        ┌──────────────┐
           └──────────────────────────────────│Rating Aggregate│       │Spam & Fraud  │
                                              │Worker Service │        │ML Pipeline   │
                                              └───────────────┘        └──────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Spatial Indexing for Proximity Search: Geohash vs. Elasticsearch BKD-Trees

Unlike ride-hailing cars that move dynamically, **businesses are static geographical points of interest (POIs)**.

#### Option A: Geohash Spatial Grid (In-Memory / Key-Value)
- Geohash converts 2D `(latitude, longitude)` coordinates into a Base32 string using Morton Z-order space-filling curves:
  - **Level 5 Geohash** ($\approx 4.9\text{ km} \times 4.9\text{ km}$): Represents city neighbourhoods.
  - **Level 6 Geohash** ($\approx 1.2\text{ km} \times 0.6\text{ km}$): Represents local shopping blocks.
- **The 8-Neighbor Bounding Box Rule**:
  - If a user is standing near the edge of cell `9q8yy`, searching only inside `9q8yy` will miss a restaurant 20 meters away in adjacent cell `9q8yz`.
  - The service computes the user's Geohash plus **all 8 surrounding neighbor cells** (a $3 \times 3$ grid) to prevent boundary clipping.

#### Option B: Elasticsearch with BKD-Trees (Chosen Production Solution)
While Geohashing is great for simple key-value lookups, real Yelp queries are **heavily compound and faceted**:
> *"Find Tacos within 5 km that have outdoor seating, rating $\ge 4.0$, price $\le \$$, and are currently open on Sunday evening."*

- **Elasticsearch BKD-Tree Geospatial Engine**:
  - Encodes coordinates into multidimensional binary trees (BKD-trees).
  - Evaluates spatial distance (`geo_distance`) alongside standard inverted indices for tags, categories, and ratings in a single memory-mapped query:
  ```json
  {
    "query": {
      "bool": {
        "must": [
          { "match": { "category": "mexican" } },
          { "range": { "avg_rating": { "gte": 4.0 } } }
        ],
        "filter": {
          "geo_distance": {
            "distance": "5km",
            "pin.location": { "lat": 37.7749, "lon": -122.4194 }
          }
        }
      }
    }
  }
  ```
  - Yields sub-30ms search latency across 50 Million records.

---

### 5.2 Concurrency & Real-Time Average Rating Recalculation

The reference GitHub note highlights a fundamental trade-off: **How do we efficiently calculate and update average business ratings without killing the database?**

#### 1. Naive On-the-Fly Aggregation — *Flawed*
```sql
SELECT AVG(rating), COUNT(*) FROM reviews WHERE business_id = :id;
```
- For popular places (e.g., *La Taqueria* with 5,000+ reviews), running this query on every business page view causes severe table scans, buffer pool thrashing, and high latency.

#### 2. Scheduled Cron Job — *Flawed*
- Running an hourly or daily batch script to recompute averages leads to stale ratings. A newly opened cafe receiving its first 5-star reviews will continue displaying *"No ratings yet"* for hours.

#### 3. Production Atomic In-Database Math + Asynchronous Event Propagation — *Optimal*
- Store two running counters in the `businesses` table: `total_rating_sum` (BIGINT) and `review_count` (INT):
  $$\text{avg\_rating} = \frac{\text{total\_rating\_sum}}{\text{review\_count}}$$

When a review is submitted:
```sql
-- Atomic update of running sum and count
UPDATE businesses 
SET total_rating_sum = total_rating_sum + :new_rating,
    review_count = review_count + 1,
    avg_rating = ROUND((total_rating_sum + :new_rating)::numeric / (review_count + 1), 2)
WHERE id = :business_id;
```
- **Handling Modified Reviews**:
  If a user updates an existing review from 3 stars to 5 stars:
  $$\Delta_{\text{diff}} = 5 - 3 = +2$$
  ```sql
  UPDATE businesses 
  SET total_rating_sum = total_rating_sum + :delta_stars,
      avg_rating = ROUND((total_rating_sum + :delta_stars)::numeric / review_count, 2)
  WHERE id = :business_id;
  ```
- **Asynchronous Sync**:
  - The review service pushes a `ReviewSubmittedEvent` to **Apache Kafka**.
  - A consumer service invalidates the Redis profile cache and updates the denormalized document in **Elasticsearch** so search ranking reflects the new rating within $< 2\text{ seconds}$.

---

### 5.3 Enforcing "One Review Per User Per Business"
How do we enforce that a user cannot spam multiple reviews for the same restaurant?

1. **Database Relational Integrity**:
   ```sql
   CREATE TABLE reviews (
       id UUID PRIMARY KEY,
       user_id UUID NOT NULL,
       business_id UUID NOT NULL,
       rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
       text TEXT,
       created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
       updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
       CONSTRAINT uq_user_business UNIQUE (user_id, business_id)
   );
   ```
2. **Fast Pre-Flight Check with Redis**:
   - Before hitting PostgreSQL, the API checks:
     ```redis
     HEXISTS user_reviews:{user_id} {business_id}
     ```
   - If present, the request is routed directly to the review update flow (`UPDATE reviews`) rather than trying to insert a duplicate.

---

### 5.4 Caching Architecture & Tiered Invalidation
- **Hot Profile Cache (Redis)**:
  - Cache Key: `business:profile:{business_id}` (stores JSON payload of business info + top 5 reviews).
  - TTL: 24 hours with **Event-Driven Cache Invalidation**:
    - When a business owner updates operating hours or when a review is submitted, a Kafka event triggers an immediate `DEL business:profile:{business_id}`.
- **Top Reviews Pre-Computation**:
  - Most users only read the first page of reviews sorted by "Most Helpful".
  - The top 20 most useful reviews are pre-sorted and stored in a Redis List/ZSET, avoiding sorting millions of historical reviews on read.

---

### 5.5 Anti-Fraud & Spam Detection Pipeline
Yelp is heavily targeted by competitors submitting fake 1-star reviews or business owners buying fake 5-star reviews.
- **Asynchronous Fraud Pipeline**:
  ```
  [ Review Service ] ──▶ [ Kafka Topic: raw-reviews ] ──▶ [ Fraud Detection Service ]
                                                                  │
                                      ┌───────────────────────────┴───────────────────────────┐
                                      ▼                                                       ▼
                              [ ML Classifier ]                                   [ Heuristic Rules ]
                              - Text similarity check                             - Review burst from same IP
                              - Account age & karma score                         - Sudden 1-star spike anomaly
                                      │                                                       │
                                      └───────────────────────────┬───────────────────────────┘
                                                                  ▼
                                                      [ Review Status Flag ]
                                                    (PUBLISHED / UNDER_REVIEW /
                                                            SHADOWBANNED)
  ```
- If flagged as suspicious, the review is marked `SHADOWBANNED` (visible only to the author) and excluded from the business's public `total_rating_sum` and `avg_rating`.

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Spatial Indexing** | Elasticsearch BKD-Tree `geo_distance` | Pure Geohash Key-Value (Redis) | Yelp queries combine spatial distance with complex text, category, and rating filters in a single sub-30ms pass. |
| **Rating Calculation** | Atomic In-DB Running Sum + Asynchronous Kafka Sync | On-the-fly `SELECT AVG()` / Daily Cron | Eliminates expensive table scans on read; provides near real-time rating updates without stale data. |
| **Duplicate Prevention**| Relational `UNIQUE(user_id, business_id)` + Redis Pre-check | Client-side validation only | Hard database constraint prevents duplicate review spamming even under concurrent network retries. |
| **Catalog Storage** | Relational SQL (PostgreSQL) + Read Replicas | NoSQL Key-Value (DynamoDB) | 50M static businesses take $< 100\text{ GB}$; relational engine cleanly enforces review constraints and foreign keys. |
| **Media Handling** | Pre-signed S3 URLs + Global CDN Edge | Direct Application Server Upload | Offloads massive image bandwidth ($1.5 \text{ PB}$) from backend application servers directly to cloud storage. |
| **Review Integrity** | Asynchronous Kafka ML Fraud Filter | Synchronous Pre-Submit Inspection | Decouples heavy AI spam analysis from the critical user submission latency path. |

