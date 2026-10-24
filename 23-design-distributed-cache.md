# System Design: Distributed In-Memory Cache (Redis / Memcached)

> **Source Reference**: [Dodesimo System Design Notes: Distributed Cache](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Distributed%20Cache.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](file:///d:/system%20design%20generation/00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG standard).  
> **Topic**: Designing a High-Throughput, Low-Latency Distributed In-Memory Key-Value Cache (Redis Cluster, Memcached)

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Client routes key via **MurmurHash3 on a Consistent Hashing Ring (256 Virtual Nodes/host)** directly to the primary shard node. Shard node executes $O(1)$ operations via an in-memory **Hash Map + Doubly-Linked List (LRU)**. Replicas sync asynchronously via in-memory replication log; cluster membership and failure detection managed via Gossip protocol.

| Metric / Dimension | Production Target / Value |
| :--- | :--- |
| **Read Latency (p99)** | $< 1	ext{ms}$ (in-memory RAM lookup) |
| **Write Latency (p99)** | $< 5	ext{ms}$ |
| **Cluster Throughput** | $100,000 - 1,000,000+	ext{ QPS}$ (90% read : 10% write) |
| **RAM Footprint** | $1	ext{ TB}$ active memory across 32 nodes ($32	ext{ GB}$ per node) |
| **Eviction Policy** | Approximate LRU / LFU with passive and active TTL sweeps |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **Modulo Hashing vs Consistent Hashing**:
   - *Trap*: *"Why not just do `hash(key) % N` to shard across N servers?"*
   - *Defense*: Modulo hashing changes the denominator for $(N-1)/N pprox 95\%$ of keys whenever a node joins or crashes, wiping out the cache and destroying the database. Consistent hashing ensures only $K/N$ keys relocate.
2. **Thundering Herd / Cache Stampede**:
   - *Trap*: *"What happens when a hot key expires under 100,000 concurrent QPS?"*
   - *Defense*: Enforce a **Distributed Mutex Lock** (`SETNX lock:key {uuid} EX 10`). Only 1 worker queries the database to warm the cache; the remaining 99,999 requests sleep/retry or receive stale data.
3. **Cache Penetration vs Breakdown vs Avalanche**:
   - *Defense*: **Penetration** (querying non-existent keys): Reject via **Bloom Filter** or cache `null` with short TTL. **Breakdown** (1 hot key expires): Distributed mutex lock. **Avalanche** (cluster-wide expiry): Jitter all TTLs by $\pm 10\%$.

> **Golden Architectural Rule**: Always add virtual nodes (100–256 per physical server) to consistent hashing to prevent hotspots, and jitter all TTLs to prevent simultaneous cluster-wide expiration.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Core CRUD Operations**:
   - `PUT / SET(key, value, [ttl])`: Insert or update a key-value pair with an optional Time-To-Live (TTL) in seconds.
   - `GET(key)`: Retrieve the stored value associated with a key. Return `null` if expired or missing.
   - `DELETE / DEL(key)`: Explicitly remove an entry from the cache.
2. **Configurable TTL & Expiration**: Keys should expire automatically after their TTL. Both passive (on access) and active (periodic cleanup) expiration must be supported.
3. **Eviction Policies**: When the cache reaches memory capacity, evict entries based on a configurable policy:
   - **LRU (Least Recently Used)** (Default)
   - **LFU (Least Frequently Used)**
   - **FIFO (First In, First Out)**
4. **Data Types**: Support raw byte strings, strings, and JSON objects (extensible to lists, sets, and hashes).

---

### 1.2 Non-Functional Requirements (NFR)
1. **Sub-Millisecond Latency**: 
   - `GET` operations: $< 1\text{ms}$ (at p99).
   - `SET` operations: $< 5\text{ms}$.
2. **High Availability (99.999%)**: Cache shard failure should not bring down the entire caching tier or wipe out data. Automated failover to replicas.
3. **Scalability**: Must linearly scale horizontally to support **1 TB+ of RAM** and **100,000+ to 1,000,000+ QPS**.
4. **Consistency Model**:
   - Favors **Availability and Low Latency (AP)** over strict linearizable consistency. Asynchronous primary-replica replication with eventual consistency across read replicas is standard.
5. **Resilience to Cache Failures**: Mitigate classic hazards: **Cache Stampede (Thundering Herd)**, **Cache Penetration**, **Cache Breakdown**, and **Cache Avalanche**.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput Estimations
- **Total Throughput**: Peak load of $100,000 \text{ QPS}$ (spiking to $500,000 \text{ QPS}$ across the cluster).
- **Read-to-Write Ratio**: Read-heavy ($90:10$ Read:Write ratio):
  $$\text{Read QPS} = 90,000 \text{ reads/sec}$$
  $$\text{Write QPS} = 10,000 \text{ writes/sec}$$

#### Storage & Shard Estimations
- **Target Memory Capacity**: $1 \text{ TB}$ of active data.
- **Average Key-Value Size**:
  - Key: 64 bytes (average string key)
  - Value: 1 KB (serialized JSON / binary payload)
  - Metadata & Pointers (LRU linked list pointers, hash bucket): $\approx 100 \text{ bytes}$
  - Total per item $\approx 1.16 \text{ KB}$
  - Number of keys in 1 TB:
    $$N = \frac{1 \text{ TB}}{1.16 \text{ KB}} \approx 860 \text{ Million keys}$$
- **Sizing Cluster Nodes**:
  - A single physical cache node (e.g., AWS `r6g.2xlarge`) provides **64 GB RAM**.
  - To prevent OOM and allow memory headroom for replication buffers, we cap utilization at **75%** ($\approx 48 \text{ GB}$ usable per node).
  - Number of Primary Shards:
    $$\text{Primary Nodes} = \frac{1,000 \text{ GB}}{48 \text{ GB}} \approx 21 \text{ Primary Nodes}$$
  - With $1:1$ Primary-Replica pairing for High Availability:
    $$\text{Total Cluster Nodes} = 21 \text{ Primaries} + 21 \text{ Replicas} = 42 \text{ Nodes}$$

#### Network Bandwidth
- Total Ingress / Egress:
  $$\text{Bandwidth} = 100,000 \text{ QPS} \times 1.16 \text{ KB} \approx 116 \text{ MB/s} \approx 928 \text{ Mbps}$$
  Easily sustained across a 25 Gbps cloud network backbone.

---

## 2. Step 2: Identification of Core Entities & In-Memory Data Structures

```
+-----------------------------------------------------------------------------------+
|                                 Single Cache Node                                 |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  |                        Hash Map (Fast Key Lookup O(1))                      |  |
|  |  [ "user:101" ]  ──▶  Node A  (value, expiry, prev, next)                    |  |
|  |  [ "user:102" ]  ──▶  Node B                                                |  |
|  |  [ "user:103" ]  ──▶  Node C                                                |  |
|  +-----------------------------------------------------------------------------+  |
|                                         │                                         |
|                                         ▼                                         |
|  +-----------------------------------------------------------------------------+  |
|  |                     Doubly Linked List (LRU Order O(1))                     |  |
|  |                                                                             |  |
|  |   [ HEAD (Most Recent) ] ◀──▶ [ Node C ] ◀──▶ [ Node A ] ◀──▶ [ TAIL (Old) ] |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

### Core Entities:
1. **Cache Entry**: `key` (String), `value` (Bytes/String), `ttl` (Seconds), `created_at` (Timestamp), `expires_at` (Timestamp), `access_count` (for LFU).
2. **LRU Doubly Linked List Node**: `prev` pointer, `next` pointer, `key_ptr`, `val_ptr`.
3. **Hash Map Index**: Fast $O(1)$ mapping of `Key` $\to$ Pointer to `ListNode`.
4. **Consistent Hash Ring Node**: `node_id`, `ip_address`, `port`, `virtual_token_hashes`.

---

## 3. Step 3: API & Protocol Design

While standard HTTP REST is supported, production caches utilize a **lightweight binary or text TCP protocol** (e.g., Redis RESP or Memcached binary protocol) to eliminate HTTP header overhead.

### 3.1 Core Command Specifications

#### 1. SET (Insert or Update Key)
```text
SET key value [EX seconds] [NX|XX]

Parameters:
- EX seconds: Set expiration time in seconds.
- NX: Only set the key if it does NOT already exist.
- XX: Only set the key if it DOES already exist.

Returns:
- "OK" on success.
- "(nil)" if NX/XX condition was not met.
```

#### 2. GET (Retrieve Key)
```text
GET key

Returns:
- "<value>" if key exists and has not expired.
- "(nil)" if key does not exist or has expired.
```

#### 3. DEL (Delete Key)
```text
DEL key [key2 ...]

Returns:
- Integer representing the number of keys removed.
```

#### 4. EXPIRE (Update TTL)
```text
EXPIRE key seconds

Returns:
- 1 if timeout was set.
- 0 if key does not exist.
```

---

## 4. Step 4: High-Level Architecture

![Distributed Cache Architecture](https://ik.imagekit.io/interviewwithbunny/caching.png?updatedAt=1760735564122)
*Figure 4.1: Distributed Caching Architecture (Source: [Interview with Bunny - System Design Handbook](https://www.interviewwithbunny.com/systemdesignhandbook))*

![Consistent Hashing Ring](https://ik.imagekit.io/interviewwithbunny/consistentHashing.png?updatedAt=1760735475619)
*Figure 4.2: Consistent Hashing Ring Topology (Source: [Interview with Bunny - System Design Handbook](https://www.interviewwithbunny.com/systemdesignhandbook))*

### Component Interaction Topology
```
[ Application Client Pods ] (Web Servers / Microservices)
             │
             │ (1. Hash key to determine destination node)
             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Client Routing / Proxy Tier                        │
│   (Option A: Smart Client with local Consistent Hashing Ring)           │
│   (Option B: Dedicated Proxy like Twemproxy / Envoy)                    │
└─────────────────────────────────────────────────────────────────────────┘
          │                                  │                     │
          ▼ (Key: "u_101")                   ▼ (Key: "u_402")      ▼ (Key: "u_999")
┌──────────────────┐               ┌──────────────────┐  ┌──────────────────┐
│  Shard Node 1    │               │   Shard Node 2   │  │   Shard Node 3   │
│  [ Primary ]     │               │   [ Primary ]    │  │   [ Primary ]    │
└──────────────────┘               └──────────────────┘  └──────────────────┘
          │ (Async Replication)              │                     │
          ▼                                  ▼                     ▼
┌──────────────────┐               ┌──────────────────┐  ┌──────────────────┐
│  Shard Node 1    │               │   Shard Node 2   │  │   Shard Node 3   │
│  [ Replica ]     │               │   [ Replica ]    │  │   [ Replica ]    │
└──────────────────┘               └──────────────────┘  └──────────────────┘
          ▲                                  ▲                     ▲
          └──────────────────────────────────┴─────────────────────┘
                                     │
                        ┌────────────────────────┐
                        │   Cluster Coordinator  │
                        │ (Gossip / ZooKeeper /  │
                        │    Redis Sentinel)     │
                        └────────────────────────┘
```


---

### 4.2 Step-by-Step Architectural Evolution (How to Drive the Interview Whiteboard)

In a FAANG system design interview, **never jump straight to a full multi-node sharded cluster**. Walk the interviewer through the failure modes of simpler designs:

#### Phase 1: Naive Single-Node In-Memory Cache
- **What to draw on the whiteboard**: `Client App` ──► `Single Redis / Memcached Server` (In-memory Hash Table + Doubly Linked List for LRU).
- **How to explain the flow**:
  - *"We start with a single in-memory instance providing $O(1)$ read and write operations. The client connects over TCP, and keys are evicted when memory reaches capacity using LRU."*
- **Bottlenecks identified (Why it breaks)**:
  1. *Memory Ceiling*: A single physical server is bounded by RAM capacity (typically 64GB–256GB). We require 1TB+ of cache.
  2. *Single Point of Failure (SPOF)*: If the single node crashes, 100% of traffic cascades directly into the primary database, causing total system failure.
  3. *Network & CPU Saturation*: A single Redis process is single-threaded for execution, bottlenecking at $pprox 100,000	ext{ QPS}$ due to Linux network socket syscall overhead.

#### Phase 2: Naive Sharding via Modulo Hashing
- **What to add next**: Introduce $N$ cache instances. Shard keys using `server_index = hash(key) % N`.
- **How to explain the transition**:
  - *"To scale memory beyond a single node, we partition keys across $N$ cache servers using a hash of the key."*
- **Bottlenecks identified (The Fatal Flaw)**:
  - When node $N+1$ is added or a node crashes, the denominator changes for almost all keys. $(N-1)/N pprox 95\%$ of keys suddenly hash to the wrong server, causing a **Massive Cache Stampede** that immediately destroys the database.

#### Phase 3: Consistent Hashing Ring with Virtual Nodes
- **What to add next**: Map cache nodes and keys onto a **Consistent Hashing Ring ($0$ to $2^{32}-1$)** with **256 Virtual Nodes per physical server**.
- **How to explain the transition**:
  - *"We replace modulo hashing with consistent hashing. When a server is added or removed, only $1/N$ of keys are re-mapped. To prevent hotspots where one server gets an unfair share of keys, each physical server is assigned 256 virtual positions on the ring, ensuring $< 2\%$ load variance."*

#### Phase 4: Client-Side Smart Routing vs. Dedicated Proxy Tier
- **What to add next**: Compare **Smart Client** (MurmurHash3 ring embedded in application SDK) vs **Proxy Tier** (Twemproxy / Envoy).
- **How to explain the trade-off**:
  - *"For internal ultra-low latency ($< 1	ext{ms}$), we use Smart Clients where the app computes the destination node and opens direct TCP connections, avoiding an intermediate proxy hop. If managing multi-language SDKs is too complex, we fall back to a lightweight Envoy proxy tier."*

#### Phase 5: High Availability & Automated Failover
- **What to add next**: Add **Primary-Replica Shard Pairs** (1 Primary + 2 Replicas per shard) and **Gossip / Redis Sentinel** coordination.
- **How to explain the transition**:
  - *"To ensure 99.999% availability, each shard has read replicas syncing asynchronously. Sentinels monitor heartbeats via Gossip protocol and promote a replica to primary within 3 seconds if the primary drops."*

#### Phase 6: Thundering Herd & Penetration Protections
- **What to add next**: Add **Bloom Filter** in front of cache + **Distributed Mutex Lock** on cache miss.
- **How to explain the transition**:
  - *"To prevent cache penetration from bogus keys, a Bloom filter discards queries for non-existent IDs. For hot key expiration, a distributed lock ensures only 1 worker warms the cache from the DB while others wait, eliminating cache stampede."*

---
---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Sharding: Why Modulo Hashing Fails vs. Consistent Hashing

#### The Failure of Modulo Hashing
The naive way to distribute keys across $N$ servers is:
$$\text{Server Index} = \text{hash}(\text{key}) \pmod N$$
- *Catastrophic Flaw*: If you have 20 servers and add 1 server ($N = 21$) or 1 server crashes ($N = 19$), the denominator changes for **almost all keys**.
- **Over 95% of existing keys suddenly hash to the wrong server!**
- The entire cache cluster experiences a near-100% cache miss rate simultaneously $\implies$ **Cache Stampede destroys the backing database**.

#### The Production Solution: Consistent Hashing with Virtual Nodes
1. **The Hash Ring**:
   - Imagine a circular ring of values from $0$ to $2^{32}-1$.
   - Both cache servers and keys are hashed using a uniform hash function (e.g., MurmurHash3 or MD5) onto this ring.
   - To find the destination server for a key, traverse the ring clockwise until encountering the first server node.
2. **Adding or Removing a Node**:
   - When Server $B$ is added between $A$ and $C$, **only the keys between $A$ and $B$ are re-mapped**. Keys on all other servers remain completely unaffected!
3. **Virtual Nodes (Hotspot & Non-Uniform Distribution Fix)**:
   - If physical servers are mapped to only 1 point on the ring, uneven distribution (hotspots) can overload one node.
   - **Solution**: Each physical server is assigned **100 to 256 Virtual Nodes** on the ring (e.g., `Server1#1`, `Server1#2`, ...).
   - This ensures keys and memory load are distributed with $< 2\%$ statistical variance.

```
                  [ Server 1 (v1) ] (Hash: 10,000)
                        /       \
                       /         \
 [ Server 3 (v2) ]    /           \     [ Server 2 (v1) ]
  (Hash: 90,000)     |  HASH RING  |    (Hash: 35,000)
                      \           /
                       \         /
                  [ Server 1 (v2) ] (Hash: 60,000)
```

---

### 5.2 Single-Node In-Memory Mechanics: LRU Eviction Algorithm
A single node must achieve $O(1)$ operations for `get`, `put`, and `evict`.

#### The Classical LRU Implementation:
- **Data Structures**:
  1. **Hash Map (`unordered_map` / dictionary)**: Maps `Key` $\to$ Pointer to Node in Doubly Linked List.
  2. **Doubly Linked List (`head`, `tail`)**:
     - `HEAD` = Most Recently Used (MRU).
     - `TAIL` = Least Recently Used (LRU).
- **Operations**:
  - `GET(key)`:
    1. Look up key in Hash Map. If missing, return `null`.
    2. Check if expired (`now > expires_at`). If expired, delete and return `null`.
    3. Detach node from its current linked list position and splice it directly after `HEAD` ($O(1)$).
    4. Return value.
  - `SET(key, value, ttl)`:
    1. If key exists: Update value and move node to `HEAD`.
    2. If key is new:
       - If memory is full: Remove node at `TAIL->prev`, erase its key from the Hash Map ($O(1)$).
       - Create new node, insert into Hash Map, prepend to `HEAD`.

#### Production Optimization: Approximated LRU (Redis Model)
- Classical LRU requires 2 pointers (16 bytes) per entry in the linked list. For 1 Billion keys, pointers alone consume **16 GB of pure overhead**.
- **Redis Approximated LRU**:
  - Instead of linked list pointers, each key stores a 24-bit timestamp of its last access time.
  - When memory limit is reached, the server **samples $N$ random keys** (e.g., $N=5$ or $10$).
  - It evicts the oldest key among the sampled pool.
  - Mathematically, with $N=10$, this achieves $99\%$ similarity to true LRU while saving gigabytes of pointer overhead!

---

### 5.3 TTL Expiration Strategy: Active vs. Passive Cleanup

How does the cache reclaim memory from expired keys that are never accessed again?

```
┌─────────────────────────────────────────────────────────────┐
│                 Dual Expiration Architecture                │
└─────────────────────────────────────────────────────────────┘
          │                                           │
          ▼                                           ▼
┌──────────────────┐                         ┌──────────────────┐
│Passive Expiration│                         │Active Expiration │
│ (Lazy on Access) │                         │ (Periodic Sweep) │
└──────────────────┘                         └──────────────────┘
  - Checks timestamp during `GET`             - Runs every 100ms in background
  - If expired: Deletes key & returns nil     - Samples 20 keys with TTL
  - Zero background CPU overhead              - If >25% expired, repeats sweep
```

1. **Passive (Lazy) Expiration**:
   - Every `GET(key)` inspects `expires_at`.
   - If `current_time > expires_at`, the entry is deleted immediately on the spot, returning `nil`.
2. **Active (Periodic Statistical) Expiration**:
   - If keys are written and never queried, lazy expiration leaves orphaned memory.
   - **The Algorithm**:
     1. A background loop runs 10 times per second (every 100ms).
     2. Samples **20 random keys** with an active TTL.
     3. Deletes all expired keys found in the sample.
     4. If **more than 25% (5 keys)** were expired, it repeats step 2 immediately without waiting for the next 100ms tick.
   - Keeps memory leaks under $25\%$ of total expired keys with negligible CPU usage ($< 1\%$).

---

### 5.4 Replication, Failover & High Availability

To ensure the cluster does not lose cached data when a node dies:
1. **Primary-Replica Asynchronous Replication**:
   - Each shard consists of 1 Primary and 1+ Replicas.
   - All writes go to the Primary. The Primary acknowledges the client immediately ($< 1\text{ms}$) and streams the operation asynchronously to Replicas.
2. **Failure Detection & Automatic Failover**:
   - **Gossip Protocol / Sentinel**: Cache nodes exchange heartbeat pings every second.
   - If a Primary fails to reply for 5 consecutive seconds, it is marked `PFAIL` (Possible Fail).
   - If a quorum of nodes agree, it transitions to `FAIL`.
   - The healthy Replica is elected and promoted to Primary.
   - The client routing topology updates within $< 2\text{ seconds}$.

---

### 5.5 Mitigating Classic Distributed Cache Disasters

| Cache Hazard | Problem Description | Production Mitigation Strategy |
| :--- | :--- | :--- |
| **Cache Stampede (Thundering Herd)** | A popular key expires (e.g., top news story). 50,000 concurrent requests get a cache miss simultaneously and hammer the backing database. | **Distributed Mutex (Single-Flight)**: First thread to miss acquires an atomic lock (`SET lock:key NX EX 5`) to query the DB and repopulate cache. All other 49,999 threads wait or serve stale cache for 50ms. |
| **Cache Penetration** | Malicious users query non-existent keys (e.g., random UUIDs `GET /user/9999999`). Cache misses every time, causing 100% of traffic to slam the database. | **Bloom Filter**: In-memory probabilistic filter sitting in front of cache. If Bloom filter says "Key does NOT exist", reject immediately without querying cache or DB. Alternatively, cache `(key, NULL, 60s TTL)`. |
| **Cache Breakdown** | A single extremely hot key (e.g., world cup final score) expires suddenly. | **Probabilistic Early Expiration (XFetch)**: The application starts refreshing the key in the background a few seconds before actual expiration based on probability $\Delta \times \beta \times \ln(\text{rand}())$. |
| **Cache Avalanche** | Millions of keys are seeded with the exact same TTL (e.g., 24 hours). At hour 24, all keys expire at the exact same second, wiping out the cache tier. | **TTL Jitter**: Add random variance to the expiration time: `ttl = base_ttl + rand(-300s, +300s)`. Spreads eviction across a smooth distribution curve. |

---

### 5.6 Write Policies & Invalidation Patterns

```
1. Cache-Aside (Lazy Loading) [Most Common]
   - Read: App checks Cache. On miss, reads DB, writes to Cache.
   - Write: App updates DB, then DELETES key from Cache (never update in-place to avoid race conditions).

2. Write-Through
   - App writes to Cache. Cache synchronously writes to DB before returning success.
   - High data consistency, but adds write latency.

3. Write-Back (Write-Behind)
   - App writes to Cache. Cache acknowledges immediately and queues asynchronous batch write to DB.
   - Blazing fast write speed, but risk of data loss if cache node crashes before flushing to disk.
```

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Cluster Sharding** | Consistent Hashing with Virtual Nodes | Modulo Hashing (`hash(k) % N`) | Adding/removing nodes remaps only $1/N$ keys instead of invalidating the entire cluster. |
| **Eviction Algorithm** | Approximated LRU (Random Sampling) | Classical Strict Linked List LRU | Saves 16 bytes of pointer overhead per key ($\approx 16\text{ GB}$ saved on 1B keys) with 99% accuracy. |
| **Expiration Engine** | Hybrid: Passive on access + Active statistical sweep | Pure Background Thread Scan | Avoids O(N) full-table scans; keeps expired memory footprint $< 25\%$ without CPU spikes. |
| **Replication Model** | Asynchronous Primary-Replica | Synchronous 2-Phase Commit | Delivers sub-millisecond write latency ($< 1\text{ms}$) at the cost of transient eventual consistency. |
| **Stampede Prevention**| Single-Flight Distributed Mutex | Unthrottled DB Miss Queries | Collapses 50,000 concurrent cache misses down to 1 single DB query. |
| **Invalidation Policy**| Cache-Aside with Invalidation on Write | In-place Cache Update on Write | Prevents stale race conditions between concurrent database writes. |

