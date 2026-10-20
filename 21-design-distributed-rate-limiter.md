# System Design: Distributed Rate Limiter (Token Bucket / Sliding Window)

> **Source**: [Interview with Bunny - Episode 21](https://youtu.be/SfV42UuZEBY)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/21](https://www.interviewwithbunny.com/systemdesign/21)  
> **Topic**: Designing a High-Throughput Distributed Rate Limiting System

---

## 1. Step 1: Requirements Gathering & Estimations

A Rate Limiter acts as a defensive perimeter for distributed systems. It throttles incoming traffic to protect downstream microservices and databases from cascading overload, noisy neighbor starvation, denial-of-service (DoS/DDoS) attacks, and API quota abuse, while enforcing subscription monetization tiers (e.g., Free vs. Paid tiers for ChatGPT/Gemini APIs).

### 1.1 Functional Requirements (FRs)

1. **Multi-Dimensional Rate Limiting**:
   - Ability to limit requests based on diverse subject keys: `IP Address`, `User ID`, `API Key`, `Endpoint Path`, or `Tenant ID`.
2. **Dynamic Runtime Configuration**:
   - Rate limit policies (thresholds, window intervals, algorithm selection) must be configurable at runtime via admin APIs without requiring application restarts or gateway redeployments.
3. **Graceful Throttling & Informative Error Signaling**:
   - Requests exceeding quotas must be promptly dropped with HTTP status `429 Too Many Requests`.
   - The response must return standardized rate limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`).
4. **Burst Traffic Handling**:
   - The system must accommodate natural bursts in user activity without rejecting legitimate temporary traffic spikes.
5. **Multi-Tier Subscription Support**:
   - Differentiate limits by user tier (e.g., Free: 10 requests/min, Gold: 60 requests/min, Enterprise: 500 requests/min).
6. **Code-Level Extensibility (LLD Requirement)**:
   - Provide an extensible Object-Oriented design (Strategy + Factory patterns) enabling interchangeable rate-limiting algorithms.

---

### 1.2 Non-Functional Requirements (NFRs)

1. **Ultra-Low Latency Budget**:
   - Because the rate limiter is positioned inline on the critical request path for *every* inbound call, its decision latency must be **$< 5\text{ ms}$** (target: $\le 1\text{ to }2\text{ ms}$).
2. **High Availability over Consistency (CAP Theorem)**:
   - **AP Prioritization**: The rate limiter must remain 99.999% available. If rate limiter storage fails or partitions, the system must **fail-open** rather than block legitimate user traffic. Nominal rule propagation delay (a few seconds from Policy DB to cache) is completely acceptable.
3. **Massive Scale Throughput**:
   - Must sustain **$1\text{ Million requests per second (RPS)}$** peak throughput globally.
4. **Fault Tolerance & Memory Efficiency**:
   - Rate-limiting states and counters must have minimal memory overhead and auto-expire via TTLs to prevent memory leaks.

---

### 1.3 Capacity Estimations & Back-of-the-Envelope Math

#### Scale Assumptions:
- **Peak Inbound Throughput**: $1,000,000\text{ requests per second (RPS)}$
- **Active Tracked Subjects (Users/IPs)**: $50\text{ Million}$ active keys per 24-hour cycle.

#### Memory Math:
- Data per rate-limiting counter in Redis:
  - Key format: `rl:{subject_type}:{subject_id}:{endpoint}` $\approx 32\text{ bytes}$
  - Value: Hash storing `tokens` (int) and `last_updated_at` (epoch long) $\approx 16\text{ bytes}$
  - Redis overhead per key: $\approx 48\text{ bytes}$
  - Total per entry: $\approx 96\text{ bytes}$
- Total Memory for $50\text{ Million}$ active keys:
  $$\text{RAM Required} = 50,000,000 \times 96\text{ bytes} \approx 4.8\text{ Gigabytes (GB)}$$
- *Conclusion*: A modest 3-node Redis Cluster easily holds the entire state in RAM with immense headroom.

#### Throughput & IOPS Math:
- $1,000,000\text{ RPS}$ hitting Redis cannot be served by a single Redis node (single-threaded Redis core maxes out at $\approx 100,000\text{ to }150,000\text{ ops/sec}$).
- **Clustering Strategy**:
  - Deploy a **Redis Cluster with at least 16 master shards** ($16 \times 80,000\text{ ops/sec} = 1.28\text{M IOPS}$) distributed across read-replicas.
  - Consistent hashing on the Redis Key distributes the load uniformly across cluster slots.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+       +--------------------+       +--------------------+
|  RateLimitRule     |       |    SubjectClient   |       |    BucketState     |
+--------------------+       +--------------------+       +--------------------+
| rule_id (UUID)     | 1   * | client_id (VARCHAR)| 1   1 | redis_key (TEXT-PK)|
| subject_type       |<----->| subject_type       |<----->| tokens (INT/FLOAT) |
| subject_value      |       | tier (FREE/GOLD)   |       | last_refill_ts(MS) |
| endpoint (TEXT)    |       | api_key (VARCHAR)  |       | ttl_seconds (INT)  |
| algorithm (ENUM)   |       +--------------------+       +--------------------+
| limit_capacity(INT)|
| refill_rate_per_sec|
| window_size_sec    |
+--------------------+
```

### Entity Descriptions:
1. **RateLimitRule**: Configurable policy entity defined in persistent storage (`PostgreSQL`). Specifies the target subject, endpoint route, algorithm type (`TOKEN_BUCKET`, `SLIDING_WINDOW_COUNTER`, `LEAKY_BUCKET`, `FIXED_WINDOW`), threshold limits, and window sizes.
2. **SubjectClient**: Client identity representation derived from authentication headers (API key, Bearer JWT user ID, or client IP).
3. **BucketState / CounterState**: In-memory ephemeral entity stored in **Redis**. Tracks remaining tokens or timestamps for an active client key.

---

## 3. Step 3: API Design & HTTP Signatures

### 3.1 Admin Policy Management APIs
```http
POST /api/v1/admin/rules
Authorization: Bearer <admin_token>
Content-Type: application/json

Request Body:
{
  "subject_type": "USER_TIER",
  "subject_value": "FREE_TIER",
  "endpoint": "/api/v1/generate-image",
  "algorithm": "TOKEN_BUCKET",
  "limit_capacity": 10,
  "refill_rate_per_sec": 0.1667, // 10 tokens per minute
  "window_size_sec": 60
}

Response: 201 Created
{
  "rule_id": "rule_9918-a",
  "status": "ACTIVE",
  "created_at": "2026-10-18T10:00:00Z"
}
```

---

### 3.2 Public Request Response Headers (When Allowed)
When a client request succeeds, the gateway enriches downstream response headers:
```http
HTTP/1.1 200 OK
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 84
X-RateLimit-Reset: 1729254060
```

---

### 3.3 Public Throttling Response (When Rate-Limited)
When a client breaches their assigned rate limit:
```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 36
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1729254036

{
  "error": "Too Many Requests",
  "message": "Quota exceeded for endpoint '/api/v1/generate-image'. Please retry after 36 seconds.",
  "retry_after_seconds": 36
}
```

---

## 4. Step 4: High-Level Architecture

Bunny addresses a fundamental architectural question: **Where should the Rate Limiter be placed?**

```
Client Placements Analyzed:
1. Client-Side ──────────► Vulnerable to tampering, bypassed by curl/bots. (Rejected)
2. Application Pods ─────► Local memory counters mismatch across replicated pods. (Rejected)
3. API Gateway Tier ─────► Optimal enforcement point; intercepts traffic before reaching backends.
```

```
[ Client (Web / Mobile / Third-Party SDK) ]
                     │ (Inbound HTTP/gRPC Requests)
                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           API Gateway & Load Balancer                           │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                     Rate Limiter Interceptor / Plugin                     │  │
│  │                                                                           │  │
│  │  1. Extract Subject Key (IP / User ID / API Key)                          │  │
│  │  2. Fetch Policy Config (Local Gateway In-Memory LRU Cache)               │  │
│  │  3. Execute Atomic Rate Limit Evaluation (Redis Cluster via Lua Script)    │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
         │                                                        │
         │ (Allowed: Forward Request)                             │ (Quota Exceeded)
         ▼                                                        ▼
┌───────────────────────────────┐                       ┌───────────────────┐
│     Backend Microservices     │                       │  HTTP 429 Error   │
│ (Image Gen / LLM / Order Svc) │                       │ (Retry-After: 36) │
└───────────────────────────────┘                       └───────────────────┘
         │                                                        │
         │                                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               State & Policy Plane                              │
│                                                                                 │
│   ┌───────────────────────────┐                 ┌───────────────────────────┐   │
│   │   Rate Limiter Policy Svc │                 │       Redis Cluster       │   │
│   │   (CRUD Rules & Limits)   │                 │ (In-Memory Token Buckets) │   │
│   └─────────────┬─────────────┘                 └─────────────▲─────────────┘   │
│                 │                                             │                 │
│                 ▼                                             │ (Write-Through) │
│   ┌───────────────────────────┐                               │                 │
│   │   Policy DB (PostgreSQL)  │───────────────────────────────┘                 │
│   └───────────────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### End-to-End Execution Flow:
1. **Rule Configuration**: Admin creates a rate-limit policy via `Rate Limiter Policy Service`. It is committed to PostgreSQL and written through to the Redis Cluster.
2. **Gateway Interception**: Inbound client request arrives at the API Gateway.
3. **Identity Resolution**: Gateway extracts client credentials (API Key or JWT `user_id`).
4. **Token Evaluation**: The gateway executes an atomic Redis Lua script matching the client's configured algorithm.
5. **Enforcement**:
   - If tokens are available: Redis decrements counter and returns `allow = true`. The Gateway forwards the request to downstream microservices with rate-limit headers.
   - If tokens are depleted: Redis returns `allow = false` and seconds until reset. The Gateway immediately aborts the call, responding with HTTP `429 Too Many Requests`.

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Comparative Analysis of All 5 Rate-Limiting Algorithms

Bunny provides a comprehensive breakdown of the 5 industry algorithms:

```
+---------------------------------------------------------------------------------------------------+
| 1. Fixed Window Counter       │ 2. Sliding Window Log         │ 3. Sliding Window Counter         |
| Window: [0s - 10s]            │ Logs each timestamp in a ZSET │ Weighted calculation between      |
| Counter resets at boundary.   │ Memory: O(N) per request!     │ previous & current window.        |
| Flaw: 2x burst at boundary!   │ Flaw: High RAM consumption.   │ Memory: O(1). Smooth & efficient. |
+-------------------------------+-------------------------------+-----------------------------------+
| 4. Token Bucket (Chosen)      │ 5. Leaky Bucket                                                   |
| Capacity B, refills at rate R.│ FIFO queue with constant outflow drain rate.                      |
| Burst friendly! O(1) memory.  │ Flaw: Queues requests; introduces latency for burst traffic.      |
+---------------------------------------------------------------------------------------------------+
```

#### Detailed Breakdown:

1. **Fixed Window Counter**:
   - Divides timeline into fixed intervals (e.g., 00:00–01:00).
   - *Critical Flaw*: **Boundary Burst Vulnerability**. If a user sends 100 requests at 00:59 and 100 requests at 01:01, they successfully executed 200 requests within a 2-second interval, violating the 100 requests/minute rule by $2\times$!
2. **Sliding Window Log**:
   - Appends timestamps of every request into a sorted set (Redis ZSET). When a new request arrives, elements older than $(now - window)$ are evicted with `ZREMRANGEBYSCORE`.
   - *Critical Flaw*: **Memory Explosion**. For 1,000 requests/sec, the ZSET stores 1,000 timestamp elements per user. Saturated RAM makes it unviable for large-scale production.
3. **Sliding Window Counter (Approximation)**:
   - Blends the previous window's counter with the current window:
     $$\text{Estimated Requests} = (\text{PrevWindowCount} \times (1 - \text{OverlapRatio})) + \text{CurrentWindowCount}$$
   - *Advantage*: Smooths out boundary bursts with negligible $O(1)$ memory. Approximation error is under $0.05\%$.
4. **Token Bucket (Industry Standard - AWS, Stripe, Cloudflare)**:
   - A bucket has fixed capacity $C$ and refills at a steady rate $R$ tokens/second.
   - Incoming request consumes 1 token. If tokens $> 0$, request passes; else rejected.
   - *Advantage*: **Burst-Friendly**. A user who has been idle can execute an immediate burst up to capacity $C$ without delay.
5. **Leaky Bucket**:
   - Requests enter a FIFO buffer of fixed capacity and drain at a strictly constant rate.
   - *Trade-off*: Good for smoothing traffic into legacy systems, but adds queuing latency to client requests during bursts.

---

### 5.2 Deep Dive 2: Atomic Concurrency & The Distributed Race Condition Trap

Bunny exposes a classic distributed concurrency interview trap:
- **The Naive Read-Modify-Write Flaw**:
  ```python
  # NAIVE NON-ATOMIC IMPLEMENTATION (CRITICAL BUG)
  current_tokens = redis.get("rl:user_101")
  if current_tokens > 0:
      redis.set("rl:user_101", current_tokens - 1)
      return ALLOW
  return DROP
  ```
- *Why it fails*: If two concurrent requests arrive simultaneously at different API Gateway pods, both read `current_tokens = 1`. Both decrement to 0 and both allow the request through. Quotas are violated under high concurrency.
- **The Optimal Solution: Atomic Redis Lua Scripting**:
  - Redis executes Lua scripts as a single atomic transaction. No other client command can execute midway.

#### Production-Grade Token Bucket Redis Lua Script:
```lua
-- KEYS[1]: Rate limit key (e.g., "rl:user:usr_101:gen_image")
-- ARGV[1]: Bucket capacity (e.g., 10)
-- ARGV[2]: Refill rate per second (e.g., 2)
-- ARGV[3]: Current timestamp in seconds (e.g., 1729254000)
-- ARGV[4]: Requested tokens (e.g., 1)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- Retrieve current bucket state
local data = redis.call("HMGET", key, "tokens", "last_updated")
local tokens = tonumber(data[1])
local last_updated = tonumber(data[2])

if tokens == nil then
  -- First time initialization
  tokens = capacity
  last_updated = now
else
  -- Calculate tokens replenished since last update
  local delta = math.max(0, now - last_updated)
  local replenished = delta * refill_rate
  tokens = math.min(capacity, tokens + replenished)
  last_updated = now
end

-- Check if sufficient tokens exist
if tokens >= requested then
  tokens = tokens - requested
  redis.call("HMSET", key, "tokens", tokens, "last_updated", last_updated)
  -- TTL set to time required to completely refill the bucket
  local ttl = math.ceil(capacity / refill_rate) * 2
  redis.call("EXPIRE", key, ttl)
  return {1, math.floor(tokens)} -- Allowed, remaining tokens
else
  -- Denied: Return retry-after estimate in seconds
  local missing_tokens = requested - tokens
  local retry_after = math.ceil(missing_tokens / refill_rate)
  return {0, retry_after} -- Denied, retry_after
end
```

---

### 5.3 Deep Dive 3: Object-Oriented Low-Level Design (Strategy & Factory Pattern)

When an interviewer requests executable code for the rate limiter, Bunny implements the **Strategy Pattern**:

```
           ┌───────────────────────────────────┐
           │     <<interface>>                 │
           │     RateLimiterStrategy           │
           ├───────────────────────────────────┤
           │ + isAllowed(clientId, tokens):bool│
           └─────────────────▲─────────────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  TokenBucket  │     │ SlidingWindow │     │  LeakyBucket  │
│  RateLimiter  │     │  RateLimiter  │     │  RateLimiter  │
└───────────────┘     └───────────────┘     └───────────────┘
```

#### Production Java LLD Implementation:

```java
// 1. Strategy Interface
public interface RateLimiterStrategy {
    RateLimitResult isAllowed(String clientId, int tokensRequested);
}

// 2. Result Data Holder
public class RateLimitResult {
    private final boolean allowed;
    private final int remainingTokens;
    private final long retryAfterSeconds;

    public RateLimitResult(boolean allowed, int remainingTokens, long retryAfterSeconds) {
        this.allowed = allowed;
        this.remainingTokens = remainingTokens;
        this.retryAfterSeconds = retryAfterSeconds;
    }
    public boolean isAllowed() { return allowed; }
    public int getRemainingTokens() { return remainingTokens; }
    public long getRetryAfterSeconds() { return retryAfterSeconds; }
}

// 3. Token Bucket Concrete Strategy (Redis-Backed)
public class TokenBucketRateLimiter implements RateLimiterStrategy {
    private final JedisCluster jedisCluster;
    private final int capacity;
    private final double refillRatePerSecond;
    private final String luaSha;

    public TokenBucketRateLimiter(JedisCluster jedisCluster, int capacity, double refillRatePerSecond, String luaSha) {
        this.jedisCluster = jedisCluster;
        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        this.luaSha = luaSha;
    }

    @Override
    public RateLimitResult isAllowed(String clientId, int tokensRequested) {
        String key = "rl:tb:" + clientId;
        long now = System.currentTimeMillis() / 1000L;

        List<String> keys = Collections.singletonList(key);
        List<String> args = Arrays.asList(
            String.valueOf(capacity),
            String.valueOf(refillRatePerSecond),
            String.valueOf(now),
            String.valueOf(tokensRequested)
        );

        List<Long> response = (List<Long>) jedisCluster.evalsha(luaSha, keys, args);
        boolean allowed = response.get(0) == 1L;
        long metric = response.get(1);

        if (allowed) {
            return new RateLimitResult(true, (int) metric, 0);
        } else {
            return new RateLimitResult(false, 0, metric);
        }
    }
}

// 4. Rate Limiter Factory
public class RateLimiterFactory {
    public static RateLimiterStrategy createLimiter(RuleConfig config, JedisCluster redis, String sha) {
        switch (config.getAlgorithm()) {
            case TOKEN_BUCKET:
                return new TokenBucketRateLimiter(redis, config.getCapacity(), config.getRefillRate(), sha);
            case SLIDING_WINDOW_COUNTER:
                return new SlidingWindowCounterRateLimiter(redis, config.getWindowSeconds(), config.getCapacity());
            default:
                throw new IllegalArgumentException("Unsupported algorithm: " + config.getAlgorithm());
        }
    }
}

// 5. API Gateway Enforcement Interceptor
public class ApiGatewayRateLimitFilter {
    private final RateLimiterFactory factory;
    private final PolicyRepository policyRepo;

    public boolean filterRequest(RequestContext context) {
        String clientId = context.extractClientId(); // API key or JWT user ID
        RuleConfig rule = policyRepo.getRuleForClient(clientId, context.getPath());

        RateLimiterStrategy limiter = factory.getLimiter(rule);
        RateLimitResult result = limiter.isAllowed(clientId, 1);

        if (!result.isAllowed()) {
            context.setResponseStatus(429);
            context.addHeader("Retry-After", String.valueOf(result.getRetryAfterSeconds()));
            context.addHeader("X-RateLimit-Remaining", "0");
            context.setResponseBody("{\"error\": \"Too Many Requests\"}");
            return false; // Halt request pipeline
        }

        context.addHeader("X-RateLimit-Remaining", String.valueOf(result.getRemainingTokens()));
        return true; // Forward to downstream microservice
    }
}
```

---

### 5.4 Deep Dive 4: Fail-Open vs. Fail-Closed & System Resilience

Bunny emphasizes a critical reliability decision: *What happens when Redis Cluster experiences a network partition, cluster failover, or latency spike?*

| Resilience Mode | Behavior on Redis Failure | Trade-Off & Evaluation |
| :--- | :--- | :--- |
| **Fail-Closed** | Block all requests returning HTTP 500/429. | Protects backend services at the cost of total system downtime. Destroys customer trust. |
| **Fail-Open (Recommended)** | Bypass rate limiter; let requests pass to backend. | Backends take more load, but legitimate users experience zero disruption during temporary cache blips. |

#### Production Implementation: Circuit Breaker with Local Fallback:
- Wrap Redis calls in a **Circuit Breaker** (e.g., Netflix Hystrix / Resilience4j):
  - If Redis call exceeds $5\text{ ms}$ or error rate $> 10\%$, trip the breaker.
  - Fall back to an in-memory **Local Caffeine Cache** on the API Gateway pod with temporary conservative limits, or log an alert and let traffic pass through (**Fail-Open**).

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: Compare the 5 rate limiting algorithms (Fixed Window, Sliding Window Log, Sliding Window Counter, Token Bucket, Leaky Bucket). Which would you choose for a production API and why?

Comparison of 5 rate limiting algorithms:

- **Fixed Window Counter**:
  Pros: Simple (single counter per window), low memory (one int per user), fast (single INCR), Cons: Boundary problem (spike at edges: 100 req at 10:00:59 + 100 req at 10:01:00 = 200 req in 2 sec), Implementation: Redis INCR + EXPIRE, Use case: Simple internal APIs, not user-facing.

- **Sliding Window Log**:
  Pros: Highly accurate (true rolling window, no boundary issue), perfect precision, Cons: Expensive memory (stores every request timestamp: 100 req/min × 1M users = 100M timestamps), cleanup overhead (ZREMRANGEBYSCORE on each request), Implementation: Redis ZSET (ZADD, ZCOUNT, ZREMRANGEBYSCORE), Use case: Low-traffic critical APIs where accuracy crucial (fraud detection).

- **Sliding Window Counter**:
  Pros: More accurate than fixed window (smooths boundaries), lower memory than log (2 counters vs all timestamps), Cons: Approximation (not 100% accurate), complexity (weighted calculation), Implementation: Two counters (current + previous window), weighted sum, Use case: Good middle ground, not widely adopted (token bucket preferred).

- **Token Bucket (RECOMMENDED)**:
  Pros: Supports bursts naturally (capacity = burst size, e.g., 100 token bucket allows 100 instant requests then refills), smooth refill (10 tokens/sec = steady 600/min), production-proven (AWS API Gateway, Stripe, GitHub), handles spiky traffic well, Cons: Slightly complex (track tokens + last_refill), Implementation: Redis HASH (tokens, last_refill) + Lua script (atomic check + refill calculation), Use case: Production APIs (most common choice).

- **Leaky Bucket**:
  Pros: Smooth output traffic (fixed processing rate), good for traffic shaping (prevent downstream overload), Cons: Can delay/drop requests (queuing), requires background processor (leak at fixed rate), not typical for sync APIs, Implementation: Redis LIST (queue) + background worker, Use case: Traffic shaping, not typical API rate limiting. Production choice: Token Bucket. Reasoning:

- **Supports bursts**:
  Users expect to make 100 requests instantly if quota available (e.g., batch operations), token bucket allows this, fixed/sliding window would reject after hitting limit in burst,

- **Smooth refill**:
  No sudden resets at window boundaries, users can make requests continuously at refill rate (10/sec),

- **Industry standard**:
  AWS, Stripe, GitHub all use token bucket, proven at scale (billions of requests/day),

- **Good UX**:
  Burst + refill feels natural to users,

- **Reasonable complexity**:
  Redis Lua script handles atomicity, not much more complex than fixed window. Implementation example (Token Bucket): Capacity: 100 tokens, Refill: 10 tokens/sec, Scenario: User makes 100 requests in 1 sec (burst) → all allowed (consumes all 100 tokens), After 5 sec: bucket refilled to 50 tokens (10 tokens/sec × 5 sec), User makes 50 requests → all allowed, User makes 51st request → rejected (bucket empty), After 1 sec: 10 tokens refilled, user can make 10 more requests. Result: Flexible (supports bursts), fair (smooth refill), production-ready (proven algorithm).

---

### Q2: How do you implement distributed rate limiting across multiple API Gateway instances using Redis? Handle race conditions and ensure consistency.

Distributed rate limiting with Redis:

- **Challenge**:
  Multiple API Gateway instances (10-100+), each handling 10K req/sec, all must enforce same limit (100 req/min per user), without coordination → race conditions (2 instances allow request simultaneously → user gets 101 requests in window).

- **Redis as shared state**:
  All API Gateway instances connect to same Redis cluster, Token state stored in Redis: rate_limit:user123 → {tokens: 95, last_refill: 1706270445}, Every rate limit check queries Redis (centralized truth).

- **Race condition problem**:
  Request 1 (Instance A): GET tokens → 1 token remaining, allow request, Request 2 (Instance B): Simultaneously GET tokens → also sees 1 token, also allows, DECR tokens (both execute), Result: tokens = -1 (over-limit, race condition!).

- **Solution**:
  Redis Lua script (atomic execution): lua_script = local key = KEYS[1], local capacity = tonumber(ARGV[1]), local refill_rate = tonumber(ARGV[2]), local now = tonumber(ARGV[3]), local bucket = redis.call('HMGET', key, 'tokens', 'last_refill'), local tokens = tonumber(bucket[1]) or capacity, local last_refill = tonumber(bucket[2]) or now, local elapsed = now - last_refill, local refilled = math.min(capacity, tokens + (elapsed * refill_rate)), if refilled >= 1 then redis.call('HMSET', key, 'tokens', refilled - 1, 'last_refill', now), redis.call('EXPIRE', key, 3600), return 1 -- allow, else return 0 -- reject, end, API Gateway call: result = redis.eval(lua_script, 1, 'rate_limit:user123', 100, 10, current_timestamp), Atomicity: Lua script runs as single atomic operation in Redis, no other command can interleave, guarantees: only one instance decrements token, race condition prevented.

- **Consistency guarantees**:
  Redis single-threaded execution: Commands processed serially (one at a time), Lua scripts: Execute atomically (all-or-nothing), even across Redis Cluster, Result: Distributed consistency without locks, CAP tradeoff: CP within Redis (consistent), AP across system (if Redis down, fail open for availability).

- **Redis Cluster (horizontal scaling)**:
  Multiple Redis shards (master + replicas), Consistent hashing: hash(user_id) → shard (user123 always routes to same shard), Read from replicas: GET operations can use replicas (reduce master load), Write to master: EVAL (Lua script) must go to master (consistency), Replication lag: <1ms (acceptable slight over-limit during lag).

- **Performance optimization**:
  Connection pooling: Each API Gateway maintains 50 Redis connections (reuse, avoid handshake), Pipelining: Batch multiple rate limit checks (if checking 10 users → send 10 EVAL in one round-trip), Redis latency: p99 <5ms (fast enough for rate limiting, doesn't bottleneck API), Local cache: Cache policy rules (limit, refill_rate) locally (avoid Redis query for policy, only check token state).

- **Failure handling**:
  Redis unavailable: Circuit breaker: After 10 consecutive Redis failures → open circuit for 30 sec, During open circuit: Fail open (allow all requests, log error) OR fail closed (reject with 503), Auto-recovery: After 30 sec → half-open (try one request), if succeeds → close circuit (resume normal), Monitoring: Alert if Redis errors > 1% (investigate), track latency (if p99 > 10ms → scaling issue).

- **Multi-region**:
  Redis in each region: us-east-1, eu-west-1, asia-south-1 (low latency), Global rate limit: Aggregate across regions (complex, eventual consistency), OR per-region limit: 100 req/min per region (simpler, slightly relaxed global limit). Result: Distributed rate limiting with strong consistency (Redis atomic Lua), scales horizontally (Redis Cluster + multiple API Gateways), low latency (<5ms), handles failures gracefully (circuit breaker).

---

### Q3: Walk through complete request flow: client makes request → rate limiter checks → either allowed or rejected. Include policy lookup, token bucket check, and 429 response with headers.

Complete rate limiting flow (Token Bucket):

- **Client request**:
  GET /api/users (with header: Authorization: Bearer {jwt_token}), Load Balancer: Routes to API Gateway instance (round-robin).

- **API Gateway authentication**:
  Extract JWT: Parse token, validate signature, extract user_id: 'user123', (or extract api_key from header: X-API-Key).

- **Rate Limiter**:
  Identify subject: user_id='user123', endpoint='/api/users', Fetch policy (with caching): Check Redis cache: policy:user123:/api/users, if cache hit: policy = {limit: 100, window: 60, refill_rate: 10, enforce_by: 'USER'}, if cache miss: Query Policy DB: SELECT limit, window, refill_rate, enforce_by FROM rate_limit_rules WHERE (subject_type='USER' AND subject_value='user123' AND endpoint='/api/users') OR (subject_type='USER' AND subject_value='user123' AND endpoint='*') OR (subject_type='TIER' AND subject_value=(SELECT tier FROM clients WHERE user_id='user123') AND endpoint='*'), Fetch first matching rule (specific endpoint > wildcard, user > tier), Cache: SET policy:user123:/api/users {limit: 100, window: 60, refill_rate: 10} EX 3600.

- **Token Bucket check (Redis Lua script)**:
  Prepare: key = 'rate_limit:user123:/api/users', capacity = 100, refill_rate = 10, now = current_unix_timestamp, Execute: result = redis.eval(lua_script, 1, key, capacity, refill_rate, now), Lua script logic: Fetch bucket: tokens, last_refill = redis.call('HMGET', key, 'tokens', 'last_refill'), Calculate refill: elapsed = now - last_refill (e.g., 5 seconds since last request), refilled = min(capacity, tokens + elapsed × refill_rate) = min(100, 90 + 5×10) = min(100, 140) = 100 tokens (capped at capacity), Check: if refilled >= 1 (100 >= 1 → true), Consume: tokens = 100 - 1 = 99, Update: redis.call('HMSET', key, 'tokens', 99, 'last_refill', now), Return: 1 (allow).

- **Request allowed**:
  result = 1 (allow), Calculate remaining: remaining = 99 tokens, Calculate reset time: reset = now + (capacity - remaining) / refill_rate = now + (100 - 99) / 10 = now + 0.1 sec (when bucket fully refilled), Forward request: Proxy to Backend Svc: GET /api/users, Add headers: X-RateLimit-Limit: 100, X-RateLimit-Remaining: 99, X-RateLimit-Reset: {reset_timestamp}, Backend response: {users: [...]}, Forward to client: 200 OK with data + rate limit headers.

- **Request rejected (if no tokens)**:
  Scenario: User already made 100 requests in last 10 sec, tokens = 0, Lua script: refilled = min(100, 0 + 0×10) = 0 (no refill yet, too soon), Check: refilled >= 1 → false, Return: 0 (reject),

- **Return 429 Too Many Requests**:
  Status: 429 Too Many Requests, Headers: X-RateLimit-Limit: 100, X-RateLimit-Remaining: 0, X-RateLimit-Reset: 1712340788 (Unix timestamp when bucket refills to 1 token), Retry-After: 1 (seconds until retry allowed, ceil((1 - 0) / 10) = 1 sec), Body (JSON): {error: 'rate_limit_exceeded', message: 'You have exceeded 100 requests per minute. Retry after 1 second.', limit: 100, remaining: 0, reset_at: '2024-04-05T10:01:28Z'}, Client behavior: Receive 429, parse Retry-After: 1, wait 1 second, retry request → likely succeeds (1 token refilled).

- **Monitoring**:
  Log rate limit events: {user_id, endpoint, allowed: false, tokens_remaining: 0, timestamp}, Metrics: Increment counter: rate_limit_rejections.incr(labels={endpoint='/api/users', tier='FREE'}), Track: Rejection rate (% of requests rejected), per-endpoint breakdown, Dashboard: Alert if rejection rate > 5% (potential attack or misconfigured limit). Result: Complete flow with policy caching (fast lookup), atomic token check (race-free), proper 429 response (helps client implement backoff), monitoring (visibility into rate limiting).

---

## 7. Interview Summary & Key Takeaways

| Architectural Dimension | Bunny's Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Placement Tier** | API Gateway / Reverse Proxy Interceptor | Client-side or Individual Backend Services | Client-side is easily hacked; per-backend pod counters fail under horizontal replication. Gateway enforces globally before resource consumption. |
| **Algorithm Selection** | Token Bucket Algorithm | Fixed Window Counter or Sliding Window Log | Fixed window suffers from $2\times$ boundary bursts; sliding log consumes massive $O(N)$ memory. Token bucket is $O(1)$ memory and burst-friendly. |
| **Concurrency & State** | Centralized Redis Cluster with Atomic Lua Scripting | Local in-memory caches or Distributed Locks (Redlock) | Avoids non-atomic read-modify-write race conditions. Lua scripts execute atomically in Redis without distributed locking overhead. |
| **System Reliability** | Fail-Open with Circuit Breaker | Fail-Closed | High availability priority (AP). A rate limiter failure must never cause global service outage for paying customers. |
| **Extensibility Model** | Strategy + Factory Design Pattern | Hardcoded conditional `if/else` checks | Allows dynamic algorithm swapping at runtime per client tier without gateway restarts. |
