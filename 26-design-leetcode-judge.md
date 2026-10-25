# System Design: Online Code Judge Platform (LeetCode / HackerRank / Codeforces)

> **Source Reference**: [Dodesimo System Design Notes: Design LeetCode](https://github.com/Dodesimo/SystemDesignNotes/blob/main/Questions/Design%20Leetcode.md)  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG Standard).  
> **Topic**: Designing a Secure, High-Throughput Online Code Execution Engine, Multi-Language Sandbox, and Real-Time Contest Leaderboard Platform.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: The user writes code in a web-based **Monaco Editor** and submits it via `POST /submit`. The API server marks the submission as `PENDING` in the database, pushes the code to an **Asynchronous Message Queue partitioned by language** (e.g. Python, C++, Java), and returns `HTTP 202 Accepted` immediately. A pool of **Container Workers (Docker)** pulls jobs from the queue and executes the code using `docker exec` against hidden test cases in an isolated environment. The container enforces **read-only filesystems, strict CPU/memory limits, a 2-second timeout, and zero internet access**. Results (`AC`, `WA`, `TLE`, `MLE`) are saved to the database, and the client polls `GET /check/:id` until complete. Live competition scores are updated in a **Redis Sorted Set (`ZSET`)** for instant leaderboard ranking.

| Metric / Dimension | Production Target / Value (Dodesimo Scale) |
| :--- | :--- |
| **System Scale** | $\approx 4,000\text{ problems}$; a few hundred thousand registered engineers |
| **Contest Concurrency** | **100,000 simultaneous competitors** in timed weekly contests |
| **Execution Turnaround** | Results returned to user in **$< 5\text{ seconds}$** |
| **Contest Peak Compute**| 10k submissions $\times 100\text{ test cases} \times 100\text{ms} = \mathbf{27\text{ CPU hours}}$ |
| **Sandbox Ceilings** | CPU: 1.0s limit; RAM: 256MB limit; Network: Completely blocked |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **How to Execute User Code (The 3 Options)**:
   - *Option 1 (Bad - On API Server)*: Running code directly on the web server allows malicious code (like `rm -rf /` or infinite loops) to crash the server for everyone.
   - *Option 2 (Okay - Full Virtual Machines)*: Strongest security, but VMs take 10–30 seconds to boot and consume too much memory.
   - *Option 3 (Best - Container Sandbox with Docker)*: Lightweight, boots in $< 100\text{ms}$, and shares the OS kernel while isolating the process. Reusable language containers run code using `docker exec`.
2. **Handling 100,000 Contestants (The 27 CPU Hours Math)**:
   - *Trap*: *"Why not just run submissions synchronously when the user clicks Submit?"*
   - *Defense*: In a contest, 10,000 simultaneous submissions running 100 test cases take 27 hours of cumulative CPU time. A synchronous API server would run out of threads and crash. We make it **asynchronous via message queues**: the API returns `HTTP 202 Accepted` in 20ms, workers process jobs at a steady rate, and the client polls for results.
3. **The Live Leaderboard Meltdown**:
   - *Trap*: *"How do you calculate live rankings for 100,000 contestants during a contest?"*
   - *Defense*: Running a SQL `GROUP BY userId` query with `COUNT(DISTINCT problemId)` crushes the database under contest traffic. Instead, update a **Redis Sorted Set (`ZSET`)** on every accepted submission, allowing instant $O(\log N)$ rank lookups.

> **Golden Architectural Rule**: Never execute untrusted user code inside the API server process. Always buffer submissions in message queues and run them inside isolated containers with strict CPU, memory, timeout, and network boundaries.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **View List of Problems**: Users can browse a paginated catalog of coding problems filtered by difficulty (Easy, Medium, Hard) and tags.
2. **View Problem & Code Stubs**: Users can open a specific problem, view the description, sample test cases, and starter code stubs in multiple programming languages.
3. **Submit Solution & Instant Feedback**: Users can run and submit code in Python, C++, Java, or JavaScript, receiving instant verdicts:
   - Accepted (`AC`)
   - Wrong Answer (`WA`)
   - Time Limit Exceeded (`TLE`)
   - Memory Limit Exceeded (`MLE`)
   - Runtime Error (`RTE`)
4. **Live Competition Leaderboard**: Users can view a live ranked leaderboard during timed contests, ranked by number of problems solved with ties broken by earliest finish time.

---

### 1.2 Non-Functional Requirements (NFR)
1. **System Security & Isolation**: Hostile user code must never crash the server, access files of other users, or connect to the internet.
2. **Fast Feedback SLA**: Submissions must evaluate and return results within **$< 5\text{ seconds}$**.
3. **Contest Scalability**: Must easily handle **100,000 concurrent contestants** submitting code during weekly competitions.
4. **Availability over Consistency (AP)**: Browsing problems and queueing submissions must remain available even under extreme traffic bursts.

---

### 1.3 System Scale & Capacity Math (The 27 CPU Hours Calculation)

#### System Footprint
- **Problem Catalog**: $\approx 4,000\text{ problems}$ total (small dataset, can be cached entirely in memory).
- **Users**: A few hundred thousand registered users, with **100,000 active participants** during live weekly contests.

#### The Contest Bottleneck Math (Dodesimo Model)
- Imagine a weekly contest where **10,000 submissions** arrive in the first few minutes:
  - Each problem has $\approx 100\text{ test cases}$.
  - Average execution time per test case $\approx 100\text{ ms}$.
  - Time to evaluate 1 submission:
    $$\text{Time per Submission} = 100\text{ test cases} \times 100\text{ ms} = 10\text{ seconds of CPU time}$$
  - Total compute required for 10,000 submissions:
    $$\text{Total Compute} = 10,000 \times 10\text{ s} = 100,000\text{ CPU seconds} \approx \mathbf{27.7\text{ CPU hours!}}$$
- *Conclusion*: A single server trying to run 27 hours of compute will freeze and drop requests.
  - If we want to clear these 10,000 submissions within **60 seconds**, we need:
    $$\text{Required Parallel Cores} = \frac{100,000\text{ seconds}}{60\text{ seconds}} \approx \mathbf{1,667\text{ Parallel CPU Cores}}$$
  - **Architecture Decision**: We must decouple the API server from execution using **Asynchronous Message Queues** and an **Autoscaling Pool of Container Workers**.

---

## 2. Step 2: Identification of Core Entities & Data Modeling

For LeetCode, a **NoSQL Document Database (such as Amazon DynamoDB or MongoDB)** is ideal for problems because:
1. Problems don't require complex relational table joins.
2. Code stubs for 10+ languages and test cases can be stored cleanly as nested subdocuments inside the same problem record.

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
│   { "input": "[2,7,11,15], 9", "output": "[0,1]" },                    │
│   { "input": "[3,2,4], 6",     "output": "[1,2]" }                     │
│ ]                                                                      │
└────────────────────────────────────────────────────────────────────────┘
```

### Relational Schema for Submissions (PostgreSQL)

```sql
CREATE TABLE submissions (
    submission_id VARCHAR(64) PRIMARY KEY,
    problem_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    competition_id VARCHAR(64),          -- Nullable (set during contests)
    code TEXT NOT NULL,
    language VARCHAR(20) NOT NULL,       -- python, cpp, java, javascript
    status VARCHAR(20) NOT NULL,         -- PENDING, RUNNING, ACCEPTED, WRONG_ANSWER, TLE, MLE
    runtime_ms INT,
    memory_kb INT,
    passed_cases INT DEFAULT 0,
    total_cases INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for querying submissions and contest results
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
    { "id": "two-sum", "title": "Two Sum", "level": "EASY", "tags": ["array", "hash-table"] }
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
  "question": "Given an array of integers nums and an integer target...",
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
  "competition_id": "contest_410"
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
  "runtime_ms": 45,
  "memory_kb": 16500,
  "passed_cases": 100,
  "total_cases": 100
}
```

### 3.5 View Contest Leaderboard
```http
GET /api/v1/leaderboard/contest_410?page=1&limit=50

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
│  (Validates code size < 64KB, Writes Submission PENDING to DB, Enqueues to Queue)      │
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
│                                  Judge Worker Fleet                                    │
│       (Pulls job, loads test cases, feeds code into isolated Container Sandbox)        │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼ (Runs Code via `docker exec`)
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        Isolated Container Sandbox (Docker)                             │
│   • Read-only filesystem (writes allowed only to a temporary in-memory folder)         │
│   • Strict CPU limit (1.0s) and Memory limit (256MB)                                   │
│   • Timeout watchdog: terminates after 2.0s to stop infinite loops                     │
│   • Network blocked: zero internet or local network access                             │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼ (If AC in Contest)
                               ┌───────────────────┐
                               │ Redis Sorted Sets │ ◀─── Client reads Leaderboard
                               │ (Live Leaderboard)│
                               └───────────────────┘
```

---

### 4.1 End-to-End Happy Flow Across All Services

Here is how the complete system functions step-by-step during a user interaction:

#### Flow 1: Viewing a Problem
1. User opens a problem page in their browser.
2. The browser loads the **Monaco Editor** (the web-based code editor powering VS Code).
3. The client calls `GET /problems/:id?language=python`.
4. The API server fetches the problem description, sample test cases, and starter code stub from the database and displays it in the editor.

#### Flow 2: Submitting Solution & Async Execution
1. User writes their code and clicks **"Submit"**:
   - The browser sends a `POST /problems/:id/submit` with the code string and language.
2. The **API Server**:
   - Inserts a new record in the `submissions` table with status **`PENDING`**.
   - Pushes the submission message to the **Message Queue** on a topic partitioned by language (e.g. `queue:python`).
   - Immediately returns **`HTTP 202 Accepted`** with a `submission_id` to the browser ($< 30\text{ms}$).
3. A **Judge Worker**:
   - Pulls the job from the queue.
   - Fetches the 100 hidden test cases from cache or database.
   - Dispatches the code to a pre-warmed **Docker container** for that language using `docker exec`.
   - The container runs the code against all test cases.
4. The worker records the verdict:
   - If all test cases pass $\implies$ updates status to **`ACCEPTED`** in PostgreSQL.
   - If a test case fails $\implies$ updates status to **`WRONG_ANSWER`**.
   - If the code exceeds 2 seconds $\implies$ updates status to **`TIME_LIMIT_EXCEEDED`**.

#### Flow 3: Client Checking the Result (Progressive Polling)
1. After receiving the `HTTP 202 Accepted`, the client browser starts polling:
   - Polls `GET /submissions/:id/check` after 1 second, then 2 seconds, then 3 seconds.
2. Once the worker updates PostgreSQL to `ACCEPTED`, the check endpoint returns the final result, execution time, and memory used.
3. The editor UI lights up green with *"Accepted"*!

#### Flow 4: Updating the Contest Leaderboard
1. If the submission was part of an active competition:
   - The judge worker updates the contestant's score in a **Redis Sorted Set (`ZSET`)**.
2. Other competitors viewing the leaderboard poll `GET /leaderboard/:contestId`:
   - The API server reads the top 100 ranks directly from Redis in $< 1\text{ms}$.

---

### 4.2 The 3 Options for Executing User Code (Dodesimo Evaluation)

When designing a code execution engine, an interviewer will ask: *"How do you safely run arbitrary user code?"* Walk through these 3 options:

#### Option 1 (Bad): Run Code Directly on the API Server
- **How it works**: The API server saves the code to a local file (`solution.py`) and executes it using a system call: `exec("python solution.py")`.
- **Why it breaks**:
  1. *Hostile Takeover*: A malicious user submits `import os; os.system("rm -rf /")` or steals database credentials.
  2. *Server Crash*: A single infinite loop (`while True: pass`) pins the CPU to 100%, freezing the server for all other users.
  3. *No Isolation*: A crashing script takes down the entire web application.

#### Option 2 (Okay): Run Code Inside Virtual Machines (VMs)
- **How it works**: Spin up an isolated Virtual Machine (e.g., AWS EC2) for each submission.
- **Why it's good**: Provides complete hardware-level isolation. If user code crashes the VM, it can be reset without touching the main system.
- **The Fatal Flaw**: VMs are very resource-heavy and take **10 to 30 seconds to boot up**. It is impossible to meet the $< 5\text{s}$ turnaround SLA.

#### Option 3 (Best / Standard): Run Code in Container Sandboxes (Docker)
- **How it works**: Containers share the host OS kernel while isolating processes, enabling them to boot in **under 100 milliseconds**.
- **How production judges use it**:
  - The judge keeps a warm pool of language containers running.
  - When code arrives, it invokes `docker exec` to run the code inside the existing container against the test cases.
  - When execution finishes, any temporary files in memory are wiped clean for the next run.

*(Alternative Cloud Option - Serverless Functions)*: Running code inside AWS Lambda / Cloud Functions. They auto-scale effortlessly, but suffer from cold start latency on the first request.

---

### 4.3 Step-by-Step Architectural Evolution (Whiteboard Script)

#### Phase 1: The Monolith MVP
- **Whiteboard**: `Client (Monaco)` ──► `API Server` ──► `PostgreSQL Database`.
- **Explanation**: *"We start with a simple monolith. The client submits code, and the server runs it and saves the result."*
- **Bottlenecks**: Running code on the web server risks security attacks and crashes under infinite loops.

#### Phase 2: Isolated Execution via Containers
- **Whiteboard**: Add **Docker Container Sandboxes** separated from the API Server.
- **Explanation**: *"To protect the web server, code is executed inside isolated Docker containers with strict memory and CPU caps."*

#### Phase 3: Message Queues for 100,000 Contestants
- **Whiteboard**: Insert a **Message Queue (Kafka / RabbitMQ)** between the API Server and the Judge Workers.
- **Explanation**: *"During contests, 10,000 users submit at the exact same minute (requiring 27 hours of CPU compute). The API returns `HTTP 202 Accepted` immediately, and workers process submissions at a steady, sustainable pace."*

#### Phase 4: Progressive Polling for Results
- **Whiteboard**: Client polls `GET /submissions/:id/check` every 1–2 seconds.
- **Explanation**: *"Instead of complex WebSockets, progressive short polling is stateless, fault-tolerant, and perfectly suited for 2-to-5 second execution windows."*

#### Phase 5: Fast Contest Leaderboard via Redis Sorted Sets
- **Whiteboard**: Add a **Redis Sorted Set (`ZSET`)** for live contest rankings.
- **Explanation**: *"Instead of running slow SQL `GROUP BY` queries on the submissions table, we store scores in Redis, serving the top 100 ranks in $< 1\text{ms}$."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: Sandbox Security in Simple Terms (The 5 Rules)

How do we safely execute untrusted user code without complex kernel jargon? Explain these 5 straightforward rules:

1. **Read-Only Filesystem**:
   - The container's main filesystem is mounted strictly read-only.
   - Code cannot overwrite system files or install software. It can only write to a temporary folder that is completely deleted right after execution.
2. **Strict CPU and Memory Limits**:
   - The container is capped at **1 CPU core** and **256 MB of RAM**.
   - If the code attempts to allocate 1 GB of memory, the system immediately terminates the code and marks it as **Memory Limit Exceeded (`MLE`)**.
3. **Execution Timeout Watchdog (Anti-Infinite Loop)**:
   - A timer is set for **2.0 seconds**.
   - If a user submits `while True: pass`, the watchdog kills the process after 2 seconds and marks it as **Time Limit Exceeded (`TLE`)**.
4. **Block Network Access**:
   - The container is disconnected from the network (no internet access).
   - User code cannot make HTTP requests, download external files, connect to databases, or launch DDoS attacks.
5. **Block Dangerous System Calls (Anti-Fork Bomb)**:
   - User code is blocked from creating background processes (e.g. `:(){ :|:& };:` fork bombs) or spawning terminal shell commands (`exec`).

---

### 5.2 Deep Dive 2: Fast Competition Leaderboard (SQL vs. Redis)

How do we rank 100,000 competitors in real time with penalty times?

#### The SQL Approach (Why it Fails):
```sql
SELECT user_id, 
       COUNT(DISTINCT problem_id) AS solved_count, 
       MIN(created_at) AS last_solve_time 
FROM submissions 
WHERE competition_id = 'contest_410' AND status = 'ACCEPTED' 
GROUP BY user_id 
ORDER BY solved_count DESC, last_solve_time ASC;
```
- Running this aggregation query every few seconds for 100,000 competitors viewing the leaderboard will lock up the database and bring the site down.

#### The Production Solution: Redis Sorted Sets (`ZSET`)
- Maintain a single Redis Sorted Set for the contest: `leaderboard:contest_410`.
- **Score Calculation**:
  - More solved problems wins; ties broken by earliest finish time.
  - We combine both into a single numeric score:
    $$\text{Score} = (\text{Solved Count} \times 10^{10}) + (10^{10} - \text{Total Penalty Seconds})$$
- **On Accepted Submission**:
  ```text
  ZADD leaderboard:contest_410 <computed_score> "usr_tourist"
  ```
- **Fetching the Top 100 Leaderboard**:
  ```text
  ZREVRANGE leaderboard:contest_410 0 99 WITHSCORES
  ```
  Returns the top 100 ranked competitors from memory in **$< 1\text{ millisecond}$**!
- **Progressive Client Polling**:
  - Competitors poll the leaderboard every 15 seconds during normal contest time, and every 3 seconds in the exciting final 5 minutes of the contest.

---

### 5.3 Deep Dive 3: Language Test Harness & Test Case Serialization

How do we test solutions in Python, C++, Java, and Go using a single set of test cases?

```
┌────────────────────────────────────────────────────────┐
│               Generic JSON Test Cases                  │
│       [ { "input": [2, 7, 11, 15], "target": 9 },      │
│         { "output": [0, 1] } ]                         │
└────────────────────────────────────────────────────────┘
                           │
                           ▼ (Fed to runner)
┌────────────────────────────────────────────────────────┐
│             Language-Specific Test Runner              │
│       (Pre-built wrapper for Python, C++, Java)        │
├────────────────────────────────────────────────────────┤
│ 1. Reads JSON input and converts into native types     │
│    (e.g., C++ vector<int>, Java ArrayList<Integer>)    │
│ 2. Calls user's function: Solution().twoSum(...)       │
│ 3. Compares return value with expected JSON output     │
│ 4. Reports verdict (ACCEPTED / WRONG_ANSWER)           │
└────────────────────────────────────────────────────────┘
```

1. **Store Once in JSON**: All 100 test cases are stored in simple JSON format.
2. **Language Test Runner**: Each language runtime has a lightweight runner script:
   - It reads the JSON input and converts it into native variables (e.g., converting a JSON array into a C++ `std::vector` or Python `list`).
   - It calls the user's function.
   - It compares the returned output with the expected output (handling custom data structures like Trees and Linked Lists).
3. **Report Verdict**: If all 100 test cases match, the runner returns `ACCEPTED`.

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Execution Sandboxing**| Pre-warmed Docker Containers | Running directly on API Server | Running on the server risks total crash and security takeover. Containers boot in $<100\text{ms}$ with strong isolation. |
| **Sandbox Security** | 5 Rules: Read-only fs, CPU/RAM caps, Timeout, No Network | Full Virtual Machines (VMs) | VMs take 10–30s to boot and waste memory. Containers provide lightweight safety boundaries. |
| **Contest Concurrency** | Language Message Queues + Worker Pools | Synchronous HTTP Evaluation | Contests generate 27 hours of compute. Message queues buffer requests safely so workers never drop submissions. |
| **Result Retrieval** | Progressive Short Polling (`GET /check/:id`) | Persistent WebSockets | WebSockets waste massive connection memory for 100k users. Short polling is simple, stateless, and handles 2–5s waits easily. |
| **Contest Leaderboard** | Redis Sorted Set (`ZSET`) | SQL `GROUP BY` Aggregations | SQL aggregates freeze the database under 100k users. Redis `ZSET` serves top rankings in $< 1\text{ms}$. |
| **Problem Storage** | NoSQL Document DB (DynamoDB / MongoDB) | Complex Relational Tables (3NF) | Problems are self-contained documents with nested language stubs and test cases; NoSQL avoids multi-table joins. |

---

### Top 4 Interview Follow-Up Questions (Rapid-Fire Defenses)

### Q1: Why use containers instead of virtual machines for executing code?
- **Answer**: Virtual machines take 10 to 30 seconds to boot up and consume significant memory per instance. Containers boot in under 100 milliseconds and allow us to reuse pre-warmed language runtimes, easily meeting the $< 5\text{ second}$ submission turnaround SLA.

### Q2: What happens if an infinite loop is submitted?
- **Answer**: The worker runs the container under a strict **2.0-second watchdog timer**. If the process has not finished when the timer expires, the worker forcefully kills the process and marks the submission as **Time Limit Exceeded (`TLE`)**.

### Q3: Why is polling better than WebSockets for submission results?
- **Answer**: A code submission only takes between 1 and 5 seconds to run. Maintaining 100,000 persistent open WebSocket connections wastes enormous server connection memory. Having the client poll `GET /check/:id` once per second for 3–5 attempts is stateless, simple, and naturally handles network reconnections.

### Q4: How do you handle custom data structures like `TreeNode` or `ListNode` in test cases?
- **Answer**: The language test runner includes deserialization helpers that convert array representations (e.g. `[1, 2, 3, null, 4]`) into actual linked nodes in memory before calling the user's function, and serializes the returned root node back into an array to verify correctness.
