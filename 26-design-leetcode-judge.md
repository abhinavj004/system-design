# System Design: Online Code Judge Platform (LeetCode / HackerRank / Codeforces)

> **Source Reference**: [Dodesimo System Design Notes: Design LeetCode](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20Leetcode.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG Standard).  
> **Topic**: Designing a Secure, High-Throughput Online Code Execution Engine, Multi-Language Sandbox, and Real-Time Contest Leaderboard Platform.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: The client writes code in a web-based **Monaco Editor** and submits it via `POST /submit`. The API server writes the submission to the database with status `PENDING`, pushes the job to a **Message Queue partitioned by language**, and returns `HTTP 202 Accepted` immediately. A horizontally scaled pool of **Judge Workers** pulls jobs and runs the code inside **hardened Container Sandboxes (Docker with cgroups v2, seccomp syscall filters, read-only rootfs, and zero network `--net=none`)**. A language test harness feeds generic JSON test cases to the code. Results (`AC`, `WA`, `TLE`, `MLE`) are saved to the DB, and clients poll `GET /check/:id` until complete. Live competition rankings are updated atomically in **Redis Sorted Sets (`ZSET`)**.

| Metric / Dimension | Production Target / Value (Dodesimo Scale) |
| :--- | :--- |
| **System Scale** | $\approx 4,000\text{ problems}$; a few hundred thousand registered engineers |
| **Contest Concurrency** | **100,000 simultaneous competitors** in timed weekly contests |
| **Submission Turnaround**| Results returned in **$< 5\text{ seconds}$** |
| **Contest Execution Load**| 10k submissions $\times 100\text{ test cases} \times 100\text{ms} = \mathbf{27\text{ CPU hours}}$ of compute |
| **Security Ceiling** | CPU: 1.0s limit; RAM: 256MB limit; PIDs: max 32 (anti-forkbomb) |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **The Synchronous Execution Trap**:
   - *Trap*: *"Why can't the API server compile and run the user's code directly in the HTTP request handler?"*
   - *Defense*: Untrusted code is hostile and CPU-heavy. Running code on the API server allows a single `while(true)` or `fork()` bomb to crash the entire web server, taking down the platform for all users. Submissions must be **asynchronous via message queues** into isolated worker sandboxes.
2. **Standard Docker Isolation Flaws**:
   - *Trap*: *"Is standard Docker containerization completely secure for untrusted code execution?"*
   - *Defense*: Standard Docker containers share the host Linux kernel. A zero-day privilege escalation or kernel panic compromises the host. We must enforce **defense-in-depth**: `seccomp` system call filtering (blocking `fork`, `execve`, sockets), **cgroups v2** (hard CPU and memory caps), **read-only filesystems**, and strict network airgapping (`--net=none`).
3. **The 100k Contest Leaderboard Meltdown**:
   - *Trap*: *"How do you calculate live rankings for 100,000 contestants submitting code simultaneously?"*
   - *Defense*: Running SQL `GROUP BY userId` with `COUNT(DISTINCT problemId)` crushes the database under contest load. Instead, use a **Redis Sorted Set (`ZSET`)** where the score encodes solved problem count and penalty time, allowing $O(\log N)$ rank lookups with progressive client polling.

> **Golden Architectural Rule**: Never execute untrusted code on shared API server processes. Always isolate untrusted execution behind asynchronous queues with multi-layered container security ceilings.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Problem Catalog**: Users can view a paginated list of algorithmic coding problems filtered by difficulty and tags (`GET /problems`).
2. **Problem Details & Code Stubs**: Users can view a specific problem's description, sample test cases, and starter code stubs in multiple programming languages (`GET /problems/:id`).
3. **Submit Code & Get Feedback**: Users can submit code in Python, C++, Java, JavaScript, or Go, and receive instant feedback: Accepted (`AC`), Wrong Answer (`WA`), Time Limit Exceeded (`TLE`), Memory Limit Exceeded (`MLE`), or Runtime Error (`RTE`).
4. **Live Contest Leaderboard**: Competitors can view a live ranked leaderboard during timed coding contests, ranked by number of problems solved and tie-broken by submission penalty time.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Availability over Consistency (AP)**: The problem catalog and submissions queue must remain highly available during massive contest traffic spikes.
2. **Absolute Sandboxing & Security**: Untrusted user code must never escape its sandbox, access the host filesystem, make outbound network requests, or exhaust host resources.
3. **Low Latency SLA**: Submissions must evaluate and return results to the user within **$< 5\text{ seconds}$**.
4. **Massive Contest Concurrency**: Must support **100,000 concurrent participants** submitting code simultaneously during weekly competitions.

---

### 1.3 System Scale & Capacity Math (The 27 CPU Hours Contest Math)

#### Catalog & User Base Scale
- **Problem Catalog**: $\approx 4,000\text{ problems}$ (compact, easily cached in memory).
- **User Base**: A few hundred thousand active engineers, peaking at **100,000 concurrent users** during timed contests.

#### The Contest Execution Bottleneck Math (Dodesimo Model)
- Suppose a weekly contest has **10,000 active submissions** during the first 10 minutes:
  - Each problem has $\approx 100\text{ test cases}$.
  - Average execution time per test case $\approx 100\text{ ms}$.
  - Execution time per submission:
    $$\text{Time per Submission} = 100\text{ test cases} \times 100\text{ ms} = 10\text{ seconds of CPU time}$$
  - Total compute required for 10,000 submissions:
    $$\text{Total Compute} = 10,000 \times 10\text{ s} = 100,000\text{ CPU seconds} \approx \mathbf{27.7\text{ CPU hours!}}$$
- *Conclusion*: A single machine running 27 hours of compute will fail. If we want all 10,000 submissions evaluated within a **1-minute window** ($60\text{ seconds}$):
  $$\text{Required Parallel CPU Cores} = \frac{100,000\text{ seconds}}{60\text{ seconds}} \approx \mathbf{1,667\text{ CPU Cores}}$$
  $\implies$ **We must distribute execution across an autoscaling pool of Judge Workers via asynchronous language-partitioned message queues!**

---

## 2. Step 2: Identification of Core Entities & Data Modeling

For LeetCode, a **NoSQL Document Database (e.g. Amazon DynamoDB or MongoDB)** is optimal for problems because:
1. Problems have no complex relational joins.
2. Starter code stubs for 10+ languages and dozens of test cases can be nested directly as subdocuments inside a single problem record.

```
┌────────────────────────────────────────────────────────────────────────┐
│                              Problem Document                          │
├────────────────────────────────────────────────────────────────────────┤
│ id: "two-sum"                                                          │
│ title: "Two Sum"                                                       │
│ question: "Given an array of integers nums and an integer target..."   │
│ level: "EASY"                                                          │
│ tags: ["array", "hash-table"]                                          │
│ codeStubs: {                                                           │
│   "python": "class Solution:\n    def twoSum(nums, target):...",       │
│   "cpp": "class Solution {\npublic:\n    vector<int> twoSum(...)...",  │
│   "java": "class Solution {\n    public int[] twoSum(...)..."          │
│ }                                                                      │
│ testCases: [                                                           │
│   { "input": "[2,7,11,15], 9", "expected_output": "[0,1]" },           │
│   { "input": "[3,2,4], 6",     "expected_output": "[1,2]" }            │
│ ]                                                                      │
└────────────────────────────────────────────────────────────────────────┘
```

### Relational / Document Schema for Submissions

```sql
CREATE TABLE submissions (
    submission_id VARCHAR(64) PRIMARY KEY,
    problem_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    competition_id VARCHAR(64),          -- Nullable for regular practice
    code TEXT NOT NULL,
    language VARCHAR(20) NOT NULL,       -- python, cpp, java, go
    status VARCHAR(20) NOT NULL,         -- PENDING, RUNNING, AC, WA, TLE, MLE, RTE
    passed_cases INT DEFAULT 0,
    total_cases INT DEFAULT 0,
    runtime_ms INT,
    memory_kb INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for contest ranking queries
CREATE INDEX idx_submissions_comp ON submissions(competition_id, user_id, status, created_at);
```

---

## 3. Step 3: API & System Interface Design

### 3.1 View Problem List (Paginated)
```http
GET /api/v1/problems?page=1&limit=50&level=MEDIUM

Response: 200 OK
{
  "problems": [
    {
      "id": "two-sum",
      "title": "Two Sum",
      "level": "EASY",
      "tags": ["array", "hash-table"]
    }
  ],
  "total_pages": 80,
  "current_page": 1
}
```

### 3.2 View Specific Problem & Code Stub
```http
GET /api/v1/problems/two-sum?language=python

Response: 200 OK
{
  "id": "two-sum",
  "title": "Two Sum",
  "question": "Given an array of integers nums...",
  "code_stub": "class Solution:\n    def twoSum(self, nums: List[int], target: int) -> List[int]:\n        pass",
  "sample_cases": [
    { "input": "nums = [2,7,11,15], target = 9", "output": "[0,1]" }
  ]
}
```

### 3.3 Submit Solution (Asynchronous Evaluation)
```http
POST /api/v1/problems/two-sum/submit
Content-Type: application/json
{
  "code": "class Solution:\n    def twoSum(self, nums, target):\n        lookup = {}\n        for i, n in enumerate(nums):\n            if target - n in lookup: return [lookup[target-n], i]\n            lookup[n] = i",
  "language": "python",
  "competition_id": "contest_weekly_410"
}

Response: 202 Accepted
{
  "submission_id": "sub_98124",
  "status": "PENDING",
  "check_url": "/api/v1/submissions/sub_98124/check"
}
```

### 3.4 Poll Submission Execution Result
```http
GET /api/v1/submissions/sub_98124/check

Response: 200 OK (When complete)
{
  "submission_id": "sub_98124",
  "status": "ACCEPTED",
  "runtime_ms": 48,
  "memory_kb": 16800,
  "passed_cases": 100,
  "total_cases": 100
}
```

### 3.5 View Contest Leaderboard
```http
GET /api/v1/leaderboard/contest_weekly_410?page=1&limit=50

Response: 200 OK
{
  "rankings": [
    { "rank": 1, "user_id": "usr_tourist", "score": 4, "penalty_time": "00:32:15" },
    { "rank": 2, "user_id": "usr_coder99", "score": 4, "penalty_time": "00:41:02" }
  ]
}
```

---

## 4. Step 4: High-Level Architecture (HLD)

### Component Interaction Topology

```
[ Web Browser (Monaco Code Editor) ]
               │
               │ (1. POST /problems/:id/submit)
               ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   API Gateway Server                                   │
│  (Authenticates JWT, Validates Code Size < 64KB, Writes Submission PENDING to DB)       │
└────────────────────────────────────────────────────────────────────────────────────────┘
         │                                                        │
         │ (2. Enqueue Job)                                       │ (4. Poll Results)
         ▼                                                        ▼
┌───────────────────────────────────────────────┐        ┌───────────────────────────────┐
│     Asynchronous Submission Message Queue     │        │      PostgreSQL Database      │
│  Topic: `submissions` (Partitioned by Lang)   │        │     (Submissions & State)     │
└───────────────────────────────────────────────┘        └───────────────────────────────┘
         │                                                        ▲
         │ (3. Pull Job)                                          │ Update Status (AC, WA..)
         ▼                                                        │
┌─────────────────────────────────────────────────────────────────┴──────────────────────┐
│                                   Judge Worker Pool                                    │
│   (Pulls job, loads test cases from Cache, feeds into isolated Container Sandbox)      │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼ (Executes Code inside Sandbox)
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        Hardened Container Sandbox (Docker / gVisor)                    │
│   ┌────────────────────────────────────────────────────────────────────────────────┐   │
│   │ • Read-Only Root Filesystem + Ephemeral tmpfs (50MB)                           │   │
│   │ • Linux cgroups v2: CPU cap (1.0s), RAM cap (256MB), PIDs cap (max 32)         │   │
│   │ • Linux seccomp: Blocks hostile syscalls (fork, execve, socket, connect)       │   │
│   │ • Air-gapped Network: `--net=none` (Zero inbound/outbound internet access)     │   │
│   └────────────────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼ (If AC in Contest)
                               ┌───────────────────┐
                               │ Redis Sorted Sets │ ◀─── Client reads Leaderboard
                               │ (Live Leaderboard)│
                               └───────────────────┘
```

---

### 4.1 Step-by-Step Architectural Evolution (Whiteboard Script)

#### Phase 1: The Naive Baseline MVP (Monolith + In-Process Execution)
- **Whiteboard**: `Client (Monaco)` ──► `API Server` ──► `Saves File & Runs system("python solution.py")`.
- **Explanation**: *"The API server receives code, writes it to a local temporary file, runs it using a shell command, and returns the output directly in the HTTP response."*
- **The Fatal Flaw**:
  1. *Hostile Takeover*: An adversarial user submits `import os; os.system("rm -rf /")` or steals API server database credentials.
  2. *Server Crash*: A single infinite loop (`while True: pass`) pins the API server's CPU to 100%, causing the web server to become unresponsive for all other users.

#### Phase 2: Isolation via Dedicated Virtual Machines (VMs)
- **Whiteboard**: API server invokes code inside dedicated Virtual Machines (e.g. AWS EC2).
- **Explanation**: *"To prevent security leaks, we execute code inside dedicated VMs where hardware virtualization prevents host system compromise."*
- **Bottlenecks**: VMs are resource-intensive and take **10 to 30 seconds to boot**, making it impossible to evaluate submissions within the $< 5\text{s}$ SLA.

#### Phase 3: Lightweight Container Sandboxes (Docker `exec`)
- **Whiteboard**: Use pre-warmed Docker containers per language runtime.
- **Explanation**: *"Containers share the host kernel and boot in $< 100\text{ ms}$. We maintain a warm pool of language containers and use `docker exec` to run the code."*
- **Remaining Risk**: Containers share the host kernel. A kernel exploit or fork bomb can still starve host resources unless strict limits are applied.

#### Phase 4: Multi-Layered Hardening (cgroups, seccomp, `--net=none`)
- **Whiteboard**: Add **cgroups v2** (CPU/RAM caps), **seccomp filters** (syscall whitelisting), **read-only filesystems**, and **network airgapping**.
- **Explanation**: *"We lock down the container: limit CPU to 1.0s, RAM to 256MB, block network access completely, and strip out syscalls like `fork` and `socket`."*

#### Phase 5: Decouple Execution via Message Queues for 100k Contests
- **Whiteboard**: Insert a **Message Queue (Kafka / RabbitMQ)** between the API Gateway and the Judge Workers.
- **Explanation**: *"During weekly contests, 10,000 users submit code at the exact same minute. The API server returns `HTTP 202 Accepted` immediately. Workers pull from language-partitioned queues and process jobs at a steady, sustainable rate without dropping requests."*

#### Phase 6: Live Contest Leaderboard via Redis Sorted Sets
- **Whiteboard**: Add a **Redis `ZSET`** updated on every accepted contest submission.
- **Explanation**: *"Instead of running slow SQL `GROUP BY` queries on the submissions table, we record scores and penalty timestamps in Redis Sorted Sets, enabling instant sub-millisecond rank lookups."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: Execution Sandboxing Progression (API Server vs. VM vs. Container vs. Serverless)

| Execution Model | Isolation Level | Boot Latency | Resource Overhead | Verdict / Trade-off |
| :--- | :--- | :--- | :--- | :--- |
| **1. Direct API Server Process** | **None** (Shared process) | $0\text{ ms}$ | Minimal | **Fatal**: Code can delete server files, steal environment keys, or crash the server. |
| **2. Virtual Machine (VM)** | **Hardware-Level (Hypervisor)** | $10 - 30\text{ s}$ | High (Full OS per VM) | Secure, but far too slow and resource-heavy for real-time code evaluation. |
| **3. Container Sandbox (Docker / gVisor)** | **OS-Level (Namespaces + cgroups)** | $< 50\text{ ms}$ | Low (Shared kernel) | **Ideal**: Pre-warmed containers provide instant execution with hardened kernel boundaries. |
| **4. Serverless (AWS Lambda)** | **MicroVM (Firecracker)** | $100 - 500\text{ ms}$ (cold) | Pay-per-use | Good for low maintenance, but cold starts add unacceptable latency for competitive contests. |

---

### 5.2 Deep Dive 2: Multi-Layered Sandbox Security (Defense-in-Depth)

How do we safely execute untrusted C++, Python, and Java code?

```
┌────────────────────────────────────────────────────────────────────────┐
│                   THE 5 LAYERS OF SANDBOX DEFENSE                      │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Read-Only Rootfs : File system is read-only; ephemeral /tmp capped  │
│ 2. cgroups v2 Caps  : cpu.max = 100ms/1s; memory.max = 256M; pids = 32 │
│ 3. seccomp-bpf      : Blocks fork, clone, execve, socket, connect      │
│ 4. Network Airgap   : --net=none (Zero socket/DNS access)              │
│ 5. Execution Watchdog: Hard OS SIGKILL signal after 2.000 seconds      │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Read-Only Filesystem & Ephemeral `tmpfs`**:
   - The container's root filesystem is mounted strictly **read-only** (`--read-only`).
   - Code can only write to a temporary in-memory directory (`/tmp`) mounted with `tmpfs` and capped at **50 MB**. The directory is destroyed immediately after the submission finishes.
2. **Linux cgroups v2 (Resource Ceilings)**:
   - `cpu.max = 100000 100000` (Strictly 1 CPU core allocation).
   - `memory.max = 268435456` (Hard 256MB RAM cap; exceeding this triggers `MLE` via immediate SIGKILL without crashing the host).
   - `pids.max = 32` (Prevents fork-bombs like `:(){ :|:& };:` from spawning infinite processes).
3. **Linux `seccomp` (System Call Filtering)**:
   - Untrusted code needs only basic arithmetic, memory allocation, and standard I/O (`read`, `write`, `exit`).
   - We inject a strict `seccomp` profile that disables dangerous syscalls:
     - `socket`, `connect`, `bind` (Prevents SSRF, network scanning, and cryptomining botnet connections).
     - `fork`, `vfork`, `clone` (Prevents multiprocessing and fork attacks).
     - `execve` (Prevents invoking external shell binaries).
4. **Network Airgapping (`--net=none`)**:
   - Docker containers are started with `--network none`. No virtual ethernet interface (`eth0`) is attached, physically blocking all inbound and outbound IP packets.
5. **Execution Timeout Watchdog**:
   - The worker runs the process under an OS timer. If the process does not terminate within **2.00 seconds**, the host issues a `SIGKILL` and marks the submission as **Time Limit Exceeded (`TLE`)**.

---

### 5.3 Deep Dive 3: The 100,000 Contestant Scale (Queues vs. Polling vs. WebSockets)

During a contest, 10,000 submissions arrive within seconds, requiring **27 CPU hours** of compute. How do we deliver results in $< 5\text{ seconds}$ without dropping connections?

```
  User clicks Submit
         │
         ▼
[ API Gateway Server ] ──▶ (Returns HTTP 202 Accepted + submission_id in <30ms)
         │
         ▼ (Enqueues job)
[ Kafka Topic: submissions ] (Partitioned by language: python, cpp, java, go)
         │
         ▼
[ Autoscaling Judge Worker Fleet ] ──▶ [ Hardened Container Sandbox ]
         │
         ▼
[ PostgreSQL: Status = ACCEPTED ]
         ▲
         │ Client Polls GET /check/:id (Progressive: 1s, 2s, 3s)
[ Web Client ]
```

#### Why Asynchronous Polling Beats WebSockets Here:
- While WebSockets provide instant push notifications, maintaining **100,000 idle persistent WebSocket connections** on the API servers consumes enormous socket file descriptor memory.
- Since a code run takes between **1 to 5 seconds**, **Progressive Short Polling** (`GET /submissions/:id/check` every 1 second for up to 5 attempts) is dramatically simpler, stateless, and fault-tolerant.
- If a client disconnects or refreshes the page, their submission still completes safely in the background.

---

### 5.4 Deep Dive 4: Fast Competition Leaderboard Architecture

How do we rank 100,000 competitors in real time with penalty times?

#### The SQL Anti-Pattern (Crushes Database):
```sql
SELECT user_id, 
       COUNT(DISTINCT problem_id) AS solved_count, 
       MIN(created_at) AS last_solve_time 
FROM submissions 
WHERE competition_id = 'contest_410' AND status = 'ACCEPTED' 
GROUP BY user_id 
ORDER BY solved_count DESC, last_solve_time ASC;
```
- Running this aggregate query every 5 seconds for 100,000 users reading the leaderboard will cause $100\%$ database CPU lockup!

#### The Production Solution: Redis Sorted Sets (`ZSET`)
- Maintain a single Redis Sorted Set for the contest: `leaderboard:contest_410`.
- **Composite Score Encoding**:
  - In competitive programming, higher solved count wins; if tied, smaller penalty time wins.
  - We encode both into a single 64-bit float score:
    $$\text{Score} = (\text{Solved Count} \times 10^{10}) + (10^{10} - \text{Penalty Seconds})$$
- **On Accepted Submission**:
  ```text
  ZADD leaderboard:contest_410 <computed_score> "usr_tourist"
  ```
- **Fetching the Top 100 Leaderboard**:
  ```text
  ZREVRANGE leaderboard:contest_410 0 99 WITHSCORES
  ```
  Executes in $O(\log N + M) \approx \mathbf{1\text{ ms}}$ directly from memory!
- **Progressive Client Polling**:
  - Competitors poll the leaderboard every 15 seconds during the contest, increasing to every 3 seconds in the final 5 minutes of the contest.

---

### 5.5 Deep Dive 5: Multi-Language Test Harness & Serialization

How do we evaluate solutions across Python, C++, Java, and Go using a single unified set of test cases?

```
┌────────────────────────────────────────────────────────┐
│               Generic JSON Test Cases                  │
│       [ { "input": [2, 7, 11, 15], "target": 9 },      │
│         { "expected": [0, 1] } ]                       │
└────────────────────────────────────────────────────────┘
                           │
                           ▼ (Fed via stdin)
┌────────────────────────────────────────────────────────┐
│             Language-Specific Test Harness             │
│  (Pre-compiled driver wrapper for Python, C++, Java)   │
├────────────────────────────────────────────────────────┤
│ 1. Deserializes JSON input into native types           │
│    (e.g., std::vector<int> nums, int target)           │
│ 2. Invokes user's function: Solution().twoSum(...)     │
│ 3. Captures return value and compares with expected    │
│ 4. Serializes output verdict (AC / WA) to stdout       │
└────────────────────────────────────────────────────────┘
```

1. **Standardized Input/Output Representation**: Test cases are authored once in canonical JSON format.
2. **Language Harness Driver**:
   - Each supported language has a pre-built harness wrapper.
   - For example, in C++, the harness includes the user's `Solution` class, parses JSON test cases via `nlohmann/json`, executes `twoSum`, and compares results using custom comparator logic (e.g. order-insensitive checks for sets).
3. **Execution Exit Codes**:
   - Exit code `0` + output match $\implies$ **`ACCEPTED` (`AC`)**.
   - Exit code `0` + output mismatch $\implies$ **`WRONG_ANSWER` (`WA`)**.
   - Exit code `137` (SIGKILL by watchdog) $\implies$ **`TIME_LIMIT_EXCEEDED` (`TLE`)**.
   - Exit code `139` (SIGSEGV / segmentation fault) $\implies$ **`RUNTIME_ERROR` (`RTE`)**.

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Execution Sandboxing**| Pre-warmed Container Sandboxes (Docker/gVisor) | Direct API Server Execution | Direct execution allows untrusted code to compromise or crash the API host. Containers provide safe OS-level isolation. |
| **Security Ceiling** | Linux cgroups v2 + seccomp-bpf + `--net=none` | Standard Unrestricted Containers | Standard containers share kernel; seccomp syscall filtering and cgroups prevent fork-bombs and network attacks. |
| **Contest Concurrency** | Language-Partitioned Queues + Autoscaling Workers | Synchronous HTTP Evaluation | Contests generate 27 CPU hours of burst compute. Message queues buffer traffic safely without dropped requests. |
| **Submission Polling** | Progressive Short Polling (`GET /check/:id`) | WebSockets Push Notifications | WebSockets consume massive server memory for 100,000 connections during short 1–5 second execution lifecycles. |
| **Contest Leaderboard** | Redis Sorted Set (`ZSET`) with composite score | SQL `GROUP BY` Aggregation | SQL aggregate queries melt the database under 100k contest participants. Redis `ZSET` serves top rankings in $< 1\text{ ms}$. |
| **Problem Storage** | NoSQL Document Store (DynamoDB / MongoDB) | Complex Relational Tables (3NF) | Problems are self-contained documents with nested language stubs and test cases; NoSQL eliminates multi-table joins. |

---

### Top 4 Interview Follow-Up Questions (Rapid-Fire Defenses)

### Q1: How do you prevent two contestants from having identical rankings on the leaderboard?
- **Answer**: Encode a unique tie-breaker into the Redis Sorted Set score: `score = (solved_count * 10^10) + (10^10 - total_penalty_seconds)`. If both solved count and penalty time are identical, use the contestant's registration timestamp or user ID as a secondary fractional tie-breaker.

### Q2: What happens if a Judge Worker crashes in the middle of executing a user's submission?
- **Answer**: The worker acknowledges the message from the queue only **after** the evaluation completes and the result is committed to the database. If the worker crashes, the queue's acknowledgment timeout expires and the message is automatically re-delivered to another healthy worker. A retry limit (e.g. 3 attempts) sends persistently failing jobs to a Dead Letter Queue (DLQ).

### Q3: How do you handle custom data structures like `TreeNode` or `ListNode` in test cases?
- **Answer**: The language test harness includes standard deserialization routines that convert array representations (e.g., `[3, 9, 20, null, null, 15, 7]`) into linked memory nodes (`TreeNode*`) before calling the user's function, and serializes the returned root node back into an array for comparison.

### Q4: How do you protect against "memory limit exceeded" without bringing down the host?
- **Answer**: Set the Linux cgroup limit `memory.max = 256M` and `memory.swap.max = 0` (disable swap). When the user's code attempts to allocate beyond 256MB, the Linux kernel's OOM killer terminates **only the container process** with signal `SIGKILL` (exit code 137). The judge worker detects this and returns `MEMORY_LIMIT_EXCEEDED` (`MLE`).
