# The Master System Design Blueprint: The 5-Step FAANG Interview Pattern

> **Context**: This blueprint is synthesized from the **Interview with Bunny** System Design methodology. It represents the structured, repeatable 5-step delivery framework used across all high-level and low-level system design rounds.
> 
> **Core Objective**: We will use this exact mental model and delivery framework not only for the videos in this playlist, but also to architect any advanced system design problems that aren't covered in the playlist (e.g., Flash Sale, LeetCode Code Runner, Distributed Web Crawler, Real-Time Ad Aggregator, etc.).

---

## The 5-Step Architectural Blueprint

```
Step 1: Requirements & Estimations (Clarify Functional/Non-Functional Scope & Math)
                            │
                            ▼
Step 2: Core Entities & Domain Modeling (Actors, Primary Objects & Relationships)
                            │
                            ▼
Step 3: API & Interface Design (Public REST Contracts, Payloads & Status Codes)
                            │
                            ▼
Step 4: High-Level Architecture (Microservices, Load Balancer, Abstraction)
                            │
                            ▼
Step 5: Low-Level Deep Dives (Algorithms, DB Choice, Caching, Edge Cases & Trade-offs)
```

---

## Detailed Step Breakdown

### Step 1: Requirements Gathering & Estimations (0–8 Minutes)
Never jump straight into drawing boxes. The first 8 minutes define success or failure:

1. **Clarifying Questions**:
   - What are the core user personas?
   - What features are explicitly in-scope vs. out-of-scope?
2. **Functional Requirements (FRs)**:
   - Write down 4 to 6 numbered, unambiguous user capabilities (e.g., *User can upload file, User can download file, Devices sync automatically*).
3. **Non-Functional Requirements (NFRs)**:
   - **Availability vs. Consistency (CAP Theorem)**: Is this system read-heavy (AP) or financial/inventory-critical (CP)?
   - **Latency Budget**: Expected response time for read paths (e.g., $<50\text{ms}$) vs. write paths (e.g., $<200\text{ms}$).
   - **Durability & Fault Tolerance**: Can data be lost? (e.g., 11 9's durability).
4. **Capacity Estimations (Back-of-the-Envelope Math)**:
   - **Scale**: Daily Active Users (DAU), Monthly Active Users (MAU).
   - **Throughput**: Read QPS and Write QPS (Read-to-Write ratio, e.g., 100:1).
   - **Storage Growth**: Payload size $\times$ writes/sec $\times$ 5 years.
   - **Network Bandwidth**: Ingress (incoming MB/s) and Egress (outgoing MB/s).
   - **Memory / Cache**: 80-20 Pareto principle ($20\%$ of daily read volume cached in RAM).

---

### Step 2: Identification of Core Entities (8–12 Minutes)
Identify the primary domain entities and their relationships before writing code or drawing schemas:
- **Actors**: `User`, `Admin`, `System Worker`.
- **Core Entities**: The nouns that represent persistent business state.
- **Entity Relationship**: 1-to-1, 1-to-many, or many-to-many.
- *Deliverable*: A clear entity diagram or relational entity table showing primary identifiers.

---

### Step 3: API & Interface Design (12–18 Minutes)
Define the public REST / gRPC interface. Bunny stresses that API design maps 1-to-1 to Functional Requirements:
- Use clean RESTful paths (e.g., `POST /api/v1/resource`, `GET /api/v1/resource/{id}`).
- Include headers: Authentication (`Authorization: Bearer <jwt>`), idempotency tokens (`X-Idempotency-Key`).
- Specify exact Request and Response JSON schemas.
- Explicit HTTP Status codes: `200 OK`, `201 Created`, `302 Found`, `400 Bad Request`, `409 Conflict`, `429 Too Many Requests`.

---

### Step 4: High-Level Architecture (18–25 Minutes)
Sketch the initial end-to-end architecture using clean microservice abstractions:
1. **Client Tier**: Web, Mobile, Desktop (with specialized client engines if needed).
2. **Gateway Tier**: DNS, CDN, Edge Rate Limiter, Load Balancers, API Gateway.
3. **Application Tier**: Decoupled microservices separated by business boundary (e.g., `Auth Service`, `Upload Service`, `Metadata Service`, `Notification Service`).
4. **State Tier**: Abstract databases (Relational / NoSQL), In-Memory Caches, Message Queues (Kafka / SQS), Blob Storage (S3).
- *Rule*: Keep components abstract in this step. Do not commit to specific brand names (e.g., Cassandra vs. PostgreSQL) until Step 5.

---

### Step 5: Low-Level Design & Deep Dives (25–45+ Minutes)
This is the core meat of the interview where the candidate separates themselves:
1. **Algorithm & Processing Strategy**:
   - How data is sliced, transformed, or computed (e.g., Base62 encoding, sliding window rate limiting, chunking algorithms, quad-trees).
2. **Database Technology Selection & Justification**:
   - Compare Relational (PostgreSQL/MySQL) vs. NoSQL (Cassandra, DynamoDB, MongoDB, Redis).
   - Justify based on access patterns: Is it relational ACID with complex joins, or massive write-throughput Key-Value lookups?
   - Define exact table schemas, primary keys, and partition/shard keys.
3. **Concurrency, Contention & Consistency**:
   - Distributed locking, optimistic locking, idempotent keys, atomic Redis Lua scripts.
4. **Caching & CDN Strategy**:
   - Cache-aside, Write-through, Write-back.
   - Cache eviction (LRU / LFU) and cache invalidation strategies.
5. **Asynchronous Pipelines & Decoupling**:
   - Using Kafka / RabbitMQ to offload heavy post-processing (validation, thumbnail generation, virus scanning, analytics).
6. **Failure Scenarios & Bottlenecks**:
   - What happens if a node crashes?
   - How do we handle network partitions, split-brain, and database failovers?

---

## The Standard Comparative Matrix Template
At the end of every design breakdown, summarize the key architectural trade-offs:

| Architectural Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Database Model** | e.g., NoSQL (Cassandra) | Relational (PostgreSQL) | High write throughput, horizontal scale, no join overhead. |
| **Coordination / ID Gen** | e.g., Range KGS (ZooKeeper) | MD5 Hash Truncation | Zero hash collisions, high performance, no database roundtrips. |
| **Caching Layer** | e.g., Redis Cluster (LRU) | Local In-Memory Cache | Centralized cache consistency across multiple stateless app pods. |
| **Traffic Ingress** | e.g., 302 Found Redirect | 301 Moved Permanently | 302 forces client to hit server on every click, preserving analytics tracking. |

---

## 🎯 Golden Rules for Simple, Interview-Ready Delivery (The Dodesimo & Bunny Standard)

To ensure all system designs remain clear, intuitive, and practical for an actual 45-minute FAANG/top-tier interview:

### 1. The 3-Minute Cram Box (Top of Document)
Every system design file must start with a **⚡ Last-Minute Interview Cram Box**:
- **30-Second Mental Model**: The single core sentence explaining how data flows from user to storage.
- **Key Metrics Table**: Peak QPS, storage footprint, latency SLAs, consistency model.
- **Top 3 Interviewer Traps & Must-Say Defenses**: The high-frequency trick questions interviewers ask and their immediate counter-arguments.
- **Golden Architectural Rule**: The one fundamental principle that anchors the design.

### 2. Plain-English Concepts Over Pretentious Jargon
- Never recite obscure kernel flags, syscall names, or low-level library syntax (e.g., avoid `seccomp-bpf`, `--net=none`, `cgroups v2`, `epoll_create1`).
- Speak in **plain, conversational English** that is easy to say out loud:
  - *"Mount the filesystem as read-only and allow writes only to an ephemeral temporary folder that is wiped immediately after execution."*
  - *"Enforce strict CPU and memory bounds so that an out-of-control process terminates without starving the host."*
  - *"Set an explicit 2-second timeout watchdog to kill infinite loops."*
  - *"Block network access so the code cannot make outbound calls or talk to the internet."*

### 3. Always Evaluate 3 Options for Key Decisions (Bad ──▶ Okay ──▶ Best)
When presenting technical choices (e.g., code execution, caching, replication, leaderboards), structure the explanation as an evolutionary progression:
- **Option 1 (Bad / Naive)**: The obvious beginner solution. Explain *why it breaks* (hostile security takeover, server crash, thundering herd, lock contention).
- **Option 2 (Okay / Intermediate)**: The first improvement. Explain *why it helps*, but identify its fatal flaw (too slow, high resource footprint, boot latency).
- **Option 3 (Champion / Production-Grade)**: The battle-tested FAANG standard (e.g., Docker container with `docker exec`, Redis Sorted Sets with progressive polling, asynchronous language queues).

### 4. Step 4 Must Include the End-to-End Happy Flow
Never leave the architecture as static boxes. Walk through the complete user lifecycle step-by-step:
- **Tracing the User Journey**: Numbered, chronological steps showing request $\to$ gateway $\to$ service $\to$ queue $\to$ database $\to$ client response.
- **Service Boundary Justifications**: Explicitly justify *why* two services are separated (e.g., *"Why is Bidding separated from Auction?"*, *"Why is Simulation separated from Target Discovery?"*):
  1. *Independent Scaling* (spiky write-heavy vs. steady read-heavy).
  2. *Isolation of Concerns & Failure Domains* (a crashed simulation must not bring down the catalog search).
  3. *Heterogeneous Hardware* (lightweight CPU pods vs. expensive GPU clusters).

### 5. Practical Protocols & Pragmatic Choices
- **Default to REST APIs**: State that internal services use standard REST over HTTPS with JSON for developer simplicity and maintainability.
- **Mention gRPC as a Targeted Optimization**: Only propose gRPC when an internal microservice call becomes a known high-frequency, low-latency bottleneck ($<5\text{ms}$ SLA or high serialization overhead).
- **Math Must Drive Architecture**: Estimations are not just vanity math—they must directly justify why a component exists (e.g., 27 CPU hours of burst compute $\implies$ message queue + worker pool; 10M concurrent auctions $\implies$ DB sharding).


