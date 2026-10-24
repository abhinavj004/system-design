# System Design: Distributed In-Memory Cache (Redis / Memcached)

> **Source Reference**: [Dodesimo System Design Notes: Distributed Cache](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Distributed%20Cache.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG standard).  
> **Topic**: Designing a High-Throughput, Low-Latency Distributed In-Memory Key-Value Cache (Redis Cluster, Memcached).

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Client routes key via **MurmurHash3 on a Consistent Hashing Ring (256 Virtual Nodes/host)** directly to the primary shard node. The shard node executes $O(1)$ operations via an in-memory **Hash Map + Doubly-Linked List (LRU)**. Replicas sync asynchronously for high availability. TTL expiration combines **passive checks on access** with an **active background sampling thread**. Hot reads are mitigated via **suffixed key copies** (`key_1`..`key_N`), while hot writes use **in-memory write batching** or **sharded counters**.

| Metric / Dimension | Production Target / Value (Dodesimo Scale) |
| :--- | :--- |
| **Read Latency (p99)** | $< 1\text{ ms}$ (in-memory RAM lookup) |
| **Write Latency (p99)** | $< 5\text{ ms}$ |
| **Cluster Throughput** | $100,000\text{ req/sec}$ (peak $500,000 - 1,000,000+\text{ req/sec}$) |
| **Target Storage** | **$1\text{ TB}$ active memory** across cluster |
| **Eviction Policy** | LRU (Least Recently Used) via Hash Map + Doubly-Linked List |
| **Replication Model** | Asynchronous Primary-Replica with automated failover |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **Modulo Hashing vs Consistent Hashing**:
   - *Trap*: *"Why not just do `hash(key) % N` to shard across N servers?"*
   - *Defense*: Modulo hashing changes the denominator for $(N-1)/N \approx 95\%$ of keys whenever a node joins or crashes, wiping out the cache and destroying the database. Consistent hashing ensures only $K/N$ keys relocate.
2. **Handling Hot Key Reads & Writes**:
   - *Trap*: *"What happens when a single celebrity key receives 100,000 reads/sec or a viral post receives 20,000 writes/sec?"*
   - *Defense*: For **Hot Reads**, duplicate the key across nodes using suffixes (`key_1`, `key_2`, ..., `key_10`) and have clients pick a random suffix. For **Hot Writes**, either buffer and batch updates in memory before flushing, or shard the counter into 10 sub-counters (`counter_1`..`counter_10`) and aggregate on read.
3. **TTL Expiration Memory Leaks**:
   - *Trap*: *"If you only check TTL during `GET(key)`, what happens to keys that are never queried again?"*
   - *Defense*: Passive expiration alone creates a massive memory leak. We combine **passive expiration on access** with an **active background cleanup thread** that periodically samples keys (e.g. 20 random keys every 100ms) and evicts expired entries without freezing the CPU.

> **Golden Architectural Rule**: Never rely solely on vertical scaling for hot spots. Split read hotspots using suffixed key copies and write hotspots using counter sharding or batching.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Core Operations**:
   - `PUT / SET(key, value, [ttl])`: Insert or update a key-value pair with an optional Time-To-Live (TTL) in seconds.
   - `GET(key)`: Retrieve the stored value associated with a key. Return `null` if expired or missing.
   - `DELETE / DEL(key)`: Explicitly remove an entry from the cache.
2. **Configurable TTL & Expiration**: Keys should expire automatically after their TTL. Both passive (on access) and active (periodic cleanup) expiration must be supported.
3. **Eviction Policies**: When memory capacity is reached, automatically evict keys based on **LRU (Least Recently Used)** policy.
4. **Data Types**: Support raw byte strings, strings, and serialized JSON objects.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Sub-Millisecond Latency**: 
   - `GET` operations: $< 1\text{ ms}$ (at p99).
   - `SET` operations: $< 5\text{ ms}$.
2. **High Availability**: Node failures must not take down the caching tier; automated failover to replicas.
3. **Scalability**: Support **1 TB of active data** and scale from **100,000 to 1,000,000+ requests per second**.
4. **Consistency Model**: Favors **Availability and Low Latency (AP)** over strict linearizability. Asynchronous primary-replica replication with eventual consistency.
5. **Network Efficiency**: Minimize network hops via client-side consistent hashing and request batching.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput Math
- **Target Peak Throughput**: $100,000\text{ requests/sec}$ (expandable to $1,000,000\text{ req/sec}$).
- **Read-to-Write Ratio**: $90:10$ (Read-heavy):
  $$\text{Read QPS} = 90,000\text{ reads/sec}$$
  $$\text{Write QPS} = 10,000\text{ writes/sec}$$

#### Storage & Node Sizing Math (Dodesimo 1TB Target)
- **Target Active Storage**: $1\text{ TB} = 1,000\text{ GB}$.
- **Average Object Size**: $\approx 1\text{ KB}$ (Key: 64B, Value: 1KB, Metadata & Pointers: 100B).
- **Total Keys Stored**:
  $$N = \frac{1\text{ TB}}{1.16\text{ KB}} \approx 860\text{ Million keys}$$
- **Sizing Cluster Nodes**:
  - Suppose standard cloud cache nodes have **64 GB RAM**.
  - To prevent out-of-memory (OOM) panics and leave room for replication buffers, we cap utilization at **75%** ($\approx 48\text{ GB}$ usable per node).
  - Number of Primary Shards needed for storage:
    $$\text{Primary Nodes} = \frac{1,000\text{ GB}}{48\text{ GB}} \approx 21\text{ Primary Nodes}$$
  - Sizing for throughput: A single Redis core handles $\approx 50,000 - 80,000\text{ QPS}$.
    $$\text{Nodes for Throughput} = \frac{100,000\text{ QPS}}{50,000\text{ QPS/node}} = 2\text{ Nodes}$$
  - Bottleneck rule: $\max(\text{Storage Nodes}, \text{Throughput Nodes}) = \max(21, 2) = \mathbf{21\text{ Primary Nodes}}$.
  - With $1:1$ Primary-Replica pairing for High Availability:
    $$\text{Total Cluster Nodes} = 21\text{ Primaries} + 21\text{ Replicas} = \mathbf{42\text{ Nodes}}$$

---

## 2. Step 2: Identification of Core Entities & In-Memory Data Structures

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                 Single Cache Node                                 │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                        Hash Map (Fast Key Lookup O(1))                      │  │
│  │  [ "user:101" ]  ──▶  Node A  (value, expiry, prev, next)                   │  │
│  │  [ "user:102" ]  ──▶  Node B                                                │  │
│  │  [ "user:103" ]  ──▶  Node C                                                │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                         │                                         │
│                                         ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                     Doubly Linked List (LRU Order O(1))                     │  │
│  │                                                                             │  │
│  │   [ HEAD (Most Recent) ] ◀──▶ [ Node C ] ◀──▶ [ Node A ] ◀──▶ [ TAIL (Old) ] │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────┘
```

### Core Entities & In-Memory Components
1. **Cache Entry**: `key` (String), `value` (Bytes/String), `expiry_timestamp` (Epoch ms or null), `prev` pointer, `next` pointer.
2. **Hash Table (`unordered_map`)**: Maps `key` $\to$ Pointer to DLL Node ($O(1)$ lookup).
3. **Doubly Linked List**:
   - `HEAD` pointer: Most recently accessed items.
   - `TAIL` pointer: Least recently accessed items (first candidate for eviction).
4. **Consistent Hash Ring Node**: `node_id`, `ip_address`, `port`, `virtual_token_hashes`.

---

## 3. Step 3: API & Protocol Design

### 3.1 RESTful HTTP Interface (Edge / Webhooks)
```http
POST /api/v1/cache/{key}
Content-Type: application/json
{
  "value": "{\"user_id\": 101, \"name\": \"Alice\"}",
  "ttl": 3600
}

Response: 200 OK
{ "status": "stored", "key": "user:101" }

GET /api/v1/cache/{key}
Response: 200 OK
{ "value": "{\"user_id\": 101, \"name\": \"Alice\"}" }

DELETE /api/v1/cache/{key}
Response: 204 No Content
```

### 3.2 High-Performance TCP Wire Protocol (RESP / Memcached Binary)
For internal service-to-cache communication, HTTP headers add unacceptable latency ($\approx 1\text{ ms}$). Microservices connect via persistent TCP connections using a lightweight binary/RESP wire format:
```text
SET key value [EX seconds] [NX|XX]
GET key
DEL key
```

---

## 4. Step 4: High-Level Architecture (HLD)

![Distributed Cache Architecture](https://ik.imagekit.io/interviewwithbunny/caching.png?updatedAt=1760735564122)
*Figure 4.1: Distributed Caching Architecture (Source: [Interview with Bunny - System Design Handbook](https://www.interviewwithbunny.com/systemdesignhandbook))*

![Consistent Hashing Ring](https://ik.imagekit.io/interviewwithbunny/consistentHashing.png?updatedAt=1760735475619)
*Figure 4.2: Consistent Hashing Ring Topology (Source: [Interview with Bunny - System Design Handbook](https://www.interviewwithbunny.com/systemdesignhandbook))*

### Component Interaction Topology
```
[ Application Client Pods ] (Web Servers / Microservices)
             │
             │ (1. Client computes hash(key) on Consistent Hashing Ring)
             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Client Routing Tier                                │
│   (Option A: Smart Client with local Consistent Hashing Ring)           │
│   (Option B: Dedicated Proxy Tier like Twemproxy / Envoy)               │
└─────────────────────────────────────────────────────────────────────────┘
          │                                  │                     │
          ▼ (Key: "u_101")                   ▼ (Key: "u_402")      ▼ (Key: "u_999")
┌──────────────────┐               ┌──────────────────┐  ┌──────────────────┐
│  Shard Node 1    │               │   Shard Node 2   │  │   Shard Node 3   │
│  [ Primary ]     │               │   [ Primary ]    │  │   [ Primary ]    │
└──────────────────┘               └──────────────────┘  └──────────────────┘
          │                                  │                     │
          │ Async Replication Log            │                     │
          ▼                                  ▼                     ▼
┌──────────────────┐               ┌──────────────────┐  ┌──────────────────┐
│  Shard Node 1    │               │   Shard Node 2   │  │   Shard Node 3   │
│  [ Replica ]     │               │   [ Replica ]    │  │   [ Replica ]    │
└──────────────────┘               └──────────────────┘  └──────────────────┘
          ▲                                  ▲                     ▲
          └──────────────────────────────────┴─────────────────────┘
                                     │ Heartbeats & Cluster State
                        ┌────────────────────────┐
                        │   Cluster Coordinator  │
                        │ (Gossip / ZooKeeper /  │
                        │    Redis Sentinel)     │
                        └────────────────────────┘
```

---

### 4.2 Step-by-Step Architectural Evolution (Whiteboard Script)

#### Phase 1: Naive Single-Node In-Memory Cache
- **Whiteboard**: `Client` ──► `Single Redis/Memcached Server` (Hash Map + DLL).
- **Explanation**: *"We start with a single in-memory instance providing $O(1)$ read and write operations. The client connects over TCP, and keys are evicted when memory reaches capacity using LRU."*
- **Bottlenecks**:
  1. *RAM Limit*: A single server cannot hold 1TB of data.
  2. *Single Point of Failure (SPOF)*: If this node crashes, 100% of read traffic slams the database.
  3. *Single-Core Bottleneck*: Maxes out at $\approx 80,000\text{ QPS}$.

#### Phase 2: Naive Modulo Sharding
- **Whiteboard**: Add $N$ servers. `server_index = hash(key) % N`.
- **Explanation**: *"To scale memory, we partition keys across $N$ cache servers using a hash of the key."*
- **The Fatal Flaw**:
  - When node $N+1$ is added or a node crashes, the denominator changes for almost all keys. $(N-1)/N \approx 95\%$ of keys suddenly hash to the wrong server, causing a **Massive Cache Stampede** that immediately destroys the database.

#### Phase 3: Consistent Hashing Ring with Virtual Nodes
- **Whiteboard**: Circular keyspace ($0$ to $2^{32}-1$) with **256 Virtual Nodes per physical server**.
- **Explanation**: *"Consistent hashing ensures that when a server is added or removed, only $1/N$ of keys re-locate. Virtual nodes distribute keys evenly, keeping server load variance under 2%."*

#### Phase 4: High Availability via Asynchronous Primary-Replica Pairs
- **Whiteboard**: Add 1 Replica for every Primary shard + Gossip/Sentinel monitoring.
- **Explanation**: *"Writes go to the Primary and replicate asynchronously to the Replica. Sentinels monitor node health via Gossip; if a Primary drops, the Replica is promoted in $< 3\text{ seconds}$."*

#### Phase 5: Client-Side Smart Routing & Connection Pooling
- **Whiteboard**: Embed the Consistent Hash ring into the Client SDK.
- **Explanation**: *"Instead of hitting a centralized proxy which adds a network hop, the Smart Client computes the destination shard locally and reuses a persistent TCP connection pool."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: High Availability & Replication Models

How do we ensure the cache is highly available and fault-tolerant?

#### Approach 1 (Bad): Synchronous Replication
- **Flow**: When a write comes in, the primary forwards it to all replicas and **waits for all replicas to acknowledge** before responding to the client.
- **Why it Fails in Caching**:
  1. *High Write Latency*: The write operation is bounded by the slowest replica node or network jitter.
  2. *Availability Meltdown*: If even a single replica crashes or lags, write operations freeze because the cluster cannot reach full consensus. Caching requires speed over strict consistency.

#### Approach 2 (Good / Standard): Asynchronous Replication
- **Flow**:
  1. Client sends write to Primary.
  2. Primary updates memory and immediately returns success to the client ($< 1\text{ ms}$).
  3. Primary asynchronously propagates changes to replicas via an in-memory replication stream.
- **Trade-offs**:
  - *Pros*: Blazing fast writes, high availability (writes proceed even if replicas are down).
  - *Cons*: Replicas may have slight replication lag (eventual consistency). If the primary crashes before flushing its replication buffer, a promoted replica might miss recent writes. (Acceptable for cache).

#### Approach 3 (Good Alternative): Peer-to-Peer Replication (Gossip Protocol)
- **Flow**: All nodes are peers (no primary). Any node accepts reads and writes, and updates propagate across the cluster using a Gossip protocol (e.g. Amazon Dynamo / Apache Cassandra style).
- **Trade-offs**:
  - *Pros*: No single point of failure; seamless horizontal scaling.
  - *Cons*: High implementation complexity, vector clocks required for conflict resolution, higher network chatter.

---

### 5.2 Deep Dive 2: Single-Node LRU Eviction & Memory Mechanics

How does a single node achieve $O(1)$ get, put, and evict? (LeetCode 146 in Production):

```
       HEAD (Most Recent) ◀───▶ Node C ◀───▶ Node A ◀───▶ TAIL (Oldest)
                                  ▲
                                  │
                          [ Hash Map Index ]
```

1. **`GET(key)`**:
   - Look up key in Hash Map. If missing, return `null`.
   - Detach node from current position in Doubly Linked List and move to `HEAD` ($O(1)$).
   - Return value.
2. **`SET(key, value, ttl)`**:
   - If key exists: Update value and move node to `HEAD`.
   - If key is new:
     - If memory limit is reached: Remove node at `TAIL` (the oldest unaccessed item), delete its key from the Hash Map ($O(1)$).
     - Allocate new node, insert into Hash Map, prepend to `HEAD`.

#### Production Optimization: Redis Approximated LRU
- True LRU requires two 8-byte pointers per key ($16\text{ bytes}$). For 1 Billion keys, pointers alone consume **16 GB of pure overhead**.
- **Redis Approximated LRU**:
  - Instead of linked list pointers, each key stores a 24-bit timestamp of its last access time.
  - When memory is full, the server samples $N$ random keys (e.g., $N=5$ or $10$) and evicts the key with the oldest timestamp.
  - With $N=10$, this achieves $99\%$ similarity to true LRU while saving gigabytes of pointer memory!

---

### 5.3 Deep Dive 3: TTL Expiration Strategy (Passive vs. Active Cleanup)

How do we reclaim memory from expired keys?

#### The Dual Expiration Strategy:
```python
# 1. Passive Expiration (Checked during access)
def get(key):
    if key not in data:
        return None
    val, expiry = data[key]
    if expiry and current_time() > expiry:
        del data[key]       # Key expired, delete lazily
        return None
    return val

def set(key, value, ttl=None):
    expiry = current_time() + ttl if ttl else None
    data[key] = (value, expiry)
```

- **The Problem with Passive Expiration Alone**:
  - Keys that are set with a TTL but **never accessed again** will stay in memory forever, causing a silent, catastrophic memory leak!

```python
# 2. Active Background Cleanup (Periodic Sweep)
def active_cleanup():
    # Naive cleanup: scanning all keys causes O(N) CPU freeze!
    # Production cleanup (Redis algorithm): Sample-based
    sample_size = 20
    keys_to_test = sample_random_keys(sample_size)
    expired_count = 0
    
    for key in keys_to_test:
        val, expiry = data[key]
        if expiry and current_time() > expiry:
            del data[key]
            expired_count += 1
            
    # If more than 25% of sampled keys were expired, repeat immediately!
    if expired_count > (sample_size * 0.25):
        active_cleanup()
```

- **Redis Active Sampling Algorithm**:
  - 10 times per second, Redis tests 20 random keys with an active TTL.
  - Any expired key is deleted.
  - If $> 25\%$ (5 keys) were expired, it repeats immediately.
  - **Result**: Keeps expired keys under $25\%$ of total memory with negligible CPU usage ($< 1\%$).

---

### 5.4 Deep Dive 4: Hot Keys Under Disproportionate Traffic

What happens when a single key receives millions of requests? (e.g., World Cup score, celebrity tweet).

#### 1. Hot Key Reads (High Read Traffic)
- **Problem**: Consistent hashing sends 100% of queries for `key` to a single shard, melting its CPU/network interface.
- **Strategy A: Dedicated Hot Key Tier**:
  - Promote detected hot keys to a separate cluster tier powered by high-spec hardware.
  - *Downside*: Operational complexity and monitoring lag.
- **Strategy B: Read Replicas**:
  - Replicate the shard across 5–10 read replicas.
  - *Downside*: High storage overhead since the entire shard's memory is duplicated.
- **Strategy C (Champion): Suffixed Key Copies**:
  - Detect hot key (e.g. `breaking_news`).
  - Create $K$ suffixed copies across the cluster: `breaking_news_1`, `breaking_news_2`, ..., `breaking_news_10`.
  - Consistent hashing naturally scatters these suffixed keys across 10 completely different physical nodes!
  - **Reading**: Client randomly selects a suffix: `random(1, 10)`. Read traffic is perfectly diluted by $10\times$.
  - **Writing**: Primary updates all $K$ copies asynchronously.

#### 2. Hot Key Writes (High Write Traffic / Viral Counters)
- **Problem**: 20,000 concurrent writes per second hitting the exact same key (e.g. likes counter).
- **Strategy A: Write Batching / Buffering**:
  - Accumulate increments in a local client-side or worker-side buffer for 100ms.
  - Flush the aggregate batch in a single atomic update: `INCRBY key 500`.
  - *Trade-off*: Small delay in counter visibility ($\approx 100\text{ ms}$), but reduces write operations by $99\%$.
- **Strategy B: Sharded Counter (Suffix Sharding)**:
  - Split the counter into 10 separate shards: `counter:post_42:1` through `counter:post_42:10`.
  - On write: Increment a random shard: `INCR counter:post_42:rand(1, 10)`.
  - On read: Query and sum all 10 shards: `total = sum(GET counter:post_42:*)`.
  - *Trade-off*: Maximum write throughput at the cost of slight read aggregation overhead.

---

### 5.5 Deep Dive 5: Network & Client-Side Latency Optimizations

In a distributed cache, network round-trip time (RTT) dominates execution time ($1\text{ ms RTT}$ vs $10\mu\text{s}$ CPU lookup).

1. **Smart Client Routing (Eliminate Proxy Hops)**:
   - Application servers embed the consistent hashing ring directly into the client library.
   - The client hashes the key locally and connects **directly to the target cache node**, avoiding an intermediate proxy hop.
2. **Request Pipelining & Multi-Key Batching**:
   - Instead of 50 round-trips for 50 keys, client sends an `MGET` or pipelines 50 commands in a single TCP socket packet.
3. **Persistent TCP Connection Pooling**:
   - Creating a new TCP connection takes 3 packets (3-way handshake) $\approx 2 - 5\text{ ms}$.
   - Clients maintain a pre-warmed pool of persistent connections to every cache node.

---

### 5.6 Deep Dive 6: Classic Distributed Cache Disasters & Defenses

| Cache Hazard | Problem Description | Production Defense |
| :--- | :--- | :--- |
| **Cache Stampede (Thundering Herd)** | A popular key expires. 50,000 concurrent requests get a cache miss simultaneously and hammer the backing database. | **Single-Flight Distributed Mutex**: The first missing request acquires an atomic lock (`SET lock:key NX EX 5`) to warm the cache from DB. Remaining requests sleep or serve stale data for 50ms. |
| **Cache Penetration** | Attackers query non-existent keys (e.g. `GET /user/-9999`). Misses hit the DB every time. | **Bloom Filter**: In-memory probabilistic bitset sitting in front of cache. If Bloom filter says key does not exist, reject immediately without hitting DB. Or cache `(key, NULL, 60s TTL)`. |
| **Cache Avalanche** | Millions of keys seeded with the same TTL expire at the exact same second. | **TTL Jitter**: Add random variance to the expiration time: `ttl = base_ttl + rand(-10%, +10%)`. Spreads expirations evenly across time. |

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Cluster Sharding** | Consistent Hashing with 256 Virtual Nodes | Modulo Hashing (`hash(k) % N`) | Consistent hashing remaps only $1/N$ keys on node failure, preventing cluster-wide cache invalidation. |
| **Eviction Algorithm** | Approximated LRU (Random Sampling) | Strict Doubly-Linked List LRU | Saves 16 bytes of pointer overhead per key ($\approx 16\text{ GB}$ saved on 1B keys) with 99% accuracy. |
| **Expiration Engine** | Passive on access + Active random sweep | Pure passive or full background scan | Passive alone causes memory leaks; full scans freeze CPU. Active sampling limits expired keys to $<25\%$ memory with $<1\%$ CPU. |
| **Replication Model** | Asynchronous Primary-Replica | Synchronous 2-Phase Commit | Delivers sub-millisecond write latency ($< 1\text{ ms}$) at the cost of transient eventual consistency. |
| **Hot Key Mitigation** | Suffixed Copies (Reads) & Batching (Writes) | Vertical Hardware Scaling | Suffixed copies dilute read hotspots across shards; write batching collapses high-frequency counter updates. |
| **Network Hop Optimization** | Smart Client with Connection Pooling | Centralized Proxy Tier (Twemproxy) | Smart clients eliminate the intermediate proxy hop, shaving $1-2\text{ ms}$ off p99 latency. |

---

### Top 4 Interview Follow-Up Questions (Rapid-Fire Defenses)

### Q1: How do you handle cache invalidation when data in the database changes?
- **Answer**: Use the **Cache-Aside (Lazy Loading)** pattern with **Invalidation on Write**: when the DB updates, explicitly delete the key from the cache (`DEL key`) rather than updating it in-place. This prevents race conditions between concurrent database writes.

### Q2: What happens if a cache node crashes while holding dirty data?
- **Answer**: In a cache tier, data is either transient or backed by a persistent database. If using a write-behind strategy, an unexpected crash risks losing un-flushed writes. To mitigate this, use asynchronous replication to a replica node or enforce write-through caching for mission-critical records.

### Q3: How do you choose the number of virtual nodes on a consistent hash ring?
- **Answer**: Benchmark testing shows that **100 to 256 virtual nodes per physical host** strikes the ideal balance between minimal memory footprint for the hash ring lookup ($O(\log(\text{total\_vnodes}))$ in a binary search tree) and a standard deviation in key distribution under $2\%$.

### Q4: How do you prevent replication lag from serving stale data to a user who just updated their profile?
- **Answer**: Route reads from the user who performed the write directly to the **Primary shard** for a short grace window (e.g. 5 seconds) using a session token, while routing all other users to read replicas.
