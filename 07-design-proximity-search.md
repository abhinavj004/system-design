# System Design 07: Design Proximity Search (Quad Tree vs Geohash vs PostGIS vs Elasticsearch)

> **Source**: [Interview with Bunny - Episode 7](https://www.youtube.com/watch?v=6K6ihHNkO_0)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/07](https://www.interviewwithbunny.com/systemdesign/07)  
> **Topic**: Designing Scalable Proximity & Geolocation Search Services (Yelp, Google Maps, Uber, Zomato/Swiggy Nearby)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Context & Problem Scope
Proximity search (finding places, restaurants, hotels, or moving drivers within a radius $R$ of a geographic coordinate `(latitude, longitude)`) is rarely a standalone interview question; rather, it is the foundational spatial building block required inside larger designs such as **Uber/Lyft** (Ride Hailing), **Airbnb/Booking.com** (Hotel Search), **Zomato/Swiggy** (Food Delivery), and **Tinder** (Nearby Users).

```
                      ┌──────────────────────────────────────┐
                      │          Target Geolocation          │
                      │     (lat: 12.9352, lon: 77.6245)      │
                      └──────────────────┬───────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
          [ Static POIs ]      [ Moving Entities ]     [ Compound Queries ]
          Restaurants, Hotels,   Riders, Cabs, Bikes    Text: "Burger King"
          Parks, ATMs, Stores    Updated every 4 sec    + Radius: 5 km + Open
```

---

### 1.2 Functional Requirements (FR)
1. **Radius Search (k-NN / Within Distance)**: Given a user location `(latitude, longitude)` and an optional search radius $R$ (e.g., $5\text{ km}$), return all matching points of interest (POIs) or active moving agents.
2. **Compound Filtering (Text + Spatial)**: Users can search by POI name, category, or cuisine (e.g., "Sushi", "Cafe") within the bounding radius $R$.
3. **POI Onboarding & CRUD**: Business owners can register new locations with metadata (name, address, operating hours, coordinates).
4. **Real-Time Location Ingestion (Dynamic Agents)**: Moving drivers/riders periodically push GPS coordinate streams (every 3–5 seconds).
5. **Pagination & Ranking**: Support distance-sorted or relevance-sorted pagination (`limit`, `offset` / cursor).

---

### 1.3 Non-Functional Requirements (NFR)
1. **Ultra-Low Latency**: Proximity search queries must return within $< 20\text{ms}$ (p95) and $< 50\text{ms}$ (p99).
2. **High Read Throughput**: Support up to $100,000\text{ QPS}$ globally for spatial lookups during peak meal or commute times.
3. **Fast Write Ingestion for Dynamic Agents**: Ingest hundreds of thousands of location updates/sec with sub-second propagation into the spatial index.
4. **High Availability (99.99%)**: Spatial search outages cripple core business funnels (e.g., placing orders or booking rides).
5. **Accuracy & Edge Resilience**: Eliminate spatial boundary clipping errors (e.g., when a user is positioned on the edge of an indexing cell).

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Scale & POI Storage (Static Places: Yelp / Google Places)
- **Total Registered POIs globally**: $500\text{ Million}$ places.
- **Record Size**:
  - `place_id`: 16 bytes (UUID)
  - `name`: 64 bytes
  - `latitude`: 8 bytes (IEEE 754 Float64)
  - `longitude`: 8 bytes (IEEE 754 Float64)
  - `geohash`: 12 bytes
  - `category_id`: 4 bytes
  - `metadata_json`: 256 bytes
  - **Total per POI** $\approx 400\text{ bytes}$
- **Total POI Storage**:
  $$\text{Storage} = 500\text{M} \times 400\text{ bytes} \approx 200\text{ GB}$$
  *200 GB of spatial POI data can comfortably fit in RAM or a modest database cluster.*

#### High-Velocity Dynamic Agents (Uber / Swiggy Drivers)
- **Active Moving Drivers globally**: $5\text{ Million}$ concurrent drivers.
- **Update Frequency**: Every $4\text{ seconds}$.
- **Write QPS**:
  $$\text{Write QPS} = \frac{5,000,000}{4} = 1,250,000\text{ updates/sec}$$
- **Read QPS**: $50,000,000\text{ DAU}$, each refreshing nearby drivers 10 times per session:
  $$\text{Peak Read QPS} \approx 100,000\text{ queries/sec}$$

---

## 2. Step 2: Identification of Core Entities

```
+------------------------------------+           +------------------------------------+
|               Place                |           |              Category              |
+------------------------------------+           +------------------------------------+
| id (UUID, PK)                      | *       1 | id (INT, PK)                       |
| name (VARCHAR)                     |<--------->| name (VARCHAR)                     |
| description (TEXT)                 |           | icon_url (VARCHAR)                 |
| category_id (INT, FK)              |           +------------------------------------+
| latitude (DOUBLE)                  |
| longitude (DOUBLE)                 |           +------------------------------------+
| geohash_12 (VARCHAR(12), INDEX)    |           |          DynamicDriverLocation     |
| address (TEXT)                     |           +------------------------------------+
| created_at (TIMESTAMP)             |           | driver_id (UUID, PK)               |
| updated_at (TIMESTAMP)             |           | latitude (DOUBLE)                  |
+------------------------------------+           | longitude (DOUBLE)                 |
                                                 | geohash (VARCHAR(8))               |
                                                 | status (VARCHAR)                   |
                                                 | last_ping_at (TIMESTAMP)           |
                                                 +------------------------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Proximity Radius Search (Places & POIs)
```http
GET /api/v1/places/search?lat=12.9352&lon=77.6245&radius_meters=5000&query=coffee&limit=20&cursor=eyJpZCI6MTAxfQ==
Authorization: Bearer <user_jwt>
Accept: application/json

Response: 200 OK
{
  "places": [
    {
      "id": "biz_blue_bottle_01",
      "name": "Third Wave Coffee Roasters",
      "distance_meters": 342.5,
      "latitude": 12.9348,
      "longitude": 77.6212,
      "category": "Cafe",
      "rating": 4.6,
      "geohash": "tdr1w7k9"
    },
    {
      "id": "biz_philz_coffee_02",
      "name": "Starbucks Coffee",
      "distance_meters": 820.1,
      "latitude": 12.9389,
      "longitude": 77.6291,
      "category": "Cafe",
      "rating": 4.4,
      "geohash": "tdr1w7mp"
    }
  ],
  "next_cursor": "eyJpZCI6MTAzfQ==",
  "total_found": 38
}
```

---

### 3.2 Place Onboarding (CRUD)
```http
POST /api/v1/places
Authorization: Bearer <admin_or_owner_jwt>
Content-Type: application/json

Request Body:
{
  "name": "McDonald's Koramangala",
  "category_id": 4,
  "latitude": 12.934533,
  "longitude": 77.622414,
  "address": "80 Feet Rd, 4th Block, Koramangala, Bengaluru, Karnataka 560034"
}

Response: 201 Created
{
  "id": "biz_sightglass_03",
  "name": "McDonald's Koramangala",
  "latitude": 12.934533,
  "longitude": 77.622414,
  "geohash": "tdr1w7k4b9",
  "created_at": "2026-10-18T10:15:30Z"
}
```

---

### 3.3 Dynamic Agent Fast Location Update
```http
POST /api/v1/drivers/location
Authorization: Bearer <driver_jwt>
Content-Type: application/json

Request Body:
{
  "driver_id": "d-88412",
  "latitude": 12.935100,
  "longitude": 77.624100,
  "speed_kmh": 28.4,
  "bearing_degrees": 182.0,
  "timestamp": 1729252000
}

Response: 200 OK
{
  "status": "ACK",
  "geohash_level_7": "tdr1w7k"
}
```

---

## 4. Step 4: High-Level Architecture

```
[ User App (Search) ]          [ Driver App (Location Pings) ]
         │                                    │
         ▼                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / Envoy                      │
│            (Rate Limiting, Auth, Load Balancing)            │
└─────────────────────────────────────────────────────────────┘
         │                                    │
         │ (Spatial Query)                    │ (High QPS GPS Ping)
         ▼                                    ▼
┌───────────────────────┐          ┌──────────────────────────┐
│ Place Search Service  │          │ Location Ingest Service  │
│ (Stateless Pods)      │          │ (High Throughput Netty)  │
└───────────────────────┘          └──────────────────────────┘
      │           │                               │
      │           │                               ▼
      │           │                     ┌────────────────────┐
      │           │                     │ Apache Kafka / SQS │
      │           │                     └────────────────────┘
      │           │                               │
      │           ▼                               ▼
      │  ┌───────────────────────┐      ┌────────────────────┐
      │  │ Elasticsearch Cluster │      │ Redis Geo Shards   │
      │  │ (Full Text + Geo BKD) │      │ (Dynamic Cabs /    │
      │  └───────────────────────┘      │  GEOADD / GEORADIUS│
      │                                 └────────────────────┘
      ▼                                           ▲
┌───────────────────────────────────────┐         │ (Sync Ingestion)
│ PostgreSQL Primary DB + PostGIS Ext   │─────────┘
│ (Authoritative Master POI Store)      │
└───────────────────────────────────────┘
```

### Architecture Layer Separation
1. **API Gateway & Routing**: Differentiates between low-frequency read/search traffic and ultra-high-frequency dynamic driver GPS pings.
2. **Search Service (Read Path)**: Stateless service that queries Elasticsearch (for POIs with text & filtering) or Redis Geo (for moving vehicles).
3. **Location Ingestion Service (Write Path)**: Event-driven gateway receiving hundreds of thousands of UDP/WebSocket or lightweight HTTP GPS pings per second, offloading directly to Kafka.
4. **Authoritative Master DB**: PostgreSQL with PostGIS extension for relational integrity, transactional updates, and geospatial calculations.
5. **Specialized Spatial Engines**:
   - **Elasticsearch**: For compound text search + distance filtering + faceted aggregations (Static POIs).
   - **Redis Cluster (GEO / Sorted Sets)**: For microsecond-latency dynamic driver location tracking.

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Quad Tree Mechanics & Why Bunny Advises Against It in Production

#### How Quad Trees Work
A Quad Tree is a tree data structure in which each internal node has exactly **four children**:
- North-West ($NW$)
- North-East ($NE$)
- South-West ($SW$)
- South-East ($SE$)

The entire 2D coordinate space of the globe (Latitude: $-90^\circ$ to $+90^\circ$, Longitude: $-180^\circ$ to $+180^\circ$) forms the root node.

```
                    [ Root: Entire World Map ]
                   ┌────────┬────────┬────────┐
                   │        │        │        │
                 [ NW ]   [ NE ]   [ SW ]   [ SE ]
                            │
               ┌────────┬───┴────┬────────┐
               │        │        │        │
             [NW]     [NE]     [SW]     [SE]
                                         │
                             (Deeply nested leaf node)
                             [ Restaurant A, B ]
```

#### Why Quad Tree Fails in Real-World Distributed Production Systems (Bunny's Analysis)

1. **Severe Height Imbalance (Non-Uniformity of Earth)**:
   - Quad trees assume data points are evenly scattered. However, 71% of Earth is ocean, and large land masses (Greenland, Sahara, Antarctica) have near-zero restaurants.
   - Conversely, high-density metropolitan zones (e.g., Koramangala Bangalore, Manhattan NYC, Shibuya Tokyo) hold tens of thousands of venues in a few square kilometers.
   - Result: Dense quadrants split into thousands of nested levels, creating an extremely deep, skewed tree, while empty ocean quadrants remain shallow leaf nodes.

2. **The Neighbor Traversal Penalty (Backtracking Overhead)**:
   - Finding POIs within a $5\text{ km}$ radius requires searching not just the quadrant where the user sits, but also neighboring quadrants.
   - If a user is situated near a boundary line, finding adjacent nodes requires **walking up the tree** to the common parent, traversing down sibling subtrees, and recursively scanning child leaf nodes:

```
Step 1: User sits in Leaf [SW-SE-NW-0]
Step 2: No restaurants in immediate leaf -> Ascend to Parent [SW-SE-NW]
Step 3: Scan sibling subtrees [SW-SE-NE], [SW-SE-SW], [SW-SE-SE]
Step 4: Still insufficient? -> Ascend further to Grandparent [SW-SE]
Step 5: Recursively traverse down all cousins...
```
This backtracking creates multi-node memory hopping, destroying cache locality and driving up latency.

3. **In-Memory Volatility & Dynamic Mutation Bottlenecks**:
   - A Quad Tree is an **in-memory data structure**, not an off-the-shelf distributed database.
   - Whenever a new restaurant onboards or a driver changes position, nodes must split or rebalance (analogous to an AVL or Red-Black tree rebalance).
   - In a distributed environment, holding the Quad Tree in RAM requires maintaining **Read/Write replica sets**:
     - *Read Copy*: Serves search traffic without locking.
     - *Write Copy*: Mutated on background ingestion, then swapped atomically.
   - Synchronizing multi-gigabyte tree representations across dozens of stateless server pods is notoriously brittle.

> **Bunny's Verdict**:  
> *"Avoid building custom Quad Trees in production interviews unless specifically requested. Quad Trees are complex to implement, extremely difficult to rebalance, and computationally expensive to traverse across boundaries. Prefer Geohash, PostGIS, or Elasticsearch."*

---

### 5.2 Deep Dive 2: Geohash Mechanics & Spatial Prefixing

#### How Geohash Works
Geohash is a public-domain hierarchical spatial data structure invented by Gustavo Niemeyer in 2008. It translates a 2D coordinate `(latitude, longitude)` into a compact alphanumeric string using **Z-order space-filling curves** (Morton code) and Base32 encoding.

#### Mathematical Encoding Algorithm:
1. **Latitude Range**: $[-90.0, +90.0]$
2. **Longitude Range**: $[-180.0, +180.0]$
3. **Binary Bisection (Interleaving)**:
   - Divide longitude in half ($[-180, 0]$ and $[0, 180]$). If $lon \ge 0$, bit = `1`; else `0`.
   - Divide latitude in half ($[-90, 0]$ and $[0, 90]$). If $lat \ge 0$, bit = `1`; else `0`.
   - **Interleave bits**: Even bits are Longitude, odd bits are Latitude:
     $$\text{Bit Stream} = [lon_0, lat_0, lon_1, lat_1, lon_2, lat_2, \dots]$$
4. **Base32 Grouping**: Group bits into 5-bit chunks and map to Base32 character set:
   `0123456789bcdefghjkmnpqrstuvwxyz` (excluding letters `a`, `i`, `l`, `o` to avoid confusion with digits `0` and `1`).

```
Coordinate: (12.9352° N, 77.6245° E) -> Koramangala, Bengaluru

Longitude Bisection:  77.6245 -> [1, 1, 0, 1, 1, 1, 0, 1, ...]
Latitude Bisection:   12.9352 -> [1, 0, 0, 1, 0, 0, 1, 0, ...]
Interleaved Bits:     1 1 1 0 0 0 1 1 1 0 1 0 0 1 1 0 1 0 ...
Grouped 5-bit chunks: [11100] [01110] [10011] ...
Base32 Geohash:       't'     'd'     'r'   '1'   'w' ... -> "tdr1w7k"
```

#### Bunny's Key Insight: The Single-Character Prefix Property
As demonstrated by Bunny in the video (`[19:20]`):
- **McDonald's Koramangala**: `tdr1w7k4`
- **Zudio Koramangala** (100 meters away): `tdr1w7k8`
- Both locations share the common prefix **`tdr1w7k`**!
- Searching for nearby entities does not require tree traversals; it simply requires searching for entities sharing the same geohash prefix.

#### Geohash Precision Table

| Geohash Length | Cell Width | Cell Height | Error / Radius | Ideal Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **1** | $5,009\text{ km}$ | $4,992\text{ km}$ | Continental | Global continental routing |
| **2** | $1,252\text{ km}$ | $624\text{ km}$ | Sub-continental | State / Multi-province routing |
| **3** | $156.5\text{ km}$ | $156.1\text{ km}$ | Large District | Metropolitan boundary |
| **4** | $39.1\text{ km}$ | $19.5\text{ km}$ | City Wide | City search (e.g., "All Bangalore") |
| **5** | $4.89\text{ km}$ | $4.89\text{ km}$ | Suburb / Ward | **Standard 5 km Radius Search (Zomato)** |
| **6** | $1.22\text{ km}$ | $0.61\text{ km}$ | Neighborhood | **Hyperlocal 1 km Radius (Uber Pickup)** |
| **7** | $152.9\text{ m}$ | $152.4\text{ m}$ | Block / Street | Precise building / street block |
| **8** | $38.2\text{ m}$ | $19.0\text{ m}$ | Property | Delivery gate identification |
| **9** | $4.77\text{ m}$ | $4.77\text{ m}$ | Doorstep | Precise GPS pinpoint |

#### The Bounding Box Edge Problem & The 8-Neighbor Algorithm
A classic trap in naive geohash implementations:
If a user is positioned at the very edge of cell `tdr1w7`, a restaurant 20 meters away located in the adjacent cell `tdr1w6` will **not** share the same prefix!

```
┌─────────────┬─────────────┬─────────────┐
│   tdr1wb    │   tdr1wc    │   tdr1wf    │
├─────────────┼─────────────┼─────────────┤
│   tdr1w8    │   tdr1w9    │   tdr1wd    │
│             │    * User   │             │
├─────────────┼─────────────┼─────────────┤
│   tdr1w2    │   tdr1w3    │   tdr1w6    │
│             │             │ * Venue     │
└─────────────┴─────────────┴─────────────┘
```

**The Solution**: Always compute the **8 neighboring bounding geohashes** along with the central cell:
$$\text{Query Set} = \{\text{Center}\} \cup \{\text{North, NE, East, SE, South, SW, West, NW}\}$$
The query then performs a multi-cell prefix match:
```sql
SELECT * FROM places 
WHERE geohash_prefix IN ('tdr1w9', 'tdr1wb', 'tdr1wc', 'tdr1wf', 'tdr1w8', 'tdr1wd', 'tdr1w2', 'tdr1w3', 'tdr1w6');
```
This guarantees zero boundary dropouts.

---

### 5.3 Deep Dive 3: PostgreSQL + PostGIS (The Relational Spatial Engine)

PostGIS turns PostgreSQL into a high-performance spatial database by adding geographic data types and spatial indexing.

#### 1. Table Schema with PostGIS Geometry & GIST Index
```sql
-- Enable PostGIS and Fuzzy Search extensions
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE places (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    category_id INT NOT NULL,
    address TEXT,
    -- Store as GEOGRAPHY(Point, 4326) for true geodesic meter calculations
    location GEOGRAPHY(Point, 4326) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Crucial: Create Generalized Search Tree (GiST) Spatial Index
CREATE INDEX idx_places_spatial_gist ON places USING GIST (location);

-- Trigram index for fuzzy text matching
CREATE INDEX idx_places_name_trgm ON places USING GIN (name gin_trgm_ops);
```

#### 2. Spatial Query: Finding Places within 5 km
```sql
-- Bunny's highlighted PostGIS function: ST_DWithin
SELECT 
    id, 
    name, 
    ST_Distance(location, ST_MakePoint(77.6245, 12.9352)::geography) AS distance_meters
FROM places
WHERE ST_DWithin(
    location,
    ST_MakePoint(77.6245, 12.9352)::geography, -- Note: Longitude first, then Latitude!
    5000 -- Distance in meters
)
ORDER BY distance_meters ASC
LIMIT 20;
```

#### 3. Compound Spatial + Fuzzy Text Query
When users type "MacD" or "Macdonald":
```sql
SELECT 
    id, 
    name, 
    ST_Distance(location, ST_MakePoint(77.6245, 12.9352)::geography) AS distance_meters
FROM places
WHERE ST_DWithin(location, ST_MakePoint(77.6245, 12.9352)::geography, 5000)
  AND (name % 'MacD' OR name ILIKE '%MacD%') -- Fuzzy match via pg_trgm
ORDER BY distance_meters ASC
LIMIT 20;
```

> **Why PostGIS Rocks**:
> - True ellipsoidal distance calculations (Haversine/WGS 84 geodesic math).
> - Strict ACID compliance and relational integrity.
> - GiST index operates in $O(\log N)$ search complexity.

---

### 5.4 Deep Dive 4: Elasticsearch (The Full-Text + Proximity Search Engine)

When an application needs full-text scoring, autocomplete, typo tolerance, multi-faceted filtering (price, cuisine, vegan, rating), and proximity radius sorting, **Elasticsearch** is the industry standard.

#### 1. Elasticsearch Index Mapping with `geo_point`
```json
PUT /places_index
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": { 
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "suggest": { "type": "completion" }
        }
      },
      "category": { "type": "keyword" },
      "rating": { "type": "float" },
      "location": { "type": "geo_point" }
    }
  }
}
```

#### 2. Compound Search Query (`bool` query with text search + `geo_distance` filter)
```json
POST /places_index/_search
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "fresh brewed coffee",
          "fields": ["name^2", "category"],
          "fuzziness": "AUTO"
        }
      },
      "filter": [
        {
          "geo_distance": {
            "distance": "5km",
            "location": {
              "lat": 12.9352,
              "lon": 77.6245
            }
          }
        },
        {
          "range": {
            "rating": { "gte": 4.0 }
          }
        }
      ]
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 12.9352,
          "lon": 77.6245
        },
        "order": "asc",
        "unit": "m",
        "mode": "min",
        "distance_type": "arc"
      }
    }
  ]
}
```

> **Under the Hood in Elasticsearch**:
> - Uses **BKD-Trees** (Block k-d trees) since Lucene 6.
> - BKD-trees partition multidimensional numeric and geographic spaces into leaves stored contiguously on disk with block-level compression.
> - Provides ultra-fast range queries and polygon/bounding-box intersections without exploding RAM.

---

### 5.5 Deep Dive 5: Static Entities vs Dynamic Moving Agents (Uber / Swiggy Driver Tracking)

Bunny clarifies that the right technology depends heavily on whether entities are **static** (places) or **dynamic** (moving drivers):

```
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│       Static Entities           │       │    Dynamic Fast-Moving Agents   │
│   (Restaurants, Hotels, ATMs)   │       │   (Uber Cabs, Delivery Boys)    │
├─────────────────────────────────┤       ├─────────────────────────────────┤
│ • Infrequent updates (< 1/day)  │       │ • High frequency (every 3-5 sec)│
│ • Rich textual attributes       │       │ • Ephemeral state (live GPS)    │
│ • Heavy compound filtering      │       │ • Pure radius / proximity       │
│ • Best: PostGIS / Elasticsearch │       │ • Best: Redis Geo / In-Memory   │
└─────────────────────────────────┘       └─────────────────────────────────┘
```

#### Redis In-Memory Geospatial Implementation (Redis GEO)
Redis internally represents coordinates using **52-bit Geohash integers stored in a Sorted Set (ZSET)**.
The score of the sorted set is the 52-bit Geohash integer.

```bash
# 1. Driver reports location every 4s
GEOADD active_drivers:bangalore 77.6245 12.9352 "driver_401"
GEOADD active_drivers:bangalore 77.6291 12.9389 "driver_402"

# 2. User searches for nearest cabs within 3 km
GEOSEARCH active_drivers:bangalore 
    FROMLONLAT 77.6245 12.9352 
    BYRADIUS 3 km 
    WITHDIST 
    WITHCOORD 
    ASC 
    COUNT 10
```
- **Time Complexity**: $O(N + \log M)$ where $N$ is the number of elements in the bounding area and $M$ is total elements in the sorted set.
- **Latency**: Sub-millisecond ($< 1\text{ms}$).
- **Expiration / TTL**: Keys are refreshed or purged automatically when driver goes offline.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: Explain geohash and how to handle edge cases at cell boundaries.

Geohash encoding:

- **Point 1**: Interleave lat/lng bits → binary string → convert to base32 → 'sgkztus' (7 chars ≈ ±76m),

- **Precision table**:
  length 5 = ±2.4km, length 7 = ±76m, length 9 = ±2m. Proximity property: Nearby locations share common prefix ('sgkztus', 'sgkztur' both in SF area). Edge case problem: Two points 50m apart on cell boundary have different hashes ('sgkztzzz' vs 'sgku000' - different 5th char). Solution:

- **Point 1**: Calculate 8 neighbor geohash cells using library (N, NE, E, SE, S, SW, W, NW),

- **Query all 9 cells (center + 8 neighbors)**:
  WHERE geohash IN ('sgkzt', 'sgkzu', 'sgkzs', ...) - 9 total,

- **Post-filter with Haversine to get exact circular radius. Optimization**:
  Use larger geohash precision (fewer chars) to get bigger box that definitely contains circle, then filter. Example: 5km radius → use precision 5 (±2.4km cell) → single query → Haversine filter → exact results.

---

### Q2: Compare QuadTree vs Geohash vs PostGIS - when to use each?

QuadTree:

- **Structure**:
  Recursive 4-way partitioning, each node has NW/NE/SW/SE children,

- **Search**:
  O(log n) traversal, prune subtrees that don't intersect search circle,

- **Use**:
  In-memory single server (<1M points), game engines, dynamic data,

- **Pros**:
  Fast (<1ms), easy updates,

- **Cons**:
  Not distributable, lost on restart, complex implementation. Geohash:

- **Encoding**:
  Lat/lng → base32 string, prefix matching,

- **Search**:
  SQL LIKE 'prefix%' with B-tree index, query 9 cells (center + 8 neighbors),

- **Use**:
  Simple SQL databases, distributed systems, approximate OK,

- **Pros**:
  Works with any DB, easy to shard, human-readable,

- **Cons**:
  Returns box not circle, edge cases. PostGIS:

- **Extension**:
  GEOGRAPHY type, ST_DWithin for radius, GIST spatial index,

- **Search**:
  O(log n) with R-tree, exact Vincenty distance,

- **Use**:
  Static data, ACID needed, complex shapes (polygons),

- **Pros**:
  Exact distance, mature, complex queries,

- **Cons**:
  PostgreSQL-only, harder to shard, ~100ms latency. Decision: QuadTree for single-server in-memory, Geohash for distributed approximate, PostGIS for ACID + exact distance. Production: Hybrid - Redis Geo (hot data) + Elasticsearch (search) + PostGIS (source of truth).

---

### Q3: How do you optimize proximity search for millions of locations?

Multi-layer optimization:

- **Spatial indexing**:
  Use GIST index (PostGIS), B-tree on geohash, or Redis GEORADIUS - reduces from O(n) scan to O(log n) lookup,

- **Bounding box pre-filter**:
  Calculate lat/lng bounds, WHERE lat BETWEEN (lat-delta) AND (lat+delta) AND lng BETWEEN (lng-delta) AND (lng+delta), eliminates 90% before expensive distance calc, delta = radius_km / 111,

- **Caching**:
  Redis cache for hot locations (top 1000 per city), TTL=10 min, serves 80% queries in <5ms,

- **Geographic sharding**:
  Partition by region (US-West, US-East, EU) or geohash prefix, reduces per-shard dataset by 5-10x,

- **Materialized views**:
  Pre-compute popular queries (restaurants by city + 5km radius), refresh hourly,

- **Read replicas**:
  PostgreSQL replicas in each region for geo queries, route to nearest replica,

- **Lazy distance calc**:
  Return top N by bounding box first, calculate exact distance only for displayed results,

- **Result limit**:
  LIMIT 20 instead of returning all matches (user only sees first page anyway). Example: 10M restaurants → geohash prefix filter → 10K candidates → bounding box → 1K candidates → Haversine → 200 within exact radius → sort → return top 20 → total latency 50ms vs 5000ms without optimization.

---

### Q4: Implement proximity search with PostgreSQL PostGIS - show schema and query.

Setup:

- **Install**:
  CREATE EXTENSION postgis;,

- **Schema**:
  CREATE TABLE restaurants (id SERIAL PRIMARY KEY, name VARCHAR

- **Point 255**: , location GEOGRAPHY(POINT, 4326), category VARCHAR

- **Point 100**: , rating DECIMAL(2,1)),

- **Index**:
  CREATE INDEX idx_location_gist ON restaurants USING GIST (location),

- **Insert**:
  INSERT INTO restaurants (name, location, category, rating) VALUES ('Taco Bell', ST_GeomFromText('POINT(77.5946 12.9716)', 4326), 'Fast Food', 4.2). Query: SELECT id, name, category, rating, ST_Distance(location, ST_GeogFromText('SRID=4326;POINT(77.6080 12.9700)')) AS distance_meters FROM restaurants WHERE ST_DWithin(location, ST_GeogFromText('SRID=4326;POINT(77.6080 12.9700)'), 5000) AND category = 'Fast Food' AND rating >= 4.0 ORDER BY distance_meters LIMIT 20; Explanation:

- **ST_GeogFromText**:
  Creates geography point from user location (77.6080, 12.9700),

- **ST_DWithin**:
  Pre-filters using bounding box, then exact distance check ≤ 5000 meters,

- **ST_Distance**:
  Calculates exact distance in meters using Vincenty formula,

- **GIST index**:
  Enables O(log n) spatial lookup,

- **Additional filters**:
  category and rating for combined search,

- **Performance**:
  ~50-100ms for 10M rows. Alternative using geometry (faster but less accurate): Change GEOGRAPHY to GEOMETRY, use ST_MakePoint(lng, lat) instead of ST_GeomFromText, distance in degrees not meters.

---

### Q5: How does Redis GEORADIUS work internally and when should you use it?

Internal implementation:

- **Data structure**:
  Redis stores geospatial data as sorted set (ZSET), score = geohash of location, member = location ID,

- **Geohash encoding**:
  Internally encodes lat/lng to 52-bit geohash integer,

- **GEORADIUS algorithm**:
  (a) Calculate geohash of center point, (b) Determine geohash prefix range for radius, (c) ZRANGEBYLEX to get all members in geohash range (O(log n)), (d) Calculate exact Haversine distance for each candidate, (e) Filter to points within radius, sort by distance,

- **Time complexity**:
  O(log n + m) where m = results. Commands:

- **GEOADD key lng lat member**:
  Add/update location,

- **GEORADIUS key lng lat radius km WITHDIST ASC COUNT 10**:
  Search circle, return distance sorted,

- **GEORADIUSBYMEMBER key member radius km**:
  Find neighbors of existing member,

- **GEODIST key member1 member2 km**:
  Distance between two members. When to use:

- **Point 1**: Real-time moving objects (drivers, delivery agents update every 5s),

- **Point 2**: Hot/ephemeral data (active sessions, current locations),

- **Point 3**: <10ms latency requirement,

- **Point 4**: <10M points (memory constraint),

- **High read frequency (100K queries/sec). Don't use for**:
  

- **Point 1**: Billions of points (too expensive in RAM),

- **Point 2**: Complex filtering (category, rating) - use Elasticsearch,

- **ACID persistence - use PostGIS. Hybrid pattern**:
  Redis for active drivers (10K online) + PostgreSQL for all drivers (1M total), sync every 60s.

---

### Q6: How do you calculate Haversine distance and why use bounding box pre-filter?

Haversine formula:

- **Purpose**:
  Calculate great-circle distance between two points on sphere (Earth),

- **Inputs**:
  Point A (lat1, lng1), Point B (lat2, lng2), Earth radius R = 6371 km,

- **Formula**:
  a = sin²((lat2-lat1)/2) + cos(lat1)·cos(lat2)·sin²((lng2-lng1)/2), c = 2·atan2(√a, √(1-a)), distance = R·c,

- **Implementation**:
  Convert degrees to radians (multiply by π/180), apply trig functions, result in km,

- **Example**:
  SF (37.7749, -122.4194) to LA (34.0522, -118.2437): Δlat = -3.7227 rad, Δlng = 4.1757 rad → a = 0.0382 → c = 0.391 → distance = 6371 × 0.391 = 559 km. Accuracy: ±0.5% error (assumes sphere, Earth is ellipsoid). Bounding box optimization:

- **Problem**:
  Computing Haversine for 1M points = 1M trig calculations = ~1 second CPU time,

- **Solution**:
  Pre-filter with simple lat/lng range check: WHERE lat BETWEEN (user_lat - delta) AND (user_lat + delta) AND lng BETWEEN (user_lng - delta) AND (user_lng + delta),

- **Delta**:
  delta = radius_km / 111 (rough: 1 degree ≈ 111 km), for 5km radius → delta = 0.045 degrees,

- **Effect**:
  Eliminates 90-95% of points using index scan (fast),

- **Final step**:
  Haversine only for remaining 5-10% → total time <10ms. Example: 1M points, 5km radius → bounding box reduces to 10K candidates → Haversine for 10K → <10ms vs 1000ms without optimization.

---

### Q7: Design Elasticsearch mapping and query for 'pizza restaurants within 5km, rating ≥4'.

Index mapping: PUT /restaurants { 'mappings': { 'properties': { 'name': { 'type': 'text', 'analyzer': 'standard' }, 'category': { 'type': 'keyword' }, 'location': { 'type': 'geo_point' }, 'rating': { 'type': 'float' }, 'price_range': { 'type': 'keyword' }, 'is_open': { 'type': 'boolean' } } } }. Index document: POST /restaurants/_doc { 'name': 'Taco Bell', 'category': 'Fast Food', 'location': { 'lat': 12.9716, 'lon': 77.5946 }, 'rating': 4.2, 'price_range': '$', 'is_open': true }. Search query: POST /restaurants/_search { 'query': { 'bool': { 'must': [ { 'match': { 'name': 'pizza' } } ], 'filter': [ { 'geo_distance': { 'distance': '5km', 'location': { 'lat': 12.97, 'lon': 77.59 } } }, { 'range': { 'rating': { 'gte': 4.0 } } }, { 'term': { 'is_open': true } } ] } }, 'sort': [ { '_geo_distance': { 'location': { 'lat': 12.97, 'lon': 77.59 }, 'order': 'asc', 'unit': 'km' } } ], 'size': 20 }. Result: { 'hits': { 'hits': [ { '_source': { 'name': 'Pizza Hut', 'rating': 4.5 }, 'sort': [ 0.8 ] } ] } } (0.8 km distance). Aggregations for facets: 'aggs': { 'price_facets': { 'terms': { 'field': 'price_range' } }, 'distance_buckets': { 'geo_distance': { 'field': 'location', 'origin': { 'lat': 12.97, 'lon': 77.59 }, 'ranges': [ {'to': 1}, {'from': 1, 'to': 3}, {'from': 3, 'to': 5} ] } } }. Performance: ~20-50ms for 100M documents, scales horizontally.

---

### Q8: How do you shard geospatial data for horizontal scaling?

Sharding strategies:

- **Geographic sharding**:
  Divide by region (US-West, US-East, EU-West, EU-East, Asia-Pacific), each region = separate database/cluster, route queries based on location bounds, Pros: Simple routing, isolated failures, Cons: Unbalanced load (NYC has more restaurants than Montana), cross-region queries complex.

- **Geohash prefix sharding**:
  Shard by first N characters of geohash (e.g., first 2 chars = 1024 possible prefixes → 16 shards), use consistent hashing to map geohash prefix to shard, Pros: Even distribution, Cons: Radius queries may span multiple shards.

- **Hybrid**:
  Regions + geohash - shard by region first (US-East), then by geohash prefix within region, Example: US-East shard 1: geohashes starting with '9q', '9r', US-East shard 2: '9w', '9x'. Query routing:

- **Point 1**: Extract location from query,

- **Point 2**: Determine geohash or region,

- **Point 3**: Route to appropriate shard(s),

- **If query spans multiple shards (large radius near boundary), fan out to all relevant shards, merge results. Example**:
  User at NYC (40.7128, -74.0060) searches 10km → all results in US-East shard, User at Philadelphia (border of US-East/US-Mid) searches 50km → query both shards, merge by distance. Cross-shard complexity:

- **Small radius (<10km)**:
  Usually hits 1-2 shards,

- **Large radius (>50km)**:
  May hit 3-5 shards, scatter-gather pattern. Performance: Per-shard dataset reduced 10x (10M → 1M), query latency improved 5x (500ms → 100ms). Elasticsearch: Auto-shards by document ID, manual routing by geohash possible using custom routing key.

---

### Q9: What's your caching strategy for proximity searches?

Multi-layer caching:

- **Result cache**:
  Cache entire query results in Redis, key = hash(lat, lng, radius, filters), value = [location_ids], TTL=5 min, handles identical repeated queries, Example: search: {lat: 37.77, lng: -122.42, radius: 5km, category: Italian} → cache key: hash(37.77:-122.42:5000:Italian) → value: [id1, id2, id3, ...],

- **Location cache**:
  Cache popular/active locations in Redis GEOSPATIAL, GEOADD restaurants:hot:{city_id}, TTL=10 min, serves 80% queries via GEORADIUS in <5ms, Eviction: LRU (Least Recently Used), keep top 1000 per city.

- **Application cache**:
  In-memory cache (Caffeine, Guava) in API servers for very frequent queries, 100-element LRU cache, serves in <1ms without network hop.

- **CDN cache**:
  For static location data (restaurant metadata, images), cache at edge locations globally. Cache warming:

- **Point 1**: Background job analyzes query logs,

- **Point 2**: Identifies top 100 queries per city by frequency,

- **Pre-populates Redis cache before peak hours. Invalidation**:
  

- **On location update**:
  Invalidate all cache keys containing that location, Use Redis Pub/Sub or Kafka to broadcast invalidation event,

- **Pattern matching**:
  Use Redis SCAN to find keys matching location_id, DEL all matching keys,

- **Time-based**:
  Regardless of updates, expire cache entries after TTL to prevent indefinite staleness. Example cache hit flow: User searches → check Redis result cache → hit (identical query 2 min ago) → return cached results → <5ms latency. Cache miss flow: → query Elasticsearch → 50ms → cache result → return to user.

---

### Q10: How do you handle polygon-based searches (e.g., restaurants in a neighborhood)?

Polygon search implementation:

- **PostGIS**:
  SELECT * FROM restaurants WHERE ST_Contains(ST_GeomFromGeoJSON('{polygon}'), location), polygon format: GeoJSON {'type': 'Polygon', 'coordinates': [[[lng1, lat1], [lng2, lat2], [lng3, lat3], [lng1, lat1]]]}, must close loop (first point = last point), GIST index supports containment queries O(log n).

- **Elasticsearch**:
  { 'query': { 'geo_polygon': { 'location': { 'points': [ {'lat': 12.97, 'lon': 77.59}, {'lat': 12.98, 'lon': 77.60}, {'lat': 12.97, 'lon': 77.61} ] } } } }, supports up to 1000 vertices, automatically closes polygon. Optimization:

- **Bounding box pre-filter**:
  Calculate min/max lat/lng of polygon (bounding rectangle), pre-filter with WHERE lat BETWEEN min_lat AND max_lat AND lng BETWEEN min_lng AND max_lng (fast), then exact containment check ST_Contains (slower), reduces candidates by 90%,

- **Polygon simplification**:
  For complex polygons (1000+ vertices), use Douglas-Peucker algorithm to reduce to 100-200 vertices while maintaining shape, 5-10x faster queries.

- **Pre-computation**:
  For static zones (delivery areas, neighborhoods), assign zone_id to each location, query becomes WHERE zone_id = {id} (instant via index). Use cases:

- **Delivery zones**:
  Restaurant serves specific polygonal area,

- **Geofencing**:
  Driver dispatch only within polygon boundary,

- **Neighborhood search**:
  Find restaurants in 'Greenwich Village' (irregular shape). Performance: Simple polygon (10 vertices) with 10M points → ~50ms with GIST index, Complex polygon (100 vertices) → ~200ms. Example: Uber Eats delivery zone → restaurant draws polygon on map → frontend converts to GeoJSON → backend stores in zones table → orders only accepted from customers within ST_Contains(delivery_zone, customer_location).

---

## 7. Interview Summary & Architectural Decision Matrix

| Technology | Data Structure / Index | Strengths | Limitations | When to Use in System Design |
| :--- | :--- | :--- | :--- | :--- |
| **Quad Tree** | In-Memory 4-Way Balanced Tree | Intuitive conceptual model for interviews; hierarchical spatial subdivision. | Severe height imbalance on real maps; expensive parent-child backtracking; hard to balance & sync in distributed memory. | **Avoid in production**. Only mention to show theoretical knowledge or for small in-memory single-node game engines. |
| **Geohash (Base32)** | Space-Filling Z-Order Morton Code (String/Int) | Converts 2D spatial search to 1D string prefix search (`LIKE 'tdr1w%'`); easy to shard; DB agnostic. | Bounding box edge boundary clipping (must query 8 neighbor cells); rectangular cells distort at poles. | **Dynamic location pipelines**, distributed sharding key in Cassandra/DynamoDB, and Redis Geo. |
| **PostgreSQL + PostGIS** | GiST Index (Generalized Search Tree / R-Tree) | ACID transactions, native SQL `ST_DWithin`, true geodesic spherical distance; combines with `pg_trgm`. | Vertical scaling limits; heavy write overhead under millions of GPS updates/sec. | **Authoritative POI store**, low-to-medium write throughput, apps with complex relational joins. |
| **Elasticsearch** | Lucene BKD-Tree (Block k-d tree) | Best-in-class compound text search + geo distance filtering + faceted aggregations; horizontal clustering. | Eventual consistency; higher memory consumption; operational overhead of cluster maintenance. | **E-Commerce & POI Discovery (Zomato, Yelp, Airbnb)** where users search text + filters + distance. |
| **Redis GEO** | 52-bit Geohash stored in Sorted Set (ZSET) | Microsecond latency ($< 1\text{ms}$); handles $100\text{k}+$ updates/sec easily; built-in `GEOSEARCH`. | Pure in-memory (RAM expensive); lacks complex text filtering or multi-criteria queries. | **Real-time Ride Hailing & Delivery Tracking (Uber, Swiggy, Rapido)** for fast-moving driver coordinates. |

---

### Bunny's Concluding Interview Blueprint Rule
> *"If the interview is **Uber or Rapido** (finding nearest moving cabs), design the pipeline with **Redis GEO + Geohash sharded by city**.  
> If the interview is **Zomato, Swiggy, or Airbnb** (searching restaurants/hotels by name, cuisine, and radius), design with **Elasticsearch or PostgreSQL with PostGIS and pg_trgm**."*
