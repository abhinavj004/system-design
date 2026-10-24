# System Design 03: Design E-Commerce Platform (Amazon / Flipkart)

> **Source**: [Interview with Bunny - Episode 3](https://www.youtube.com/watch?v=-wJuExkI97s)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/03](https://www.interviewwithbunny.com/systemdesign/03)  
> **Topic**: Designing an End-to-End E-Commerce Architecture (Amazon, Flipkart)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Product Search & Catalog Browsing**: Users can search for products using keywords, categories, and dynamic faceted filters (brand, price range, ratings, specifications).
2. **Product Details View**: Users can view rich product descriptions, dynamic attributes, high-resolution media, seller info, and real-time inventory availability.
3. **Cart Management**: Users can add, update quantities, view, and remove items in their shopping cart across multiple sessions.
4. **Checkout & Order Placement**: Users can proceed to checkout, select shipping addresses, apply discounts/coupons, and place orders with strict idempotency (no double-charging).
5. **Payment Processing**: Seamless integration with external payment gateways (Credit Card, Debit Card, UPI, NetBanking, COD).
6. **Order Tracking & Notifications**: Users can view past orders, track live shipment status, and receive real-time email/SMS/push notifications.

---

### 1.2 Non-Functional Requirements (NFR)
1. **High Availability for Browse Path (AP)**:
   - Product catalog browsing and search must never go down (target: 99.99% availability). High read availability is favored over strict consistency.
2. **Strict Consistency for Checkout & Inventory (CP)**:
   - **Zero Overselling**: The system must guarantee that inventory cannot be negative or double-sold under high concurrent purchasing.
   - Financial transactions and order creation must be strictly ACID compliant.
3. **Low Latency Target**:
   - Product search & catalog rendering $< 100\text{ms}$.
   - Checkout processing $< 500\text{ms}$.
4. **Massive Scale**:
   - Scale for 100M+ Monthly Active Users with massive traffic spikes during festival sales (e.g., Amazon Prime Day / Flipkart Big Billion Days).
5. **Idempotency & Resilience**:
   - Network retries, timeout reconnections, or rapid double-clicks on "Pay Now" must never create duplicate charges or split orders.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)
- **Scale**: 100 Million Active Users; 100 Million distinct Product SKUs.
- **Traffic**:
  - Read-heavy workload: $100:1$ Read-to-Write ratio.
  - Browse / Search QPS $\approx 50,000 \text{ QPS}$ (Peaking at $150,000\text{ QPS}$ during sales).
  - Checkout / Order QPS $\approx 500 \text{ QPS}$ (Peaking at $5,000\text{ QPS}$).
- **Storage**:
  - 100M Products $\times$ 5 KB metadata $\approx 500 \text{ GB}$ (metadata fits easily on distributed document clusters).
  - Images & Videos: Offloaded entirely to Object Storage (**AWS S3 / GCS**) with CDN edge caching.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|        User        |        |      Product       |        |     Inventory      |
+--------------------+        +--------------------+        +--------------------+
| id (UUID)          | 1    * | id (UUID)          | 1    1 | product_id (PK, FK)|
| name, email, phone |<------>| title, description |<------>| total_stock (INT)  |
| saved_addresses    |        | category, brand    |        | reserved_stock(INT)|
+--------------------+        | attributes (JSONB) |        | available_stock    |
                              | price (DECIMAL)    |        +--------------------+
                              +--------------------+
                                      | 1
                                      | *
+--------------------+        +--------------------+        +--------------------+
|     CartItem       | *    1 |     OrderItem      | *    1 |       Order        |
+--------------------+<------>+--------------------+<------>+--------------------+
| cart_id (UUID)     |        | id (UUID)          |        | id (UUID)          |
| user_id (UUID)     |        | order_id (FK)      |        | user_id (FK)       |
| product_id (FK)    |        | product_id (FK)    |        | total_amount       |
| quantity (INT)     |        | price_at_purchase  |        | order_status (ENUM)|
+--------------------+        | quantity (INT)     |        | payment_id (FK)    |
                              +--------------------+        +--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Search & Browse Products
```http
GET /api/v1/products/search?q=macbook+pro&category=laptops&min_price=1000&max_price=3000&page=1
Response: 200 OK
{
  "total_hits": 1420,
  "page": 1,
  "products": [
    {
      "product_id": "p_mbp_16",
      "title": "Apple MacBook Pro 16-inch M3 Max",
      "price": 2499.00,
      "thumbnail_url": "https://cdn.amazon.com/images/p_mbp_16.jpg",
      "rating": 4.8,
      "in_stock": true
    }
  ]
}
```

---

### 3.2 Add to Cart
```http
POST /api/v1/cart/items
Authorization: Bearer <jwt>
Content-Type: application/json

Request Body:
{
  "product_id": "p_mbp_16",
  "quantity": 1
}

Response: 200 OK
{
  "cart_id": "cart_9912",
  "total_items": 1,
  "subtotal": 2499.00
}
```

---

### 3.3 Create Order (Idempotent Checkout)
```http
POST /api/v1/orders/checkout
Authorization: Bearer <jwt>
X-Idempotency-Key: idemp_key_001_checkout

Request Body:
{
  "cart_id": "cart_9912",
  "shipping_address_id": "addr_441",
  "payment_method": "UPI"
}

Response: 201 Created
{
  "order_id": "ord_88291",
  "status": "PAYMENT_PENDING",
  "total_amount": 2499.00,
  "payment_gateway_payload": {
    "txn_token": "tok_upi_7718",
    "gateway_url": "https://gateway.razorpay.com/pay/ord_88291"
  }
}
```

---

### 3.4 Order Status & Tracking
```http
GET /api/v1/orders/{order_id}
Authorization: Bearer <jwt>

Response: 200 OK
{
  "order_id": "ord_88291",
  "status": "SHIPPED",
  "tracking_number": "TRK_BLUEDART_991",
  "estimated_delivery": "2026-10-22"
}
```

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/EcommerceSystemDesign%20(1).png?updatedAt=1746560605689)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 03](https://www.interviewwithbunny.com/systemdesign/03))*

### Component Interaction Topology
```
[ Web & Mobile Clients ]
           │
           ▼
[ Global CDN (CloudFront) ] (Caches product images, banners, reviews)
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                 API Gateway & Load Balancer                 │
│         (Authentication, Rate Limiter, Path Router)         │
└─────────────────────────────────────────────────────────────┘
      │               │                  │                 │
 (Search)          (Cart)            (Checkout)         (Orders)
      │               │                  │                 │
      ▼               ▼                  ▼                 ▼
┌───────────┐   ┌───────────┐      ┌───────────┐     ┌───────────┐
│  Search   │   │   Cart    │      │ Checkout  │     │   Order   │
│  Service  │   │  Service  │      │  Service  │     │  Service  │
└───────────┘   └───────────┘      └───────────┘     └───────────┘
      │               │                  │                 │
      ▼               ▼                  ▼                 ▼
┌───────────┐   ┌───────────┐      ┌───────────┐     ┌───────────┐
│ Elastic-  │   │   Redis   │      │ Inventory │     │Order SQL  │
│  search   │   │  Cluster  │      │  Service  │     │ (Postgres)│
└───────────┘   └───────────┘      └───────────┘     └───────────┘
      ▲                                  │                 │
      │ (CDC via Debezium)               ▼                 │
┌───────────┐                      ┌───────────┐           │
│  Product  │                      │ Inventory │           │
│ Catalog DB│                      │  DB (ACID)│           │
│ (MongoDB /│                      └───────────┘           │
│ Postgres) │                            │                 │
└───────────┘                            ▼                 ▼
                               ┌─────────────────────────────────┐
                               │     Apache Kafka Event Bus      │
                               └─────────────────────────────────┘
                                         │               │
                                         ▼               ▼
                               ┌─────────────────┐ ┌─────────────┐
                               │ Payment Gateway │ │Notification │
                               │   Webhook Hub   │ │   Service   │
                               └─────────────────┘ └─────────────┘
```

---

### 4.2 Step-by-Step Architectural Evolution (How to Drive the Interview Whiteboard)

In a FAANG system design interview, **never draw the completed microservices diagram all at once**. Instead, build it progressively on the whiteboard by proposing an initial baseline and then introducing each component to solve a concrete bottleneck:

#### Phase 1: The Naive Baseline MVP (Monolith & Single Relational Database)
- **What to draw on the whiteboard**: `Client` ──► `Load Balancer` ──► `Monolithic E-Commerce API Server` ──► `Single PostgreSQL Database`.
- **How to explain the flow to the interviewer**:
  - *"Let's first trace the core user journey: a user searches for products, adds an item to a cart table, and checks out in a single ACID transaction (`SELECT qty` -> `UPDATE products SET qty = qty - 1` -> `INSERT INTO orders`)."*
- **Bottlenecks identified (Why it breaks at scale)**:
  1. *Search Bottleneck*: Querying 100M products using SQL `WHERE title ILIKE '%shoes%'` requires a full table scan, choking the database CPU.
  2. *Schema Rigidity*: Products have wildly differing attributes (shoes have sizes, laptops have RAM/CPU, apparel has fabrics). Rigid SQL schemas need hundreds of nullable columns or slow EAV joins.
  3. *Flash Sale Contention*: 10,000 concurrent checkout transactions attempting to decrement the same inventory row result in severe lock contention, deadlocks, and overselling.
  4. *Synchronous Cascading Failures*: Synchronous HTTP calls to payment processors (e.g., Stripe taking 3–5 seconds) block web server worker threads, leading to thread pool starvation and 504 Gateway Timeouts.

#### Phase 2: Decouple into Domain Microservices & In-Memory Caching
- **What to add next**: Split the monolith into `Product Catalog Service`, `Cart Service`, `Order Service`, and `Inventory Service`. Add **Redis Cluster**.
- **How to explain the transition**:
  - *"To prevent a failure in payment processing from bringing down product browsing, we decompose the system into domain microservices. Cart state is ephemeral, high-frequency, and device-bound—storing it in PostgreSQL adds unnecessary disk I/O, so we offload active carts to an in-memory Redis Cluster ($< 2	ext{ms}$ latency) with an automatic 30-day TTL."*

#### Phase 3: Polyglot Persistence & Fast Search Engine
- **What to add next**: Add **Elasticsearch Cluster** for search queries, and use **MongoDB / PostgreSQL JSONB** for the Product Catalog.
- **How to explain the transition**:
  - *"We separate read-heavy search from transactional writes. Elasticsearch provides an inverted index supporting typo tolerance, fuzzy matching, and multi-faceted filtering (category, brand, price bracket) in sub-50ms. For product catalog storage, document-based JSONB allows sellers to define arbitrary schema attributes without running database migrations."*

#### Phase 4: Change Data Capture (CDC Pipeline) for Search Synchronization
- **What to add next**: Connect Product DB to Elasticsearch via **Debezium CDC + Apache Kafka + Elasticsearch Sink**.
- **How to explain the transition**:
  - *"If the Product Service performs dual-writes (writing to MongoDB and then writing to Elasticsearch), network failures or crashes between the two writes cause persistent data drift. Instead, we implement Change Data Capture (CDC) using Debezium to tail MongoDB's change stream into Kafka. The Elasticsearch sink consumer updates the index with guaranteed at-least-once delivery and sub-second lag."*

#### Phase 5: Asynchronous Event Bus (Apache Kafka) for Decoupled Checkout
- **What to add next**: Place an **Apache Kafka Event Bus** between Checkout, Inventory, Payment, and Notification services.
- **How to explain the transition**:
  - *"Checkout must not block on downstream side-effects. When checkout initiates, the Order Service reserves stock and emits an `OrderCreatedEvent` to Kafka. Payment Gateway Webhook Hub, Notification Service (email/SMS receipts), Analytics, and Warehouse Fulfillment listen independently. If the third-party email provider experiences an outage, order processing continues unaffected."*

#### Phase 6: Edge Caching & Concurrency Safeguards
- **What to add next**: Add **Global CDN (CloudFront)**, **API Gateway Rate Limiter**, and **Redis Distributed Locks (`SETNX`)**.
- **How to explain the transition**:
  - *"To minimize origin traffic, 90% of static product images and CSS assets are cached globally at CloudFront edge PoPs. For high-volume flash sales (100K orders/minute), requests pass through an API Gateway token-bucket rate limiter. Inventory checks utilize Redis `SETNX` distributed holds with a 30-second lease time, shielding the relational database from unconstrained lock contention."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Dynamic Product Catalog: Why MongoDB / PostgreSQL JSONB?
E-commerce catalogs face **extreme schema heterogeneity**:
- *Shoes*: Size (8, 9, 10), Color, Sole material.
- *Laptops*: CPU, RAM, GPU, Screen refresh rate, Storage.
- *Apparel*: Fabric, Wash care, Fit type.

#### Bunny's Architectural Solution:
- A rigid relational schema requires hundreds of nullable columns or a complex Entity-Attribute-Value (EAV) table that destroys query performance.
- **Chosen Store**: **MongoDB (Document Store)** or **PostgreSQL with `JSONB`**:
  ```json
  {
    "_id": "p_mbp_16",
    "title": "Apple MacBook Pro 16",
    "category": "Computers & Accessories",
    "brand": "Apple",
    "base_price": 2499.00,
    "attributes": {
      "ram_gb": 36,
      "storage_ssd": "1TB",
      "chip": "M3 Max",
      "color": "Space Black"
    }
  }
  ```
- **Syncing to Search (CDC Pipeline)**:
  - Updates in MongoDB/PostgreSQL are captured via **Debezium (Change Data Capture)** and pushed to an **Apache Kafka** topic.
  - An Elasticsearch ingest worker transforms and indexes documents in real time, keeping faceted search indices updated within $< 1\text{ second}$.

---

### 5.2 Preventing Inventory Overselling (Atomic Reservation)
*Problem*: 1,000 users attempt to purchase the last 5 units of an iPhone during a flash drop.

#### Bunny's Concurrency Architecture:
1. **Inventory Schema (Relational PostgreSQL)**:
   ```sql
   CREATE TABLE product_inventory (
       product_id UUID PRIMARY KEY,
       total_stock INT NOT NULL,
       reserved_stock INT NOT NULL DEFAULT 0,
       CHECK (total_stock >= reserved_stock) -- Enforces zero overselling at DB engine level
   );
   ```
2. **Atomic Stock Reservation**:
   ```sql
   UPDATE product_inventory 
   SET reserved_stock = reserved_stock + :order_qty 
   WHERE product_id = :product_id 
     AND (total_stock - reserved_stock) >= :order_qty;
   ```
   - If the update returns `1 row affected`, the inventory hold is secured.
   - If it returns `0 rows affected`, stock is exhausted $\implies$ immediately return `409 Conflict (Out of Stock)` to the client without proceeding to payment.

---

### 5.3 Distributed Transactions: The Saga Pattern
A monolithic 2-Phase Commit (2PC) blocks databases across network boundaries and does not scale. Bunny recommends the **Saga Pattern (Choreography / Orchestration via Kafka)**:

```
[ Checkout Service ] ──▶ (1. Reserve Stock) ──▶ [ Inventory Service ]
                                                        │ (Success)
                                                        ▼
[ Notification Svc ] ◀── (3. Order Confirmed) ◀── [ Payment Service ]
        ▲                                               │ (Payment Failed)
        │                                               ▼
        └─────────────── (Compensating Event: ──────────┘
                          Rollback Reserved Stock)
```

1. **Step 1**: Checkout Service emits `OrderInitiatedEvent`.
2. **Step 2**: Inventory Service consumes event and reserves stock. If stock is unavailable, emits `OrderFailedEvent` $\to$ abort.
3. **Step 3**: Payment Service attempts to charge user:
   - **Payment Success**: Emits `PaymentCompletedEvent`. Order Service transitions order to `CONFIRMED`.
   - **Payment Failure / Timeout**: Emits `PaymentFailedEvent`.
4. **Compensating Transaction**: Inventory Service consumes `PaymentFailedEvent` and executes an automatic rollback:
   ```sql
   UPDATE product_inventory 
   SET reserved_stock = reserved_stock - :order_qty 
   WHERE product_id = :product_id;
   ```
   *Guarantees*: Eventual consistency across microservices without distributed table deadlocks.

---

### 5.4 End-to-End Idempotency
- Every checkout submission includes an `X-Idempotency-Key` (UUIDv4 generated by client frontend).
- **Execution in Redis**:
  ```redis
  SET idempotency:{key} "IN_PROGRESS" NX EX 120
  ```
  - If user double-clicks or browser retries on network lag, subsequent calls with the same key are rejected or wait for the initial transaction response.
  - Once order creation succeeds, the order ID is saved under `idempotency:{key}` with a 24-hour TTL.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you prevent overselling when multiple users try to buy the last item?

Multi-layer approach with distributed locks + database pessimistic locking (from image's 'check the stock availability before checkout' with Redis Lock):

- **Redis distributed lock BEFORE inventory check**:
  During checkout, FOR EACH product in cart: SETNX lock:inventory:{product_id} {checkout_session_id} EX 30, if SETNX returns 0 (lock exists) → wait and retry or return 'Another user is checking out this item, please wait', if returns 1 (acquired) → proceed to step 2,

- **Database pessimistic lock**:
  SELECT qty, reserved_qty FROM inventory WHERE product_id={product_id} FOR UPDATE (row-level exclusive lock),

- **Validate**:
  if qty < cart_qty → Release Redis lock, return 'Insufficient stock',

- **Reserve inventory**:
  UPDATE inventory SET reserved_qty = reserved_qty + {cart_qty} WHERE product_id={product_id}, this removes items from available pool but doesn't deduct yet (payment might fail),

- **Release Redis lock**:
  DEL lock:inventory:{product_id},

- **Point 6**: Create order with status='PENDING_PAYMENT',

- **Payment webhook success**:
  UPDATE inventory SET qty = qty - {cart_qty}, reserved_qty = reserved_qty - {cart_qty}, now actually deduct from stock,

- **Payment failure**:
  UPDATE inventory SET reserved_qty = reserved_qty - {cart_qty}, release reservation. Why both locks?: Redis lock is fast (1-2ms) early rejection for concurrent requests (100 users trying to checkout simultaneously), PostgreSQL FOR UPDATE ensures ACID guarantees within transaction, prevents phantom reads. Lock expiry: Redis lock auto-expires in 30s if service crashes, prevents deadlock. Example: Product has qty=1, User A and User B both click checkout → User A's SETNX succeeds → User B's SETNX fails (waits) → User A reserves qty → User B's SETNX succeeds after A releases → User B checks qty=0 (reserved=1) → Returns 'Out of stock' → No overselling. Reserved_qty column explained: qty=10, reserved_qty=3 means 10 total in warehouse, 3 in active checkouts, 7 available for new purchases, prevents showing items as available when they're locked in checkout flow.

---

### Q2: How do you implement product search with filters (category, price, rating)?

Elasticsearch-based search with CDC sync (from image shows Product → Sync Kafka → Elasticsearch):

- **Index mapping**:
  PUT /products { 'mappings': { 'properties': { 'product_id': {'type': 'keyword'}, 'title': {'type': 'text', 'analyzer': 'standard', 'fields': {'keyword': {'type': 'keyword'}}}, 'description': {'type': 'text'}, 'category': {'type': 'keyword'}, 'price': {'type': 'float'}, 'rating': {'type': 'float'}, 'brand': {'type': 'keyword'}, 'qty': {'type': 'integer'} } } },

- **CDC pipeline (from image)**:
  MongoDB product updates → Kafka 'product.updated' topic → ES indexer consumer → index.update(product_id, updated_fields), keeps ES in sync within 1-2 seconds,

- **Search query**:
  GET /v1/product/search?q=laptop&category=electronics&priceMin=500&priceMax=1500&rating>=4 → Backend builds: POST /products/_search { 'query': { 'bool': { 'must': [ { 'multi_match': { 'query': 'laptop', 'fields': ['title^3', 'description', 'brand'], 'type': 'best_fields' } } ], 'filter': [ { 'term': { 'category': 'electronics' } }, { 'range': { 'price': { 'gte': 500, 'lte': 1500 } } }, { 'range': { 'rating': { 'gte': 4 } } }, { 'range': { 'qty': { 'gt': 0 } } } ] } }, 'sort': [ { 'rating': 'desc' } ], 'from': 0, 'size': 20 },

- **Scoring**:
  'title^3' boosts title matches 3x over description, relevance score combines text match + filters,

- **Faceted search**:
  Add aggregations: 'aggs': { 'categories': { 'terms': { 'field': 'category' } }, 'price_ranges': { 'range': { 'field': 'price', 'ranges': [{'to': 500}, {'from': 500, 'to': 1000}, {'from': 1000}] } }, 'brands': { 'terms': { 'field': 'brand', 'size': 20 } } } → Returns counts: {Electronics: 5000, Books: 3000, ...} for UI filter checkboxes,

- **Autocomplete**:
  Separate index with edge_ngram tokenizer, queries on every keystroke (debounced 300ms), suggests 'laptop', 'laptop bag', 'laptop stand',

- **Synonyms**:
  Configure analyzer: 'laptop' → ['laptop', 'notebook', 'portable computer'], improves recall,

- **Caching**:
  Redis caches popular queries (key: hash(query + filters + sort)) with TTL=10 min, serves 60-70% from cache,

- **Performance**:
  ES cluster 10 nodes, 5 primary shards (100M products / 5 = 20M per shard), query p95 <100ms. Why Elasticsearch over PostgreSQL full-text?: ES designed for full-text search with relevance scoring, fuzzy matching, synonyms, faceted search - PostgreSQL's tsvector can't match this. MongoDB vs PostgreSQL for products?: Image shows MongoDB (DocumentDB) because different product types have vastly different schemas (electronics: specs={RAM, processor}, books: {author, ISBN}, clothing: {size, color}) - flexible schema avoids sparse columns and NULL hell in relational DB.

---

### Q3: How do you manage shopping cart state across sessions and devices?

Hybrid Redis + PostgreSQL cart management (from image shows Cart DB PostgreSQL + Redis caching):

- **Cart creation**:
  User logs in → Check if cart exists: SELECT cart_id FROM carts WHERE user_id={user_id}, if not: INSERT INTO carts (cart_id, user_id, created_at) VALUES ({uuid}, {user_id}, now()),

- **Add to cart**:
  POST /v1/cart/add with {product_id, qty: 2}, (a) Validate product exists and has stock (query Product Service), (b) PostgreSQL update: INSERT INTO cart_items (cart_id, product_id, qty, price, currency) VALUES ({cart_id}, {product_id}, 2, 899, 'USD') ON CONFLICT (cart_id, product_id) DO UPDATE SET qty = cart_items.qty + 2, updated_at = now(), (c) Redis cache: HSET cart:{user_id} product:{product_id} '{qty: 2, price: 899, currency: USD, added_at: timestamp}' with TTL=7 days,

- **Get cart**:
  (a) Check Redis: HGETALL cart:{user_id}, if hit: return cached data (1ms latency), (b) If miss: Query PostgreSQL: SELECT ci.product_id, ci.qty, ci.price, ci.currency, p.title, p.image_url FROM cart_items ci JOIN products p ON ci.product_id = p.product_id WHERE ci.cart_id={cart_id}, then cache in Redis,

- **Update quantity**:
  PATCH /v1/cart/items/{product_id} with {qty: 5} → Update PostgreSQL and Redis atomically,

- **Remove item**:
  DELETE /v1/cart/items WHERE cart_id={cart_id} AND product_id={product_id} → Also Redis HDEL cart:{user_id} product:{product_id},

- **Cross-device sync**:
  Cart tied to user_id (not session_id), user logs in on mobile → same cart appears (fetched from PostgreSQL), Redis cache per user ensures fast access,

- **Guest cart**:
  For non-logged-in users, use session_id instead of user_id, store in Redis only with TTL=24h, on login: migrate guest cart to user cart: UPDATE cart_items SET cart_id={user_cart_id} WHERE cart_id={guest_cart_id},

- **Cart expiry**:
  Background job deletes carts inactive >30 days: DELETE FROM carts WHERE updated_at < now() - INTERVAL 30 DAY,

- **Price updates**:
  If product price changes after adding to cart, show both: 'Added at $899, now $799', let user decide to keep or remove,

- **Stock validation at checkout**:
  Before payment, re-validate all items still in stock, if any out of stock → remove from cart, notify user. Why PostgreSQL for cart?: ACID guarantees (cart updates must be reliable), survives Redis cache evictions (source of truth), supports complex queries (JOIN with products for enriched data), relational integrity (cart → items FK). Redis for performance: Cart reads are 10x more frequent than writes (user browses, views cart multiple times before checkout), Redis serves reads in <1ms, PostgreSQL ~50ms, at scale 10K concurrent users × 100 cart reads/min = 1M reads/min → Redis essential. TTL strategy: 7 days for active carts (user likely to return), auto-cleanup prevents memory bloat, PostgreSQL keeps cart forever (until user deletes account) for business analytics.

---

### Q4: How do you handle the checkout flow from cart to order confirmation?

Multi-step checkout orchestration (from image shows Checkout → Redis Lock → Kafka flow):

- **Initiate checkout**:
  POST /v1/checkout with {cart_id, shipping_address_id, payment_method_id},

- **Checkout Service orchestration**:
  Step 1: Fetch cart items: SELECT product_id, qty, price FROM cart_items WHERE cart_id={cart_id}, Step 2: Acquire distributed locks: FOR EACH product in cart: SETNX lock:inventory:{product_id} {checkout_session_id} EX 30, if any lock fails → abort, release acquired locks, return 'Product being checked out by another user', Step 3: Validate inventory (with DB lock): FOR EACH product: SELECT qty, reserved_qty FROM inventory WHERE product_id={product_id} FOR UPDATE, if qty < cart_qty → abort, release locks, return 'Product X out of stock', Step 4: Reserve inventory: UPDATE inventory SET reserved_qty = reserved_qty + {cart_qty} WHERE product_id={product_id}, Step 5: Calculate total: total = SUM(price × qty) + shipping_fee + tax, Step 6: Create order: BEGIN TRANSACTION; INSERT INTO orders (order_id, user_id, items, total, status='PENDING_PAYMENT', shipping_address, created_at); COMMIT;, Step 7: Release Redis locks: FOR EACH product: DEL lock:inventory:{product_id}, Step 8: Create payment intent: Call Payment Service → payment_gateway.createPaymentIntent({amount: total, currency, idempotency_key: order_id}), Step 9: Return to client: {order_id, payment_url, expires_at: now() + 15min},

- **User completes payment**:
  Redirected to Payment Gateway, enters card details, confirms payment,

- **Payment webhook**:
  POST /webhooks/payment with {order_id, payment_id, status: 'success'}, (a) Idempotency check: SELECT * FROM payments WHERE order_id={order_id}, if exists return 200 (already processed), (b) BEGIN TRANSACTION; INSERT INTO payments (payment_id, order_id, amount, status='SUCCESS', timestamp); UPDATE orders SET status='PAYMENT_CONFIRMED', payment_id={payment_id} WHERE order_id={order_id}; UPDATE inventory SET qty = qty - {cart_qty}, reserved_qty = reserved_qty - {cart_qty} WHERE product_id IN ({order_product_ids}); DELETE FROM cart_items WHERE cart_id={cart_id}; COMMIT;, (c) Publish Kafka events: 'payment.success' topic: {order_id, user_id, items, total, timestamp}, 'order.confirmed' topic: {order_id, items, shipping_address, warehouse_id},

- **Kafka consumers**:
  (a) Notification Service → sends email/SMS 'Order confirmed', (b) Inventory Service → updates Redis cache (invalidate product:{product_id} cache, update inventory:{product_id} qty), (c) Warehouse Service → creates pick list for order fulfillment,

- **Payment timeout**:
  Background job checks orders with status='PENDING_PAYMENT' created >15 min ago → Release reserved inventory: UPDATE inventory SET reserved_qty = reserved_qty - {cart_qty}, order status → 'PAYMENT_TIMEOUT', send notification 'Checkout expired, items returned to stock',

- **Failure scenarios**:
  (a) Inventory insufficient during reserve → abort checkout, don't create order, release locks, (b) Payment fails (card declined) → webhook 'payment.failed' → release reserved_qty, order status='PAYMENT_FAILED', (c) Service crash during checkout → Redis locks auto-expire after 30s, reserved_qty released by timeout job. Why this flow?:

- **Point 1**: Locks prevent race conditions on limited stock,

- **Point 2**: Reserved_qty column ensures stock not shown as available during 15min payment window,

- **Point 3**: Idempotency prevents double charging on webhook replay,

- **Point 4**: Kafka decouples payment from fulfillment (warehouse can be slow, doesn't block checkout),

- **Transaction ensures atomicity (payment recorded + inventory deducted + cart cleared all-or-nothing). Performance**:
  Average checkout time 2-3 seconds (lock acquisition ~10ms, DB queries ~50ms, payment intent creation ~200ms), can handle 10K checkouts/min with horizontal scaling of Checkout Service.

---

### Q5: How do you implement CDC (Change Data Capture) for syncing Product DB to Elasticsearch?

CDC pipeline for near real-time search index updates (from image shows Product → Sync Kafka → Elasticsearch):

- **CDC source setup**:
  (a) MongoDB: Enable oplog (operations log) which records all write operations, use MongoDB Change Streams to listen: db.products.watch([{$match: {operationType: {$in: ['insert', 'update', 'replace', 'delete']}}}]), (b) PostgreSQL (for other tables): Enable logical replication, use Debezium connector to capture WAL (Write-Ahead Log) changes,

- **Kafka connector**:
  (a) MongoDB Change Stream → Kafka Connect MongoDB Source Connector → publishes to Kafka topic 'product.changes', (b) Event schema: {operation: 'update', product_id: 'LAPTOP_123', fullDocument: {product_id, title, price, qty, ...}, timestamp},

- **Elasticsearch indexer consumer**:
  (a) Kafka Consumer Group 'es-product-indexer' subscribes to 'product.changes', (b) Consumer code: for event in kafka.consume(): if event.operation == 'insert' or 'update': es.index(index='products', id=event.product_id, body=event.fullDocument), elif event.operation == 'delete': es.delete(index='products', id=event.product_id), (c) Batch indexing: Buffer 100 events, bulk index to ES every 2 seconds for efficiency: es.bulk(operations=[...]),

- **Error handling**:
  (a) ES indexing fails → Log error, push event to DLQ (Dead Letter Queue), retry with exponential backoff, (b) Kafka consumer lag monitoring: Alert if lag >1000 events (indexer falling behind), auto-scale consumer instances,

- **Initial load**:
  On first setup, bulk export all products from MongoDB → Elasticsearch: db.products.find().forEach(doc => es.index({...})), then start CDC to keep in sync,

- **Consistency**:
  (a) Eventual consistency: Product updated in MongoDB → 1-2 seconds later appears in ES search, acceptable for e-commerce search, (b) Ordering guarantee: Kafka partition by product_id ensures updates for same product processed in order,

- **Performance**:
  (a) MongoDB oplog tail: <10ms latency to capture change, (b) Kafka latency: ~50ms end-to-end, (c) ES indexing: Bulk API can index 10K docs/sec, (d) Total lag: Product update → searchable in 1-2 seconds,

- **Why CDC over direct writes?**:
  (a) Separation of concerns: Product Service doesn't need to know about ES, (b) Fault tolerance: If ES down, MongoDB still accepts writes, indexer catches up later, (c) Replay: Can rebuild entire ES index from Kafka topic (retention 7 days), (d) Multiple consumers: Same events used for Redis cache invalidation, analytics, audit logs. Alternative: Dual writes (Product Service writes to both MongoDB and ES) - bad because:

- **Point 1**: Tight coupling,

- **Point 2**: Transaction complexity (what if ES write fails?),

- **Product Service needs to know search implementation. Real-world**:
  Amazon uses similar CDC pipeline with Kinesis Streams, millions of product updates/hour, ES cluster 100+ nodes, <5 second lag p99.

---

### Q6: How do you design the payment integration with idempotency and webhook handling?

Payment flow with strong consistency guarantees (from image shows Payment Svc → Payment Gateway, Kafka → Consumer updates payment info):

- **Payment initiation**:
  Checkout Service calls Payment Service: POST /v1/payments with {order_id, amount: 1499.99, currency: 'USD', user_id, return_url},

- **Payment Service creates intent**:
  (a) Generate idempotency_key = order_id (ensures same order can't be charged twice), (b) Call Stripe/Razorpay API: stripe.paymentIntents.create({ amount: 149999, currency: 'usd', customer: stripe_customer_id, metadata: {order_id, user_id}, idempotency_key: order_id }), (c) Stripe returns: {payment_intent_id, client_secret, status: 'requires_payment_method'}, (d) Store in Payment DB: INSERT INTO payments (payment_id, order_id, amount, currency, status='PENDING', created_at) VALUES ({payment_intent_id}, {order_id}, 1499.99, 'USD', now()), (e) Return to client: {payment_url: 'https://checkout.stripe.com/...'},

- **User payment flow**:
  User redirected to Stripe checkout page, enters card details, Stripe processes (3D Secure if needed), on success: Stripe redirects back to return_url with payment_intent_id,

- **Webhook handling (CRITICAL)**:
  Stripe sends POST /webhooks/payment with {type: 'payment_intent.succeeded', data: {object: {id: payment_intent_id, amount, status: 'succeeded', metadata: {order_id}}}}, (a) Signature validation: Extract stripe_signature header, verify HMAC: stripe.webhooks.constructEvent(payload, signature, webhook_secret), prevents replay attacks and unauthorized requests, (b) Idempotency check: SELECT * FROM payments WHERE payment_id={payment_intent_id}, if status='SUCCESS' → return 200 OK (webhook already processed, duplicate event), (c) Process payment: BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE; UPDATE payments SET status='SUCCESS', gateway_response=jsonb({webhook_payload}), updated_at=now() WHERE payment_id={payment_intent_id} AND status='PENDING'; if affected_rows == 0: ROLLBACK; return 200 (concurrent webhook, lost race); UPDATE orders SET status='PAYMENT_CONFIRMED', payment_id={payment_intent_id}, paid_at=now() WHERE order_id={order_id}; UPDATE inventory SET qty = qty - {cart_qty}, reserved_qty = reserved_qty - {cart_qty} WHERE product_id IN (SELECT product_id FROM order_items WHERE order_id={order_id}); COMMIT;, (d) Publish Kafka events: 'payment.success': {order_id, payment_id, amount, timestamp}, 'order.confirmed': {order_id, user_id, items, total}, (e) Return 200 OK to Stripe (acknowledges receipt, Stripe won't retry),

- **Kafka consumers (from image shows Kafka → Consumer)**:
  (a) Notification Service: Sends email 'Payment successful, order #{order_id} confirmed', (b) Inventory Service: Invalidates Redis cache for affected products, (c) Order Fulfillment: Creates warehouse pick list,

- **Failure scenarios**:
  (a) Payment declined: Webhook 'payment_intent.payment_failed' → UPDATE orders SET status='PAYMENT_FAILED', release reserved inventory, notify user, (b) Webhook timeout: Stripe retries webhook 3 times (exponential backoff), if still fails → manual reconciliation via admin dashboard or polling Stripe API, (c) Database transaction fails: ROLLBACK, return 500 to Stripe → Stripe retries webhook → eventual consistency,

- **Idempotency guarantees**:
  (a) order_id as idempotency_key: User clicks 'Pay' multiple times → Stripe creates single payment_intent, subsequent calls return existing intent, (b) Database UNIQUE constraint: UNIQUE INDEX on payments(order_id) ensures one payment per order, prevents double charge on race condition, (c) Webhook deduplication: Check payment.status before processing, prevents double inventory deduction on webhook replay,

- **Reconciliation**:
  Nightly job compares Payment DB with Stripe API: stripe.paymentIntents.list({created: {gte: yesterday}}), flags discrepancies (payment succeeded in Stripe but order still PENDING_PAYMENT in DB) → admin investigation,

- **Testing**:
  Use Stripe test webhooks: stripe trigger payment_intent.succeeded, verify idempotency: send same webhook twice, assert order processed once. Security:

- **Point 1**: Webhook signature verification prevents forged webhooks,

- **Point 2**: HTTPS enforced for webhook endpoint,

- **Point 3**: Rate limiting 100 webhooks/min per IP,

- **Payment DB has separate credentials from other services (least privilege). Performance**:
  Payment webhook processing <200ms (transaction + Kafka publish), can handle 1000 payments/sec with horizontal scaling, Kafka decouples from slow consumers (email sending).

---

### Q7: How do you track order status and implement real-time updates for users?

Event-driven order tracking with Kafka (from image shows Order DB → Kafka → Consumer → Notification System):

- **Order lifecycle states**:
  PENDING_PAYMENT → PAYMENT_CONFIRMED → PROCESSING (warehouse) → SHIPPED → OUT_FOR_DELIVERY → DELIVERED → (optional) RETURNED/CANCELLED,

- **State transitions via Kafka events**:
  (a) Order creation: Checkout Service publishes 'order.created' to Kafka: {order_id, user_id, items, total, status: 'PENDING_PAYMENT', timestamp}, (b) Payment success: Payment webhook publishes 'payment.success' → Order Status Service consumes → UPDATE orders SET status='PAYMENT_CONFIRMED', (c) Warehouse picks items: Warehouse system POST /v1/orders/{order_id}/process → publishes 'order.processing' → Order Status Service updates → status='PROCESSING', (d) Shipment: Warehouse POST /v1/orders/{order_id}/ship with {tracking_id, carrier, estimated_delivery} → publishes 'order.shipped' → status='SHIPPED', (e) Delivery partner updates: External API calls POST /v1/orders/{order_id}/tracking with {location, status} → publishes 'order.out_for_delivery' or 'order.delivered',

- **Kafka topic structure**:
  Topics: 'order.lifecycle' (all state changes), 'order.tracking' (location updates), Event schema: {event_id, event_type: 'order.shipped', order_id, timestamp, payload: {tracking_id, carrier, estimated_delivery, warehouse_location}},

- **Consumer groups**:
  (a) Order Status Service: Updates Order DB with new status, stores history: INSERT INTO order_status_history (order_id, status, timestamp, metadata), (b) Notification Service: Sends email/SMS/push for each status change: 'Order confirmed', 'Order shipped - Track here', 'Out for delivery - ETA 2 PM', 'Delivered', (c) Analytics Service: Tracks metrics (avg delivery time, on-time rate),

- **Real-time updates to user**:
  Approach 1 - WebSocket: User opens order tracking page → Client establishes WebSocket: WS /v1/orders/{order_id}/track, Backend WebSocket Manager subscribes to Redis Pub/Sub channel: order:{order_id}:updates, Order Status Service publishes status changes to Redis channel, WebSocket Manager forwards to client's connection, Client displays real-time status and map (if location tracking), Approach 2 - Polling (fallback): Client polls GET /v1/status/{order_id} every 30 seconds, Less efficient but works if WebSocket unavailable,

- **Order status API**:
  GET /v1/status/{order_id} → Query Order DB: SELECT status, tracking_id, estimated_delivery, carrier FROM orders WHERE order_id={order_id}, Query status history: SELECT status, timestamp FROM order_status_history WHERE order_id={order_id} ORDER BY timestamp ASC, Response: {order_id, current_status: 'SHIPPED', tracking_id: 'TRK123', carrier: 'FedEx', estimated_delivery: '2025-01-25', history: [{status: 'PAYMENT_CONFIRMED', timestamp: '...'}, {status: 'PROCESSING', timestamp: '...'}, {status: 'SHIPPED', timestamp: '...'}]},

- **Location tracking integration**:
  Delivery partner API pushes updates: POST /v1/orders/{order_id}/location with {lat, lng, timestamp}, Order Tracking Service: Stores in time-series DB (InfluxDB): INSERT INTO delivery_locations (order_id, lat, lng, timestamp), Publishes to Redis Pub/Sub for real-time map updates, Client shows driver location on map, recalculates ETA using Google Maps Distance Matrix API,

- **Notification customization**:
  User preferences: Allow users to opt-in/out per notification type (email, SMS, push), send only opted-in channels, Template: Email template: 'Hi {user_name}, your order #{order_id} has been shipped! Track it here: {tracking_url}',

- **Cancellation flow**:
  User cancels: POST /v1/orders/{order_id}/cancel with {reason: 'changed_mind'}, Validation: if status in ['SHIPPED', 'OUT_FOR_DELIVERY', 'DELIVERED'] → return 400 'Cannot cancel, order already shipped', if status in ['PENDING_PAYMENT', 'PAYMENT_CONFIRMED', 'PROCESSING'] → allow cancellation, Update: UPDATE orders SET status='CANCELLED', cancelled_at=now(), cancellation_reason={reason}, Refund: If payment captured, initiate refund via Payment Gateway, Inventory: Release stock: UPDATE inventory SET qty = qty + {order_qty} WHERE product_id IN (order items), Kafka: Publish 'order.cancelled' → Notification Service sends 'Order cancelled, refund in 5-7 days',

- **SLA monitoring**:
  Track metrics: % orders delivered on time (status='DELIVERED' AND delivered_at <= estimated_delivery), avg time from PAYMENT_CONFIRMED to DELIVERED, % orders stuck in PROCESSING >24h (alert warehouse), Dashboard: Grafana shows real-time order funnel: PENDING

- **Point 100**: → CONFIRMED

- **Point 80**: → PROCESSING

- **Point 60**: → SHIPPED

- **Point 40**: → DELIVERED

- **. Performance**:
  Kafka handles 100K order events/sec, Order Status Service processes events in <100ms, WebSocket can support 1M concurrent connections (scaled horizontally), Real-time updates delivered <500ms from event publish to user's screen.

---

### Q8: How do you handle high-volume flash sales (100K orders in 1 minute)?

Flash sale architecture with queuing and rate limiting:

- **Pre-sale preparation**:
  (a) Product flagged as flash_sale=true with sale_start_time, sale_end_time, sale_quantity=1000, (b) Cache warm-up: Pre-load product details, images to CDN and Redis 1 hour before sale, (c) Database optimization: Dedicated read replicas for flash sale product queries, index on (product_id, sale_start_time), (d) Auto-scaling: Scale Checkout Service 5x normal capacity (50 → 250 instances) 10 min before sale,

- **Virtual waiting room**:
  (a) Before sale_start_time: Users directed to waiting room page (Cloudflare Waiting Room or custom), shows countdown timer, ~1M users in queue, (b) At sale_start_time: Waiting room releases users in batches (1000 users/sec) with random token, prevents thundering herd on backend, (c) Token validation: Backend accepts requests only with valid token from waiting room,

- **Inventory handling**:
  (a) Pre-allocate inventory: CREATE TABLE flash_sale_inventory (product_id, queue_position INT, user_id NULL, status ENUM('available', 'reserved', 'sold'), reserved_at TIMESTAMP), INSERT 1000 rows with status='available', queue_position=1 to 1000, (b) Checkout flow: User clicks 'Buy Now' → Backend: BEGIN TRANSACTION; SELECT queue_position, status FROM flash_sale_inventory WHERE product_id={product_id} AND status='available' ORDER BY queue_position LIMIT 1 FOR UPDATE SKIP LOCKED; if found: UPDATE SET status='reserved', user_id={user_id}, reserved_at=now(); COMMIT; else: ROLLBACK; return 'Sold out'; (c) SKIP LOCKED: PostgreSQL feature, if row locked by concurrent transaction, skip it and try next → prevents contention,

- **Queue-based checkout**:
  (a) Add to Kafka queue: Instead of synchronous checkout, publish to Kafka 'checkout.requests' topic: {user_id, product_id, timestamp}, FIFO guarantee within partition (partition by product_id), (b) Checkout workers: Consumer group of 50 workers processes queue at steady rate (100 checkouts/sec), prevents overwhelming Payment Gateway and Inventory DB, (c) User sees: 'You're in line, position #523. Estimated wait: 5 min', WebSocket updates queue position in real-time,

- **Timeout & release**:
  Reserved inventory timeout: Background job every 10 sec: UPDATE flash_sale_inventory SET status='available', user_id=NULL WHERE status='reserved' AND reserved_at < now() - INTERVAL 5 MINUTE, if user doesn't complete payment in 5 min, inventory released for next user,

- **Rate limiting**:
  (a) Per-user: 1 checkout attempt per 10 seconds (Redis: SETEX ratelimit:user:{user_id} 10 '1'), prevents bots spamming checkout, (b) Per-IP: 10 requests/sec (API Gateway rate limit), (c) CAPTCHA: Require CAPTCHA before checkout to filter bots,

- **Caching strategy**:
  (a) Product details: Cached in Redis + CDN, 99% hit rate, only 1% hits DB (1M users × 0.01 = 10K DB queries vs 1M without cache), (b) Inventory check: Redis counter for quick check: DECR flash_sale:{product_id}:remaining, if result >= 0: proceed to checkout (queue-based), if <0: return 'Sold out' immediately, (c) Sync Redis with DB: Periodically (every 5 sec) update Redis counter from DB to correct drift,

- **Payment optimization**:
  (a) Batch payment processing: Instead of 1 payment intent per order, batch 100 orders → 100 payment intents in parallel via Payment Gateway, (b) Async confirmation: User sees 'Order placed, confirming payment...' immediately, payment processes in background, (c) Webhook handling: Scale webhook consumer 10x to handle 1000 payment webhooks/sec,

- **Database write optimization**:
  (a) Batch inserts: Buffer 100 order records, INSERT in single transaction every 2 seconds, reduces DB load 100x, (b) Write to primary: All order writes to primary DB, (c) Read from replicas: User order status queries hit replicas (eventual consistency acceptable, 1-2 sec lag),

- **Post-sale cleanup**:
  (a) Disable rate limiting, scale down instances to normal capacity, (b) Analyze metrics: % users got product, avg checkout time, peak load handled, (c) Notify users on waitlist: 'Sold out in 3 minutes, sign up for restock alert'. Real-world example: Amazon Prime Day: 10M products on sale, 100K orders/minute peak, waiting room gates 5M concurrent users, queue-based checkout processes 50K checkouts/min, 99.9% uptime during sale, inventory sold out products in <10 min. Performance numbers: Without optimizations: 100K concurrent checkout requests → DB meltdown (10K writes/sec = deadlocks), Payment Gateway rate limit exceeded (max 1K/sec), overselling due to race conditions, With optimizations: Waiting room → 1K users/sec enter (controlled), Queue-based checkout → 100 checkouts/sec (sustainable), Redis SKIP LOCKED → no deadlocks, Batch DB writes → 500 writes/sec (comfortable), no overselling (flash_sale_inventory table serializes access).

---

### Q9: How do you implement product recommendations and personalization?

Multi-strategy recommendation engine:

- **Data collection**:
  (a) User events: Kafka topics capture: 'product.viewed': {user_id, product_id, timestamp, session_id}, 'product.added_to_cart': {user_id, product_id, qty}, 'product.purchased': {user_id, product_id, order_id}, 'product.searched': {user_id, query, filters}, (b) Event processing: Apache Flink/Spark Streaming consumes events, aggregates per user: recent_views (last 20 products), recent_searches (last 10 queries), purchase_history (all purchases), (c) Storage: User profile in Cassandra partitioned by user_id: {user_id, recent_views[], recent_searches[], purchase_history[], preferred_categories[], avg_price_range},

- **Recommendation strategies**:
  Strategy 1 - Collaborative filtering: (a) User-based: Find similar users (Jaccard similarity on purchase history), recommend products those users bought: similar_users = users with >50% overlap in purchase_history, recommendations = products purchased by similar_users NOT purchased by current user, (b) Item-based: Find similar products (cosine similarity on co-purchase matrix), recommend: 'Users who bought X also bought Y', co-purchase matrix: If product A and B purchased together 1000 times, similarity(A,B) = 1000 / sqrt(purchases(A) × purchases(B)), (c) Matrix factorization: Offline training (ALS algorithm) on user-product interaction matrix (100M users × 10M products = sparse), factorizes into user_factors (100M × 50) and product_factors (10M × 50), recommendation_score(user, product) = dot(user_factors[user], product_factors[product]), top 100 products per user stored in Redis, refreshed daily, Strategy 2 - Content-based: (a) Product embeddings: Use product metadata (title, description, category, brand) to generate embeddings (BERT/Sentence Transformers), 768-dimensional vector per product, (b) User profile vector: Average of embeddings of products user viewed/purchased, (c) Recommendation: Find products with highest cosine similarity to user profile vector, Elasticsearch kNN search with vector field for fast retrieval, Strategy 3 - Trending & popular: (a) Global trending: Products with highest view_count in last 24h, shown to cold-start users (no history), (b) Category trending: Top products per category, updated hourly, (c) Personalized trending: Trending in user's preferred_categories[], Strategy 4 - Business rules: (a) Margin-aware: Boost products with higher profit margin, (b) Inventory-aware: Boost overstocked products (qty >1000), (c) Promotional: Priority to products on sale or new arrivals,

- **Hybrid ranking**:
  Combine all strategies with weighted scoring: final_score = 0.4 × collaborative_score + 0.3 × content_score + 0.2 × trending_score + 0.1 × business_score, Sort products by final_score, return top 20,

- **Real-time personalization**:
  (a) Home page: GET /v1/recommendations/home → returns: {trending_products: [...], recommended_for_you: [...], recently_viewed: [...], based_on_cart: [...]}, (b) Product detail page: GET /v1/recommendations/product/{product_id} → returns 'Similar products' and 'Frequently bought together', (c) Search results: Personalize ranking based on user history (boost products in preferred_categories),

- **Serving architecture**:
  (a) Pre-compute: Nightly batch job generates top 100 recommendations per user (collaborative filtering + content-based), stores in Redis: SET recs:{user_id} '[{product_id, score}, ...]' EX 86400, (b) Real-time blending: On API call, fetch from Redis (pre-computed), blend with trending and business rules, apply diversity (don't show 10 similar products, mix categories), (c) Fallback: If user has no history (cold start), show global trending + editorial curated,

- **A/B testing**:
  (a) Experiment framework: Randomly assign users to variant A (old algo) or B (new algo), track metrics: click-through rate (CTR), add-to-cart rate, purchase conversion, revenue per user, (b) Winner: If variant B has +5% conversion with p-value <0.05, roll out to 100%,

- **Model training**:
  (a) Offline: Apache Spark job on S3 data lake (1 year of events, 10 TB), trains ALS model (collaborative filtering) weekly, trains BERT embeddings monthly, (b) Feature store: Tecton/Feast stores features (user_avg_cart_value, product_category_popularity), used by ML models, (c) Online: Low-latency feature serving from Redis (user recent_views) or PostgreSQL (user purchase_history),

- **Privacy & ethics**:
  (a) No PII in logs: Hash user_id before storing in analytics DB, (b) Diversity: Don't create filter bubbles, 20% recommendations should be from new categories (exploration vs exploitation), (c) Explainability: Show reason: 'Based on your purchase of Product X' or 'Trending in Electronics',

- **Performance**:
  (a) Recommendation API latency: <100ms p95 (Redis lookup + blending), (b) Pre-computation: 100M users × 100 recs each = 10B recs/day, Spark job takes 4 hours, (c) Storage: Redis 100GB for pre-computed recs + user profiles. Business impact: Recommendations drive 30-40% of e-commerce revenue, avg +25% cart size when users add recommended products, reduces bounce rate (users discover more relevant products).

---

### Q10: What's your disaster recovery and data consistency strategy for the e-commerce platform?

Multi-region DR with tiered consistency (from image shows strong consistency for orders, eventual for search):

- **Architecture**:
  Primary region us-east-1, Secondary us-west-2, Tertiary eu-west-1,

- **Database replication**:
  (a) MySQL (User, Order DB): Primary in us-east-1, synchronous replication to us-west-2 (hot standby, <1s lag), asynchronous to eu-west-1 (warm standby, <10s lag), uses MySQL Group Replication or AWS Aurora Global Database, (b) PostgreSQL (Cart, Inventory): Similar setup, streaming replication with replication slots, (c) MongoDB (Product): Multi-region cluster with replica sets per region, write concern majority ensures data written to >=2 nodes, read preference nearest for low latency,

- **Consistency tiers**:
  Tier 1 - Strong consistency (inventory, orders, payments): (a) Inventory updates: Synchronous replication to secondary before commit, ensures no overselling even during failover, (b) Orders: ACID transactions with 2PC (two-phase commit) if spanning services, (c) Payments: Idempotency + unique constraints prevent double charge, (d) Trade-off: Slower writes (50-100ms extra for replication), acceptable for low-frequency operations, Tier 2 - Eventual consistency (product catalog, search, cache): (a) Product updates: Written to primary MongoDB, replicated asynchronously (1-5s lag), (b) Elasticsearch: CDC pipeline syncs within 1-2 seconds, (c) Redis cache: Invalidated via Kafka events, stale cache for <10s acceptable, (d) Benefit: Fast writes, high availability, user impact minimal (showing product at old price for 5s is fine),

- **Failover procedure**:
  (a) Health checks: Route53 checks primary region every 30s (TCP, HTTP, database connectivity), (b) Trigger: If 3 consecutive failures (90s) → initiate failover, (c) Database promotion: us-west-2 promoted to primary (pg_ctl promote for PostgreSQL, aurora.promoteReadReplicaDBCluster for Aurora), takes 2-5 min, (d) Application routing: Route53 updates DNS to point to us-west-2 load balancer, (e) Kafka: MirrorMaker replicates events to secondary Kafka cluster, consumers switch to secondary, (f) Data loss: <1s for synchronous replicas (Tier 1), <10s for async (Tier 2),

- **Split-brain prevention**:
  (a) Fencing: Primary region fences itself (stops accepting writes) when it detects partition, (b) Witness node: Third region (eu-west-1) acts as tiebreaker for quorum, (c) Distributed lock: Consul/etcd ensures only one region is primary at a time,

- **RTO/RPO targets**:
  (a) Tier 1 (orders, inventory): RPO <1 second (sync replication), RTO <5 minutes (automated failover), (b) Tier 2 (products, search): RPO <10 seconds (async replication), RTO <5 minutes, (c) Backups: RPO <1 hour (hourly backups to S3 Glacier), RTO <4 hours (manual restore from backup),

- **Testing**:
  (a) Monthly DR drills: Simulate primary region failure, execute failover, verify RTO met, test backup restore, (b) Chaos engineering: Randomly terminate instances (during low traffic 2-4 AM), verify auto-recovery, (c) Tabletop exercises: Team walks through disaster scenarios (DDoS, data corruption, AWS outage),

- **Data corruption recovery**:
  (a) Point-in-time recovery: Restore database to timestamp before corruption, supported for last 7 days, (b) Logical backups: Daily mysqldump/pg_dump to S3, retention 30 days, (c) Binary logs: MySQL binlogs for replay, PostgreSQL WAL archives,

- **Kafka durability**:
  (a) Replication factor 3: Events written to >=3 brokers before acknowledged, (b) min.insync.replicas=2: Producer write fails if <2 replicas available, (c) Event retention 7 days: Can replay events to rebuild state,

- **Monitoring & alerts**:
  (a) Replication lag: Alert if lag >5 seconds (Tier 1) or >30 seconds (Tier 2), (b) Database metrics: Connections, query latency, deadlocks, slow queries, (c) Application metrics: Order placement errors, payment failures, checkout abandonment rate, (d) Business metrics: Revenue/hour, conversion rate, average order value (anomaly detection),

- **Consistency validation**:
  (a) Reconciliation job: Nightly job compares Order DB with Payment DB, flags discrepancies (order PAYMENT_CONFIRMED but no payment record), (b) Inventory audit: Compare Inventory DB qty with sum(order_items.qty) for sold products, detect leaks, (c) Financial reconciliation: Match revenue in Order DB with bank settlements,

- **Cost**:
  Multi-region adds 60-80% infrastructure cost vs single region, Trade-off: $1M/month extra cost vs $10M revenue loss for 1-hour outage, insurance for business continuity. Example: Primary region outage at 2:00 PM → Health checks fail at 2:01:30 → Failover initiated at 2:02 → Database promoted at 2:05 → DNS updated at 2:06 → Users routed to secondary at 2:08 → Service fully restored, data loss <1s (in-flight orders may need retry), total downtime <8 minutes, business impact <$50K revenue loss vs $1M for 1-hour outage without DR.

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Catalog Storage** | MongoDB / PostgreSQL JSONB | Rigid Relational SQL Tables | Dynamic JSON attributes support infinite product categories without expensive schema migrations. |
| **Search Engine** | Elasticsearch via Debezium CDC | Direct SQL `LIKE %query%` | Provides sub-50ms full-text keyword search, typo-tolerance, and multi-facet filtering. |
| **Inventory Concurrency** | Atomic SQL Updates with DB `CHECK` Constraints | Distributed Redis Redlock | Prevents overselling at the database engine level; zero split-brain risk. |
| **Distributed Workflow**| Saga Pattern with Kafka Events | Two-Phase Commit (2PC) | Eliminates synchronous cross-service locks; provides resilient compensating rollbacks on payment failure. |
| **Payment Protection** | Redis-backed Idempotency Keys | Client-side button disabling only | Prevents duplicate credit card charges and redundant orders from network retries. |
