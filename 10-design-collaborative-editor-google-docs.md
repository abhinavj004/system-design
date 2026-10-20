# System Design 10: Design Real-Time Collaborative Text Editor (Google Docs / Notion)

> **Source**: [Interview with Bunny - Episode 10](https://www.youtube.com/watch?v=-eMtcFqj8vI)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/10](https://www.interviewwithbunny.com/systemdesign/10)  
> **Topic**: Designing a Highly Scalable, Real-Time Concurrent Document Editor (Google Docs / Notion / Figma / Etherpad)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Context & Problem Scope
Collaborative text editing requires allowing multiple remote users to concurrently view, type, format, and delete text within the exact same document canvas in real time. The core algorithmic challenge is achieving **deterministic convergence** across all distributed client screens without character clobbering, race conditions, or cursor jumping.

```
[ Alice (Browser) ]                              [ Bob (Browser) ]
        │                                                │
        │ 1. Insert 'A' at pos 0                         │ 1. Insert 'D' at pos 2
        ▼                                                ▼
┌──────────────────┐                            ┌──────────────────┐
│ Client OT Engine │                            │ Client OT Engine │
└────────┬─────────┘                            └────────┬─────────┘
         │                                               │
         │ (WebSocket op)                                │ (WebSocket op)
         ▼                                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Centralized Server OT Engine                   │
│         Transforms concurrent operations into ordered stream     │
└─────────────────────────────────┬────────────────────────────────┘
                                  │
                                  ▼
                   ┌─────────────────────────────┐
                   │ Redis Canonical Memory State│
                   │ Converged String: "ABCD"    │
                   └─────────────────────────────┘
```

---

### 1.2 Functional Requirements (FR)
1. **Document Lifecycle CRUD**: Users can create, rename, view, and delete documents.
2. **Concurrent Multi-User Collaboration**: Multiple authenticated users can edit the same document simultaneously with sub-second visual propagation.
3. **Real-Time Convergence & Conflict Resolution**: All active participants must converge on the exact same document state regardless of network packet reordering or typing concurrency.
4. **User Presence & Cursor Awareness**: Live visual indicators displaying remote collaborators' cursor positions, selection highlights, and user avatar colors.
5. **Revision History & Version Rollback**: Automated snapshots capturing document states over time, allowing users to browse past versions and restore historical revisions.

---

### 1.3 Non-Functional Requirements (NFR)
1. **Ultra-Low Keystroke Latency**: Keystroke dispatch, transformation, and round-trip broadcast must complete within **$< 100\text{ms}$** to maintain an instantaneous typing feel.
2. **High Availability with Local Consistency (CAP Strategy)**:
   - **General System (AP)**: Browsing document lists, viewing metadata, and reading historical snapshots prioritizes high availability with eventual consistency.
   - **Active Editing Session (CP)**: During an active multi-user editing session on a single document, **consistency is paramount**. A centralized sequencer guarantees that all editors observe identical transformed character orders.
3. **Massive Scale**: Support **$100\text{ Million}$ active users** and billions of documents.
4. **Data Durability**: Zero data loss. Every keystroke operation must be durably appended to write-ahead event streams before eviction from memory.
5. **Bandwidth Efficiency**: Transmit tiny atomic delta operations (characters and offsets) rather than transferring full document blobs across the network.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Scale & Throughput
- **Total Registered Users**: $1\text{ Billion}$ users.
- **Daily Active Users (DAU)**: $100\text{ Million}$ DAU.
- **Active Concurrent Editors at Peak**: $10\text{ Million}$ concurrent editing sessions.
- **Keystroke Frequency**: Average user types $180\text{ characters per minute} = 3\text{ keystrokes/sec}$.
- **Peak Operation Ingestion QPS**:
  $$\text{Operation QPS} = 10,000,000 \times 3 \approx 30,000,000\text{ operations/sec}$$

#### Payload & Storage Math
- **Delta Operation Payload Size**:
  - `doc_id`: 16 bytes
  - `user_id`: 16 bytes
  - `op_type` (INSERT/DELETE): 1 byte
  - `position`: 4 bytes
  - `char/text`: 2 bytes
  - `client_version`: 4 bytes
  - Total per Operation $\approx 50\text{ bytes}$.
- **Ingress Bandwidth for Live Ops**:
  $$\text{Ingress} = 30,000,000\text{ ops/sec} \times 50\text{ bytes} \approx 1.5\text{ GB/sec} \approx 12\text{ Gbps}$$
- **Full Snapshot Storage (Amazon S3)**:
  - Average document length: $10\text{ KB}$ (plain text / markdown / delta JSON).
  - $1\text{ Billion}$ documents: $1\text{B} \times 10\text{ KB} \approx 10\text{ TB}$.
  - Storing 50 historical snapshots per document over time: $10\text{ TB} \times 50 \approx 500\text{ TB}$ (easily managed by S3 object storage).

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|        User        |        |      Document      |        |  DocumentVersion   |
+--------------------+        +--------------------+        +--------------------+
| id (UUID, PK)      |        | id (UUID, PK)      | 1    * | id (BIGSERIAL, PK) |
| name (VARCHAR)     |        | title (VARCHAR)    |<------>| doc_id (UUID, FK)  |
| email (VARCHAR)    |        | owner_id (UUID, FK)|        | version_num (INT)  |
| avatar_url (VAR)   |        | current_ver (INT)  |        | snapshot_url (VAR) |
+--------------------+        | current_url (VAR)  |        | created_by (UUID)  |
                              | created_at (TIME)  |        | created_at (TIME)  |
                              | updated_at (TIME)  |        +--------------------+
                              +--------------------+
                                        │
                                        │ 1
                                        │
                                        │ *
                              +--------------------+
                              |  DocumentOperation | (In Operation DB)
                              +--------------------+
                              | op_id (BIGSERIAL)  |
                              | doc_id (UUID, FK)  |
                              | user_id (UUID, FK) |
                              | op_type (INSERT/DEL|
                              | position (INT)     |
                              | content (TEXT)     |
                              | server_version(INT)|
                              | timestamp (TIME)   |
                              +--------------------+
```

---

## 3. Step 3: API Design (Public REST & WebSocket Contracts)

### 3.1 Document Metadata CRUD (REST)

#### 1. Create Document
```http
POST /api/v1/documents
Authorization: Bearer <jwt>
Content-Type: application/json

Request Body:
{
  "title": "Distributed Systems Architecture",
  "folder_id": "f-99120"
}

Response: 201 Created
{
  "document_id": "doc-8812-uuid",
  "title": "Distributed Systems Architecture",
  "current_version": 1,
  "created_at": "2026-10-18T12:00:00Z"
}
```

#### 2. Get Document Snapshot (Read-Only / Initial Load)
```http
GET /api/v1/documents/doc-8812-uuid
Authorization: Bearer <jwt>

Response: 200 OK
{
  "document_id": "doc-8812-uuid",
  "title": "Distributed Systems Architecture",
  "current_version": 142,
  "snapshot_url": "https://cdn.googledocs.com/snapshots/doc-8812-uuid/v142.json",
  "content": "A distributed system consists of autonomous computing nodes..."
}
```

---

### 3.2 Real-Time Collaborative Session Handshake & Framing (WebSocket)

#### 1. WebSocket Upgrade Request
```http
GET /ws/v1/documents/doc-8812-uuid/edit HTTP/1.1
Host: docs.google.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Version: 13
Authorization: Bearer <jwt>
```

#### 2. Client Ingress Operation Frame (`OP_SUBMIT`)
```json
{
  "event": "OP_SUBMIT",
  "doc_id": "doc-8812-uuid",
  "user_id": "u-alice-12",
  "client_version": 142,
  "op": {
    "type": "INSERT",
    "position": 58,
    "text": "fault-tolerant "
  },
  "timestamp": 1729254000100
}
```

#### 3. Server Broadcast Transformed Frame (`OP_BROADCAST`)
```json
{
  "event": "OP_BROADCAST",
  "doc_id": "doc-8812-uuid",
  "origin_user_id": "u-alice-12",
  "server_version": 143,
  "transformed_op": {
    "type": "INSERT",
    "position": 62,
    "text": "fault-tolerant "
  },
  "timestamp": 1729254000130
}
```

#### 4. Live Cursor & Presence Frame (`CURSOR_PING`)
```json
{
  "event": "CURSOR_UPDATE",
  "doc_id": "doc-8812-uuid",
  "user_id": "u-bob-99",
  "user_name": "Bob",
  "user_color": "#FF5733",
  "cursor_position": 184,
  "selection_range": { "start": 180, "end": 184 }
}
```

---

### 3.3 Historical Revisions & Snapshots (REST)
```http
GET /api/v1/documents/doc-8812-uuid/versions
Authorization: Bearer <jwt>

Response: 200 OK
{
  "versions": [
    {
      "version_id": 142,
      "created_at": "2026-10-18T11:45:00Z",
      "created_by": "u-alice-12",
      "snapshot_url": "https://cdn.googledocs.com/snapshots/doc-8812-uuid/v142.json"
    },
    {
      "version_id": 100,
      "created_at": "2026-10-18T10:00:00Z",
      "created_by": "u-bob-99",
      "snapshot_url": "https://cdn.googledocs.com/snapshots/doc-8812-uuid/v100.json"
    }
  ]
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Collaborators / Web Browsers ]
         │
         ├─────────────────────────────────────────────┐
         │ (HTTP REST: CRUD, History, CDN Snapshots)   │ (Duplex WebSockets: Live Typing)
         ▼                                             ▼
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│     REST API Gateway / Envoy    │       │   Dedicated WebSocket Gateway   │
│  (Auth, Rate Limiting, Routers) │       │ (TLS Offload, Sticky WS Proxy)  │
└─────────────────────────────────┘       └─────────────────────────────────┘
         │               │                                 │
         │               ▼                                 ▼
         │      ┌──────────────────┐              ┌──────────────────┐
         │      │  CDN Cache Edge  │              │ Document Editor  │
         │      │ (CloudFront S3)  │              │  Service (Pods)  │
         │      └────────┬─────────┘              └────────┬─────────┘
         │               │                                 │
         ▼               ▼                                 ▼
┌──────────────────┐ ┌─────────────┐             ┌────────────────────┐
│ Document Metadata│ │ Blob Store  │             │ Redis Cluster      │
│     Service      │ │ (Amazon S3) │             │ • Canonical Canvas │
└────────┬─────────┘ └──────┬──────┘             │ • Active Cursors   │
         │                  │                    └────────────────────┘
         ▼                  │                              │
┌──────────────────┐        │                              │ (Stream Every Keystroke)
│ Cassandra / PG   │        │                              ▼
│ Metadata DB      │        │                    ┌────────────────────┐
└──────────────────┘        │                    │ Kafka Op Queue     │
                            │                    └─────────┬──────────┘
                            │                              │
                            │                              ▼
                            │                    ┌────────────────────┐
                            │                    │ Operation Consumer │
                            │                    └─────────┬──────────┘
                            │                              │
                            │                              ▼
                            │                    ┌────────────────────┐
                            │                    │ Operation DB (Log) │
                            │                    └────────────────────┘
                            │                              ▲
                            ▼                              │ (Reconcile & Prune)
         ┌─────────────────────────────────────────────────┴──────────┐
         │        Background Session Reconciliation Worker            │
         │    (Compacts Op Log + S3 Baseline -> New Major Version)    │
         └────────────────────────────────────────────────────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: Evolution of Concurrent Editing Algorithms (Bunny's 3 Approaches)

Bunny explains why conventional distributed engineering designs fail when applied to real-time text canvases:

#### Approach 1: Full File Replacement
- Every time a user presses a key, the entire document file is submitted to the backend via HTTP `POST`, and downstream clients reload the document.
- **The Failure**: Catastrophic bandwidth explosion. A 100-page document ($\approx 500\text{ KB}$) edited by 10 users typing simultaneously generates thousands of full-file uploads per minute, thrashing network interfaces and crashing browser DOM rendering engines.

#### Approach 2: Database Locking Protocol
- **Pessimistic Locking**: When User A places a cursor on the document or paragraph, the server locks the file. User B is forced to wait until User A releases the lock.
- **Optimistic Locking (Git Merge Model)**: Both users make local edits. When User B saves, if User A already pushed an edit, User B is presented with a merge conflict dialog.
- **The Failure**: Completely violates the core functional requirement of **collaborative concurrent editing**. Google Docs requires seamless zero-friction simultaneous typing.

#### Approach 3: Atomic Delta Transmission + Concurrency Transformation (The Production Solution)
- Clients transmit **only the micro-delta event** (e.g., `INSERT 'R' at position 29`).
- The server and clients use specialized concurrency mathematics to resolve conflicts automatically without locks.

---

### 5.2 Deep Dive 2: Operational Transformation (OT) In-Depth (Bunny's Core Focus)

Google Docs was built from the ground up using **Operational Transformation (OT)** (originating from the 1989 Ellis & Gibbs algorithm, modernized by Google Wave).

#### The Classic Divergence Trap (Alice & Bob)
Imagine a document containing the initial string **`"BC"`**:
- **Alice** inserts `'A'` at position `0` $\to$ Local state: **`"ABC"`**.
- **Bob** concurrently inserts `'D'` at position `2` $\to$ Local state: **`"BCD"`**.

```
Initial Document: "BC" (Length: 2, Indices: 0, 1)

Alice Action: Op_A = Insert('A', 0) -> Canvas: "ABC"
Bob Action:   Op_B = Insert('D', 2) -> Canvas: "BCD"
```

#### What happens under Naive (Untransformed) Broadcast?
1. Alice receives Bob's raw command: `Insert('D', 2)`.
   - Alice inserts `'D'` at position 2 of `"ABC"` $\to$ **`"ABDC"`**.
2. Bob receives Alice's raw command: `Insert('A', 0)`.
   - Bob inserts `'A'` at position 0 of `"BCD"` $\to$ **`"ABCD"`**.
3. **The Disaster**: **State Divergence!** Alice sees `"ABDC"`, while Bob sees `"ABCD"`. The documents no longer match!

#### How Operational Transformation Fixes It:
The OT transformation engine $T(Op_1, Op_2)$ rewrites the operation parameters to account for the positional shifts introduced by preceding operations:

$$T(Op_B, Op_A) \implies Op_B' = \text{Insert}('D', 2 + 1) = \text{Insert}('D', 3)$$

```
Server Transformation Logic:
1. Server receives Op_A (Insert 'A', 0) first -> Assigns Server Version 1.
2. Server updates Canonical State: "ABC".
3. Server receives Op_B (Insert 'D', 2, based on Version 0).
4. Server realizes Op_A was executed at an index <= Bob's index (0 <= 2).
5. Server transforms Bob's operation: Shifts position by +1 length.
   Transformed Op_B' = Insert('D', 3).
6. Server broadcasts Op_B' to Alice.
   Alice executes Insert('D', 3) on "ABC" -> "ABCD".
7. Bob executes Op_A (Insert 'A', 0) on "BCD" -> "ABCD".
```
**Result**: Both Alice and Bob deterministically converge on **`"ABCD"`**!

#### Bunny's Key Rule for OT:
> *"Operational Transformation **requires a centralized sequencer (the Document Editor Server)**. All concurrent operations must serialize through a single authoritative timeline on the server to establish an unambiguous global sequence order."*

---

### 5.3 Deep Dive 3: Operational Transformation (OT) vs. CRDTs (Conflict-Free Replicated Data Types)

While Bunny highlights that Google Docs relies on OT, modern collaborative tools like **Figma, Notion, Apple Notes, and Zed** frequently utilize **CRDTs**.

```
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│  Operational Transformation(OT) │       │ Conflict-Free Replicated (CRDT) │
├─────────────────────────────────┤       ├─────────────────────────────────┤
│ • Centralized server sequencer  │       │ • Decentralized / Peer-to-Peer  │
│ • State-based positional shifts │       │ • Fractional/Immutable IDs      │
│ • Minimal client memory overhead│       │ • Higher memory overhead        │
│ • Example: Google Docs, Etherpad│       │ • Example: Figma, Notion, Yjs   │
└─────────────────────────────────┘       └─────────────────────────────────┘
```

#### How CRDT (Fractional Indexing) Works:
Instead of integer indices (`0, 1, 2`), CRDTs assign **globally unique fractional identifiers** between characters:
- `"B"` has ID `0.5`
- `"C"` has ID `0.75`
- Alice inserts `'A'` before `'B'` $\to$ Assigns fractional ID **`0.25`**.
- Bob inserts `'D'` after `'C'` $\to$ Assigns fractional ID **`0.875`**.
- Merging simply requires sorting characters by their immutable fractional identifiers:
  $$[0.25: 'A'] \to [0.5: 'B'] \to [0.75: 'C'] \to [0.875: 'D'] \implies \text{"ABCD"}$$
- **Trade-off**: CRDTs require zero centralized transformation server, making them ideal for offline-first P2P syncing, but fractional ID metadata significantly expands document memory footprint.

---

### 5.4 Deep Dive 4: Dual-Tier Storage Architecture & The Canonical In-Memory Canvas

To achieve sub-100ms latency, the system never performs database disk I/O inside the critical typing loop.

```
[ Active WebSocket Pod ]
         │
         │ (Sub-millisecond Read/Write)
         ▼
┌───────────────────────────────────────────────┐
│         Redis Cluster (In-Memory Canvas)      │
│ Key: doc:canonical:doc-8812-uuid              │
│ State: "The complete live text of the file..." │
│ TTL: 30 minutes of inactivity                 │
└───────────────────────────────────────────────┘
         │
         │ (Asynchronous Kafka Event Stream)
         ▼
┌───────────────────────────────────────────────┐
│     Cassandra / PostgreSQL Operation Log      │
│ Table: document_operations                    │
│ Records: Every single keystroke append log    │
└───────────────────────────────────────────────┘
         │
         │ (Autosave every 10-20 seconds)
         ▼
┌───────────────────────────────────────────────┐
│       Amazon S3 Compressed Blob Storage       │
│ Path: /snapshots/doc-8812-uuid/v143.json.gz   │
└───────────────────────────────────────────────┘
```

1. **Redis Canonical State**: The authoritative working document canvas is loaded into Redis RAM when the first collaborator joins. Every incoming transformed operation mutates the in-memory Redis string instantly.
2. **Kafka Asynchronous Write-Ahead Log**: Every keystroke delta is pushed to Kafka and batch-inserted into Cassandra/Postgres to preserve an immutable audit trail for rollback.
3. **S3 Periodic Snapshots**: High-level compressed document snapshots are stored on Amazon S3.

---

### 5.5 Deep Dive 5: The Autosave, Versioning & Background Session Reconciliation Engine

If every keystroke generated a permanent document version in S3, storage would balloon to petabytes within weeks. Bunny details the **Two-Tier Versioning and Session Reconciliation Pipeline**:

```
Live Session Typing (Keystrokes: v10.0.1, v10.0.2, ... v10.0.1000)
                             │
                             ▼ (Autosave every 10s: Temporary Minor Snapshots in S3)
                [ Temporary Minor Versions ]
                             │
                             ▼ (Session Ends: All users disconnect / 15m idle)
            ┌──────────────────────────────────┐
            │ Reconciliation Background Worker │
            └────────────────┬─────────────────┘
                             │
     ┌───────────────────────┴───────────────────────┐
     ▼                                               ▼
1. Fetch Base S3 Version (v10.0)             3. Purge Minor Intermediate Blobs
2. Replay & Compact all Ops from Log         4. Delete Session Keystroke Rows
     │                                       5. Persist Official Major Version (v11.0)
     ▼                                          into S3 + Update Metadata DB
[ Consolidated S3 Major Version v11.0 ]
```

#### The Reconciliation Sequence:
1. **Minor Snapshots**: While editing is active, an internal cron executes an **autosave every 10–20 seconds**, saving ephemeral checkpoints (`v10.0.1`, `v10.0.2`) to S3 to guarantee crash-resilience.
2. **Session Termination**: When all collaborators disconnect or remain idle for 15 minutes:
   - The **Reconciliation Worker** wakes up.
   - It reads the baseline major version (`v10.0`) from S3 and pulls the contiguous batch of operational deltas from the `document_operations` table.
   - It compacts and flattens all operations into a pristine target snapshot.
   - It writes the clean, official major version **`v11.0`** into Amazon S3.
   - It updates `documents.current_version = 11.0` in PostgreSQL.
   - **Garbage Collection**: It deletes all temporary minor S3 blobs (`v10.0.1` through `v10.0.99`) and purges the processed micro-operations from the database, preventing storage bloat.

---

### 5.6 Deep Dive 6: Cursor Positioning & Ephemeral Presence

Seeing remote cursors move in real time requires high-frequency transmission with zero database persistence:

1. As a collaborator moves their cursor or highlights text, the browser emits a lightweight WebSocket packet:
   ```json
   { "event": "CURSOR", "pos": 240, "sel_start": 230, "sel_end": 240 }
   ```
2. The WebSocket Gateway broadcasts the cursor coordinates directly to all other peers connected to the same document room.
3. The server writes the cursor coordinates into Redis with a **$10\text{-second TTL}$**:
   ```bash
   SET presence:doc-8812:u-bob-99 '{"pos":240,"color":"#FF5733"}' EX 10
   ```
4. If Bob suddenly loses internet connectivity or closes his laptop lid, his Redis presence key expires in 10 seconds, and the front-end client automatically fades out his cursor.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How does Operational Transformation (OT) work to resolve conflicts in concurrent editing?

OT transforms concurrent operations to account for each other's effects: Scenario: Document 'hello', version 10. User A: insert('!', 5) → 'hello!', User B: delete

- **→ 'hllo'. Server receives A first (version 11), then B (version 12). When B's operation arrives, server sees B's client_version=10 but current server_version=11 (A's op applied). Transformation needed**:
  B intended delete at position 1 in 'hello', but A inserted at position 5 (after), so delete position unchanged. Transform B's op: delete

- **still valid. Apply to 'hello!' → 'hllo!'. Broadcast**:
  

- **Point 1**: Send A's insert to B,

- **Send B's delete to A. Both clients apply operations in server order → converge to 'hllo!'. Complex case**:
  If A inserted at position 0 instead: insert('X', 0) → 'Xhello'. B's delete

- **needs transformation**:
  original target was 'h' at position 1, but now 'h' shifted to position 2 (X inserted before). Transformed op: delete

- **. Apply to 'Xhello' → 'Xello'. Key rules**:
  

- **Point 1**: Insert before target position → shift target right (+1),

- **Point 2**: Delete before target position → shift target left (-1),

- **Operations applied in server-assigned order ensures all clients converge. Why server sequencing?**:
  Without central ordering, Client A and B might apply operations in different orders → divergent states. Server acts as single source of truth for operation sequence. Edge cases:

- **Point 1**: Concurrent inserts at same position → server orders arbitrarily (e.g., by timestamp or user_id),

- **Overlapping deletes → first delete wins, second becomes no-op or shifts position. Production**:
  Google Docs uses OT with central server handling millions of concurrent operations per second, operation log persisted for replay and versioning.

---

### Q2: What are CRDTs and how do they differ from Operational Transformation?

CRDT (Conflict-Free Replicated Data Types) are data structures designed so operations commute (can be applied in any order and converge to same state): Character-based CRDT example: Each character has unique ID = {site_id, sequence_number}. Document 'hello': 'h'={1,1}, 'e'={1,2}, 'l'={1,3}, 'l'={1,4}, 'o'={1,5}. User A offline: insert 'X' after 'h' → creates {1,6} with position 'after {1,1}'. User B offline: delete {1,1} (deletes 'h'). When A and B sync:

- **A applies B's delete**:
  removes 'h'={1,1}, document is 'ello' with X={1,6} positioned 'after {1,1}' → X goes at beginning → 'Xello',

- **B applies A's insert**:
  adds X={1,6} 'after {1,1}' but {1,1} deleted → deterministic rule: insert at position of deleted anchor → 'Xello'. Both converge to 'Xello' regardless of operation order. Key properties:

- **Commutativity**:
  op1(op2(doc)) = op2(op1(doc)),

- **Associativity**:
  (op1 ∘ op2) ∘ op3 = op1 ∘ (op2 ∘ op3),

- **Idempotency**:
  applying same operation twice has no additional effect. CRDT advantages:

- **Point 1**: No central server needed (peer-to-peer),

- **Point 2**: Works offline (store operations locally, sync later),

- **Point 3**: Multi-master replication (multiple servers can accept writes),

- **Simpler conflict resolution (operations designed to commute). CRDT disadvantages**:
  

- **Larger memory**:
  each character has metadata (site_id, sequence), document 'hello' = 5 chars but CRDT structure = 5 objects with IDs + position pointers,

- **Tombstones**:
  deleted characters leave tombstones (invisible markers) that accumulate over time, require garbage collection,

- **Complex data structure**:
  CRDT typically uses tree or linked list (e.g., RGA - Replicated Growable Array) instead of simple string. OT vs CRDT comparison: OT: Central server sequences operations, simpler data structure (just text), smaller memory, complex transformation functions, single point of failure. CRDT: Decentralized (no central server), complex data structure (IDs + tombstones), larger memory, simpler operations (no transformation), works offline. Production usage: Google Docs uses OT (centralized architecture, millions of users), Figma uses CRDT (offline design collaboration), Notion uses OT (real-time editing with server), Automerge library implements CRDT for distributed apps. Hybrid approach: Some systems use OT for real-time editing (low latency, central server) + CRDT for offline sync (store local changes, merge when online) to get best of both. Example scenario: Collaborative design tool (Figma): User A offline: moves object 10px right, User B offline: moves same object 5px up, CRDT: Both operations commute → final position is (+10px right, +5px up) regardless of merge order. With OT: Would need server to transform one operation based on the other → more complex, requires connectivity.

---

### Q3: How do you handle offline editing and sync when the user reconnects?

Offline editing with sync-on-reconnect ensures no data loss and eventual consistency: Offline flow:

- **Point 1**: User editing document, WebSocket connection drops (network failure),

- **Client detects**:
  WebSocket onerror/onclose event fires,

- **Switch to offline mode**:
  Display banner 'Working offline, changes will sync when online', enable local storage,

- **User continues editing**:
  All operations stored locally: IndexedDB.put('offline_ops', {doc_id, ops: [{op_id, type: 'insert', position: 10, content: 'hello', timestamp}]}), also store last_known_version: 42 (last version received from server),

- **Optimistic UI**:
  Show changes immediately in editor (user sees edits despite offline). Reconnect flow:

- **Network restored, WebSocket reconnects (auto-retry with exponential backoff**:
  1s, 2s, 4s, 8s, max 30s),

- **Client sends sync request**:
  {type: 'sync_request', doc_id, last_known_version: 42, queued_ops_count: 5},

- **Server responds**:
  {type: 'sync_response', server_ops: [{op1, op2, op3}], current_version: 50},

- **Server operations**:
  ops that happened on server while client was offline (from other users),

- **Client reconciliation**:
  (a) Apply server operations first: replay ops 43-50 from server to catch up, (b) Transform local queued operations: use OT to transform local ops against server ops (adjust positions based on server changes), (c) Apply transformed local ops: insert into local document, (d) Send to server: {type: 'operations_batch', ops: [{transformed_op1, transformed_op2}]},

- **Server validates**:
  Check operations don't conflict with latest state, if valid → apply and broadcast to other users, if invalid (rare) → reject and request full resync. Conflict detection: If local op position > document length after applying server ops → conflict, show modal 'Conflicts detected during sync', options: (a) Keep local changes (discard server changes in conflict area), (b) Keep server changes (discard local changes), (c) Manual merge (show diff, let user choose per-change). Example scenario: User A offline: deletes 'hello' at position 0-5, server state: 'hello world', User B (online): inserts '!!!' at position 6, server state: 'hello !!!world', User A reconnects:

- **Server sends**:
  insert('!!!', 6) operation,

- **Client applies**:
  'hello' already deleted locally → transform insert position: 6 → 1 (5 chars deleted before position 6), apply: '!!!world',

- **Client sends**:
  delete(0, 5) operation,

- **Server receives**:
  document is 'hello !!!world', apply delete → '!!!world',

- **All clients converge to '!!!world'. Edge cases**:
  

- **Very long offline period**:
  If offline >24 hours with 1000s of operations, skip OT transformation, fetch latest snapshot from server, display merge UI showing local changes vs server state side-by-side,

- **Document deleted**:
  If document deleted on server while offline, on reconnect show 'Document no longer exists, save local changes as new document?',

- **Permission revoked**:
  If edit permission removed while offline, on reconnect convert operations to 'suggested changes' instead of direct edits. Performance optimization:

- **Compress local operations**:
  merge consecutive inserts ('h' + 'e' + 'l' + 'l' + 'o' → 'hello') before sending,

- **Incremental sync**:
  send operations in batches of 50 rather than all at once to avoid overwhelming server,

- **Background sync**:
  use Service Worker to attempt sync even when browser tab closed. Production: Google Docs supports offline editing via Chrome extension + local storage, syncs when online with OT transformation, shows conflicts if unsolvable automatically, provides version history to rollback if needed.

---

### Q4: How do you implement cursor presence and real-time cursor updates for multiple users?

Cursor presence shows where each user is typing/selecting in real-time: Architecture:

- **Client tracks cursor position**:
  Editor onChange event fires on every cursor move (arrow keys, mouse click, text selection),

- **Throttle updates**:
  Batch cursor position changes every 100ms to avoid flooding WebSocket (user typing fast = 10 chars/sec = 100 cursor updates/sec → throttled to 10 updates/sec),

- **Send via WebSocket**:
  {type: 'cursor', user_id: A, cursor_pos: 15, selection: {start: 10, end: 15}, timestamp},

- **Server receives**:
  Validates session active, extracts user info (name, color from session),

- **Broadcast to others**:
  {type: 'cursor_update', user: {user_id: A, name: 'Alice', color: '#FF5733', cursor_pos: 15, selection: {start: 10, end: 15}}},

- **Other clients render**:
  Colored cursor indicator at position 15 with label 'Alice', selection highlight from 10-15 in semi-transparent color. Cursor positioning: Calculate pixel position from character position:

- **Document text**:
  'Hello\nWorld' (\n = newline),

- **Point 2**: User A cursor at position 8 (in 'World'),

- **Client calculates**:
  Line 1 = 'Hello\n' (6 chars), position 8 = line 2, column 2 (W-o),

- **Measure text**:
  Use canvas measureText() or DOM getBoundingClientRect(), line 2 top = 20px, column 2 left = 15px,

- **Render cursor**:
  <div style={{position: 'absolute', top: 20px, left: 15px, borderLeft: '2px solid #FF5733'}}><label>Alice</label></div>. Selection rendering: Create overlay div with semi-transparent background: {position: 'absolute', top: 20px, left: 10px (start), width: 40px (end - start), height: 20px (line height), background: 'rgba(255,87,51,0.3)'}. Multiple users: Assign each user a unique color on join: colors = ['#FF5733', '#33FF57', '#3357FF', '#FF33F5'], user_color = colors[user_id % colors.length], persist in session: Redis HSET session:{session_id} 'color' {color}, broadcast to all clients on join. Ephemeral data: Cursor positions NOT persisted in database (too frequent updates, not durable data), stored in Redis with short TTL: HSET cursors:{doc_id} user:{user_id} '{cursor_pos: 15, selection: {...}, timestamp}' EX 300 (5 min), auto-expires when user disconnects. User disconnect handling: WebSocket close event → server removes cursor: HDEL cursors:{doc_id} user:{user_id}, broadcast to other clients: {type: 'cursor_removed', user_id: A}, clients remove cursor indicator and label. Cursor transformation (OT): When operations applied, cursor positions must be adjusted: User A cursor at position 10, User B inserts 'hello' at position 5, A's cursor position transforms: 10 → 15 (5 chars inserted before cursor), client automatically adjusts cursor position when receiving B's operation. Selection transformation: User A selected text 10-20, User B deletes chars 5-8 (3 chars before selection), A's selection transforms:

- **start**:
  10 → 7 (3 chars deleted before start),

- **end**:
  20 → 17 (3 chars deleted before end), adjusted selection: 7-17. Mobile optimization: On mobile, cursor updates less frequent (200ms throttle instead of 100ms), smaller cursor indicators (no labels, just dots), tap to see user name tooltip. Performance: 10 concurrent users × 10 cursor updates/sec = 100 WebSocket messages/sec, with throttling → 10 users × 1 update/0.1sec = 10 messages/sec per document (90% reduction), enables scaling to 100+ concurrent editors per document. Example UI: Google Docs shows: colored cursors with user names, selection highlights in user's color, active users list in top-right corner with colors, hover over cursor to see user name if label hidden. Advanced features:

- **Cursor following**:
  Click user in active list → scroll to their cursor position,

- **Cursor hide**:
  Toggle to hide other users' cursors if distracting,

- **Collaborative editing analytics**:
  track how long each user's cursor stays in each section (engagement metrics).

---

### Q5: How do you implement auto-save and periodic snapshots?

Auto-save prevents data loss and snapshots enable fast recovery: Triggering snapshots: Two conditions (whichever comes first):

- **Operation count**:
  Every 50 operations applied to document,

- **Time-based**:
  Every 5 minutes since last snapshot. Background job: Runs every minute, checks all active documents: SELECT doc_id, COUNT(*) as op_count FROM operations WHERE timestamp > (SELECT max(created_at) FROM snapshots WHERE doc_id=operations.doc_id) GROUP BY doc_id HAVING COUNT(*) >= 50. Snapshot creation flow:

- **Reconciliation Service picks up**:
  Triggered by job or Kafka event {type: 'snapshot_needed', doc_id, op_count: 52},

- **Fetch last snapshot**:
  SELECT blob_path, version FROM snapshots WHERE doc_id={doc_id} ORDER BY version DESC LIMIT 1 → gets S3 path docs/{doc_id}/v10.txt,

- **Download snapshot**:
  GET S3 docs/{doc_id}/v10.txt → base content 'Hello world',

- **Fetch operations**:
  SELECT * FROM operations WHERE doc_id={doc_id} AND server_version > 10 ORDER BY server_version ASC → 52 operations,

- **Replay operations**:
  FOR EACH op: apply to document (insert/delete/format) → final content 'Hello world! This is a test.',

- **Validate**:
  Check document integrity (no negative positions, content length matches expected),

- **Compress**:
  gzip compress final content (reduces size by 70%: 1000 chars → 300 bytes),

- **Upload to S3**:
  PUT docs/{doc_id}/v11.txt.gz with compressed content,

- **Update database**:
  BEGIN TRANSACTION; INSERT INTO snapshots (snapshot_id, doc_id, version: 11, blob_path: 's3://...v11.txt.gz', operations_since_last: 52, created_at: now()); UPDATE documents SET latest_snapshot_version=11, updated_at=now(); COMMIT;,

- **Update Redis cache**:
  SET doc:{doc_id}:state {uncompressed_content} EX 1800 (30 min),

- **Publish event**:
  Kafka 'doc.snapshot_created' → {doc_id, version: 11, op_count: 52} for analytics. Incremental snapshots: Instead of full document, store delta: {base_version: 10, operations: [op1, op2, ...52]}, saves storage but slower to reconstruct, full snapshot every 10 versions (versions 10, 20, 30, 40) for faster recovery. Auto-save indication: Client shows:

- **Point 1**: 'Saving...' indicator when operations sent,

- **'Saved' with timestamp when server acknowledges**:
  {type: 'ack', op_id, server_version},

- **Point 3**: 'All changes saved to Drive' when idle for 5 seconds,

- **'Unsaved changes' if WebSocket disconnected (operations queued locally). Fast recovery from snapshot**:
  User opens document:

- **Fetch latest snapshot**:
  GET S3 docs/{doc_id}/v11.txt.gz,

- **Decompress**:
  gunzip → 'Hello world! This is a test.',

- **Fetch recent operations**:
  SELECT * FROM operations WHERE doc_id={doc_id} AND server_version > 11 (only ops after snapshot),

- **Replay**:
  Apply 5 operations (since last snapshot 2 min ago) → current state,

- **Load time**:
  Snapshot fetch (50ms) + decompress (10ms) + replay 5 ops (5ms) = 65ms vs 500ms to replay 1000 ops from beginning. Garbage collection: Cleanup old operations after snapshots:

- **Archive operations**:
  SELECT * FROM operations WHERE doc_id={doc_id} AND server_version <= {snapshot_version - 1000},

- **Store in S3 Glacier**:
  PUT archive/{doc_id}/ops_2026_01.json.gz with compressed ops,

- **Delete from hot DB**:
  DELETE FROM operations WHERE doc_id={doc_id} AND server_version <= {snapshot_version - 1000},

- **Retention**:
  Keep operations for 90 days in hot storage, then archive, keep snapshots forever. Snapshot versioning: Each snapshot is a version: v1 (initial), v2 (after 50 ops), v3 (after 100 ops), enables rollback: user clicks 'Version history' → sees v1, v2, v3 with timestamps, can restore to any version. Concurrent editing during snapshot: Snapshot creation happens in background:

- **Point 1**: Operations continue to be applied during snapshot generation,

- **Point 2**: Snapshot captures state at specific server_version (e.g., version 62),

- **New operations (version 63+) applied after snapshot completes, no downtime. Production numbers**:
  Google Docs generates ~100M snapshots/day across all documents, average snapshot size 50KB (compressed), stored in GCS (Google Cloud Storage) with lifecycle management (hot → cold → archive), enables sub-second document loading for 99% of users.

---

### Q6: How do you handle version history and rollback to previous versions?

Version history enables users to view and restore previous document states: Version creation: Automatic versioning via snapshots:

- **Point 1**: Every 50 operations → new snapshot = new version,

- **Manual versions**:
  user clicks 'Name current version' → creates named snapshot with label (e.g., 'Final draft v1'),

- **Timestamps**:
  each version stores created_at for sorting and display. Fetching version history: User clicks 'Version history' button: GET /v1/api/docs/{doc_id}/versions →

- **Query database**:
  SELECT snapshot_id, version, created_at, operations_since_last, created_by, label FROM snapshots WHERE doc_id={doc_id} ORDER BY version DESC LIMIT 50,

- **Join with users**:
  get creator names,

- **Response**:
  [{version: 11, created_at: '2026-01-22T10:30:00Z', label: null, created_by: 'system', ops_count: 52}, {version: 10, created_at: '10:25:00Z', label: 'Draft 1', created_by: 'Alice', ops_count: 50}],

- **UI displays**:
  Timeline with version numbers, timestamps, creators, operation counts. Viewing specific version: User clicks version 10: GET /v1/api/docs/{doc_id}/version/10 →

- **Fetch snapshot**:
  GET S3 docs/{doc_id}/v10.txt.gz,

- **Decompress and return**:
  {version: 10, content: 'Hello world', metadata: {created_at, created_by}},

- **Display**:
  Read-only preview with banner 'Viewing version 10 from 10:25 AM',

- **Show changes**:
  Diff current version vs selected version (highlight additions in green, deletions in red),

- **Navigation**:
  'Previous version' / 'Next version' buttons to browse history. Restoring version: User clicks 'Restore this version': POST /v1/api/docs/{doc_id}/restore with {target_version: 10} →

- **Validate**:
  Check user has edit permission,

- **Fetch target version content**:
  GET S3 docs/{doc_id}/v10.txt.gz → 'Hello world',

- **Create restore operation**:
  {type: 'replace_all', content: {v10_content}, user_id, timestamp},

- **Append to op-log**:
  INSERT INTO operations (op_id, doc_id, type: 'restore', content, server_version: 63),

- **Create new snapshot**:
  Upload to S3 docs/{doc_id}/v12.txt.gz with v10 content (restored state becomes new version 12),

- **Update database**:
  INSERT INTO snapshots (version: 12, label: 'Restored from v10', created_by: user_id),

- **Notify active editors**:
  Broadcast via WebSocket {type: 'document_restored', from_version: 10, to_version: 12, by_user: 'Bob'},

- **Clients reload**:
  Fetch latest version (v12), discard local unsaved changes (with confirmation prompt). Version comparison: Side-by-side diff: User selects two versions (v8 vs v10) → server computes diff:

- **Point 1**: Fetch both contents from S3,

- **Run diff algorithm (Myers diff)**:
  computes line-by-line changes,

- **Response**:
  [{type: 'equal', text: 'Hello'}, {type: 'delete', text: 'old'}, {type: 'insert', text: 'world'}],

- **UI renders**:
  Side-by-side view with color coding (red=deleted, green=added, white=unchanged),

- **Merge option**:
  'Accept left' / 'Accept right' / 'Accept both' for conflicting changes. Named versions: User can name versions for milestones:

- **Point 1**: Click 'Name current version' → modal prompts for name,

- **POST /v1/api/docs/{doc_id}/versions with {label**:
  'Final draft before review'},

- **Creates snapshot with label**:
  INSERT INTO snapshots (..., label: 'Final draft before review'),

- **Shows in version history with special marker (star icon). Auto-naming**:
  System automatically names versions:

- **Major edits**:
  'Version from 2 hours ago (50 operations)',

- **Daily**:
  'January 22, 2026 10:30 AM',

- **User-triggered**:
  'Saved by Alice'. Retention policy: Keep all versions for:

- **30 days**:
  full access and restore,

- **90 days**:
  view-only (can't restore directly, must copy content),

- **1 year**:
  archived to Glacier (retrieve takes 12 hours),

- **Forever**:
  named versions never expire (user-marked important milestones). Change attribution: Version history shows:

- **Who made changes**:
  SELECT DISTINCT user_id FROM operations WHERE doc_id={doc_id} AND server_version BETWEEN {v8} AND {v10},

- **Change summary**:
  'Alice added 500 words, Bob deleted 200 words, Carol formatted 3 paragraphs',

- **Activity timeline**:
  Graph showing edit activity over time (operations/hour),

- **Detailed changes**:
  Drill down to see individual operations by user. Optimizations:

- **Lazy loading**:
  Load version list (50 most recent), load older versions on scroll (pagination),

- **Thumbnail previews**:
  Generate image thumbnails of document at each version (first page screenshot) for visual navigation,

- **Fast diff**:
  Use precomputed diffs between consecutive versions instead of computing on-demand,

- **Compression**:
  Old versions compressed more aggressively (bzip2 instead of gzip) to save storage. Mobile UX: Simplified version history: show only major versions (daily snapshots), restore requires confirmation with explanation, diff view shows inline changes instead of side-by-side (screen space limited). Production example: Google Docs stores version history for all documents: millions of versions generated per day, enables rollback to any point in time within 30 days, named versions kept forever, change attribution shows exactly who made each edit with timestamps.

---

### Q7: How do you implement the Reconciliation Service and what does it do?

Reconciliation Service ensures data consistency by validating and rebuilding canonical document state from operation log: Purpose:

- **Validate consistency**:
  Ensure operations applied correctly and document state matches expectations,

- **Generate snapshots**:
  Create periodic snapshots for fast recovery,

- **Detect anomalies**:
  Identify corrupted operations or invalid states,

- **Rebuild state**:
  Reconstruct document from op-log if cache/snapshot lost. Architecture: Kafka consumer + background jobs:

- **Kafka consumer**:
  Subscribes to 'doc.operations' topic with consumer group 'reconciliation-service', processes every operation in real-time,

- **Background jobs**:
  Periodic full reconciliation runs nightly (2 AM) to validate all documents. Real-time reconciliation flow:

- **Operation published**:
  Editor Service → Kafka 'doc.operations' topic: {doc_id, op_id, type: 'insert', position: 10, content: 'hello', user_id, server_version: 42, timestamp},

- **Reconciliation Service consumes**:
  Receives operation event from Kafka,

- **Validate operation**:
  (a) Check position valid: position <= document_length (can't insert at position 1000 if doc only 100 chars), (b) Check user permissions: query DB to ensure user_id has edit permission, (c) Check sequence: server_version increments by 1 from previous (no gaps),

- **Update metrics**:
  Increment counters {operations_processed, operations_by_doc, operations_by_user},

- **Check snapshot trigger**:
  SELECT COUNT(*) FROM operations WHERE doc_id={doc_id} AND timestamp > (SELECT max(created_at) FROM snapshots WHERE doc_id={doc_id}), if count >= 50 → trigger snapshot creation. Snapshot generation:

- **Fetch last snapshot**:
  SELECT blob_path, version FROM snapshots WHERE doc_id={doc_id} ORDER BY version DESC LIMIT 1 → S3 path docs/{doc_id}/v10.txt.gz,

- **Download**:
  GET S3 docs/{doc_id}/v10.txt.gz → base content,

- **Fetch operations**:
  SELECT * FROM operations WHERE doc_id={doc_id} AND server_version > 10 ORDER BY server_version ASC → 52 operations since v10,

- **Replay operations**:
  Apply each operation to document: (a) insert(10, 'hello') → insert 'hello' at position 10, (b) delete(15, 3) → delete 3 chars starting at position 15, (c) Validate after each: check position valid, content length matches expected,

- **Compute checksum**:
  SHA256 hash of final content for integrity verification,

- **Upload snapshot**:
  PUT S3 docs/{doc_id}/v11.txt.gz with compressed content + checksum metadata,

- **Update DB**:
  INSERT INTO snapshots (snapshot_id, doc_id, version: 11, blob_path, checksum, operations_since_last: 52, created_at),

- **Update Redis**:
  SET doc:{doc_id}:state {content} EX 1800. Anomaly detection:

- **Invalid position**:
  If operation position > document length → log error, skip operation, flag document for review: INSERT INTO reconciliation_errors (doc_id, op_id, error: 'Invalid position', severity: 'high'),

- **Sequence gap**:
  If server_version jumps (42 → 44, missing 43) → check if operation 43 lost, attempt recovery from Kafka offset,

- **Checksum mismatch**:
  After snapshot creation, recompute checksum from replay vs stored checksum, if mismatch → document state corrupted → alert admin,

- **Performance anomaly**:
  If snapshot generation takes >10 seconds → document too large or operations too complex → flag for optimization. Full reconciliation (nightly):

- **Trigger**:
  Cron job at 2 AM (low traffic time),

- **Batch process**:
  SELECT doc_id FROM documents WHERE updated_at > now() - INTERVAL '7 days' ORDER BY updated_at DESC → active documents,

- **FOR EACH document**:
  (a) Fetch last snapshot, (b) Replay all operations since snapshot, (c) Compute checksum, (d) Compare with Redis cache (if exists), (e) If mismatch: Generate new snapshot, log inconsistency,

- **Report**:
  {total_docs_checked: 10000, snapshots_created: 500, errors_detected: 2, total_time: '45 min'},

- **Alerts**:
  If errors_detected > 10 → PagerDuty alert to on-call engineer. State rebuild: If document state lost (Redis evicted, snapshot corrupted):

- **Service receives request**:
  GET /v1/api/docs/{doc_id},

- **Fetch operations from beginning**:
  SELECT * FROM operations WHERE doc_id={doc_id} ORDER BY server_version ASC,

- **Replay all operations from empty document**:
  start with '', apply each insert/delete sequentially,

- **Generate snapshot**:
  Save current state to S3,

- **Cache in Redis**:
  SET doc:{doc_id}:state {content} EX 1800,

- **Time**:
  Replaying 10K operations takes ~5 seconds (acceptable fallback). Metrics & monitoring:

- **Grafana dashboards**:
  operations/sec, snapshot creation rate, reconciliation errors, op-log replay time,

- **Alerts**:
  if reconciliation_errors > 10/hour → alert, if snapshot_creation_time > 10 sec → alert (doc too large),

- **SLIs**:
  99.9% of operations processed within 1 sec, 99% of snapshots created within 5 sec. Scaling:

- **Kafka consumer parallelism**:
  10 consumer instances process 10 partitions (partition by doc_id hash),

- **Snapshot workers**:
  50 workers in thread pool handle snapshot generation concurrently,

- **Rate limiting**:
  Max 100 snapshots/sec to avoid overloading S3, queue excess for later. Error recovery:

- **Corrupted operation**:
  Skip and log, continue with next, mark document as 'inconsistent' for manual review,

- **Database failure**:
  Kafka retains events for 7 days, replay from offset after recovery,

- **S3 failure**:
  Retry snapshot upload with exponential backoff, temporarily serve from Redis cache only. Production: Google Docs reconciliation processes billions of operations daily, generates millions of snapshots, detects and auto-corrects inconsistencies, ensures 99.99% data integrity, enables disaster recovery (rebuild any document from op-log).

---

### Q8: How would you scale this system to handle 1 million concurrent editors?

Scaling to 1M concurrent editors requires horizontal scaling at every layer: WebSocket Gateway scaling:

- **Capacity**:
  Each instance handles 10K connections, need 1M / 10K = 100 instances,

- **Load balancing**:
  Use TCP load balancer (ELB/ALB) with sticky sessions (IP hash), ensures user's WebSocket stays on same instance for duration,

- **Auto-scaling**:
  Scale based on connection count, target 7K connections/instance (30% headroom),

- **Connection handling**:
  WebSocket upgrade from HTTP, maintain persistent TCP connections, heartbeat every 30 sec to detect disconnects,

- **Memory**:
  10K connections × 10 KB/connection = 100 MB per instance (manageable),

- **Network**:
  10K connections × 10 updates/sec × 500 bytes = 50 MB/sec = 400 Mbps per instance. Document Editor Service scaling:

- **Stateless service**:
  Each instance can process any document (no affinity),

- **Horizontal scaling**:
  200 instances to handle operations from 1M users,

- **Load distribution**:
  Operations partitioned by doc_id hash, Kafka topic has 100 partitions (2 instances per partition for redundancy),

- **Operation processing**:
  Each instance processes 5K ops/sec (1M users / 200 instances, assuming 1 operation per user per second average),

- **Cache hit rate**:
  Redis provides 95% cache hits for document state, only 5% hit DB. Redis scaling:

- **Redis Cluster**:
  50 nodes (master + replicas), each node handles 20K documents,

- **Sharding**:
  Shard by doc_id hash, hash(doc_id) % 50 = shard_id,

- **Memory**:
  1M active documents × 50 KB avg = 50 GB, with replication (3x) = 150 GB total,

- **Throughput**:
  50 nodes × 100K ops/sec = 5M ops/sec capacity (1M users × 10 ops/sec = 10M ops/sec required, so need 100 nodes),

- **Eviction policy**:
  LRU eviction with TTL=30 min for document state, hot documents stay cached, cold documents evicted and refetched from S3. Database (PostgreSQL) scaling:

- **Read replicas**:
  10 replicas for read queries (fetching operations, user permissions, version history),

- **Write primary**:
  1 primary for writes (append operations to op-log),

- **Sharding**:
  Partition operations table by doc_id hash (100 shards), each shard handles 1% of documents,

- **Write throughput**:
  1M users × 1 operation/sec = 1M ops/sec, with batching (10 ops per transaction) = 100K transactions/sec, PostgreSQL handles ~10K txn/sec per shard, need 10 shards minimum,

- **Connection pooling**:
  Each API instance maintains 200 connections, 200 instances × 200 connections = 40K connections (use PgBouncer to pool down to 500 connections per database). Kafka scaling:

- **Cluster**:
  20 brokers with replication factor 3,

- **Partitions**:
  100 partitions for 'doc.operations' topic, enables parallel consumption,

- **Throughput**:
  1M ops/sec × 500 bytes/op = 500 MB/sec write, Kafka handles 1 GB/sec per broker, 20 brokers = 20 GB/sec capacity (plenty of headroom),

- **Retention**:
  7 days retention (enables replay for disaster recovery),

- **Consumers**:
  100 consumer instances (1 per partition) for Reconciliation Service, Notification Service, Analytics Service. S3 scaling:

- **Snapshots**:
  1M active documents × 1 snapshot/5 min = 12M snapshots/hour = 3333 snapshots/sec,

- **S3 write capacity**:
  5500 requests/sec per prefix (first letter of doc_id), use multiple prefixes (a/, b/, c/, ... z/) = 5500 × 26 = 143K writes/sec (sufficient),

- **Read capacity**:
  Document opening = fetch snapshot, 1M users opening docs over 10 min = 100K opens/min = 1666 ops/sec (well within limits),

- **CDN caching**:
  CloudFront caches frequent snapshots (popular documents), 90% cache hit rate reduces S3 reads by 10x. Network bandwidth:

- **WebSocket traffic**:
  1M users × 10 ops/sec × 500 bytes = 5 GB/sec = 40 Gbps,

- **Kafka traffic**:
  500 MB/sec write + 500 MB/sec × 3 consumers = 2 GB/sec = 16 Gbps,

- **S3 traffic**:
  3333 snapshots/sec × 50 KB = 166 MB/sec = 1.3 Gbps,

- **Total**:
  ~60 Gbps, use AWS VPC with 100 Gbps capacity. Cost optimization:

- **Spot instances**:
  Use spot instances for WebSocket Gateway (70% discount), graceful failover on interruption,

- **Reserved instances**:
  Reserve Editor Service instances (1-year commitment for 40% discount),

- **S3 lifecycle**:
  Move old snapshots (>90 days) to Glacier (10x cheaper),

- **Compression**:
  gzip snapshots (70% size reduction), saves S3 storage and transfer costs,

- **Total cost**:
  ~$500K/month for 1M concurrent users (WebSocket $200K + Databases $150K + Kafka $50K + S3 $50K + Network $50K). Monitoring & observability:

- **Metrics**:
  operations/sec, WebSocket connections, cache hit rate, snapshot generation time, error rate,

- **Dashboards**:
  Grafana with real-time graphs per service,

- **Alerts**:
  PagerDuty for critical errors (DB down, Kafka lag >1 min, 5xx errors >1%),

- **Distributed tracing**:
  Jaeger for request tracing across services (WebSocket → Editor → Kafka → DB → S3). Failure handling:

- **WebSocket instance failure**:
  Client auto-reconnects to different instance (exponential backoff), requests sync to catch up on missed operations,

- **Editor Service failure**:
  Kafka retains events, new instance replays from last committed offset,

- **Redis failure**:
  Fallback to fetching snapshots from S3 + replaying recent operations from DB,

- **Database failure**:
  Promote read replica to primary (2-3 min RTO), Kafka retains events for replay after recovery,

- **S3 failure**:
  Serve from Redis cache (covers last 30 min of activity), queue snapshot writes for later. Production examples: Google Docs handles millions of concurrent editors using similar architecture, scales horizontally at every layer, uses custom OT algorithm optimized for millions of operations per document, maintains 99.9% uptime, handles peak loads during major events (e.g., collaborative planning for large conferences).

---

## 7. Interview Summary & Architectural Decision Matrix

| Architectural Dimension | Bunny's Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Concurrency Resolution** | Operational Transformation (OT) | CRDT (Fractional Indexing) | OT maintains minimal client-side memory footprint and fast string arrays; centralized server enforces global sequence. |
| **Transport Protocol** | Full-Duplex WebSockets | HTTP REST / Polling | Sub-millisecond continuous framing; bidirectional delta dispatch and cursor broadcasting without HTTP connection churn. |
| **Active Canvas State** | In-Memory Redis Canonical Cache | Direct PostgreSQL/S3 Disk | Sub-millisecond read/write access enables 30M ops/sec keystroke throughput without saturating disk spindles. |
| **Data Transmission Model**| Micro-Delta Operations | Full Document Replacement | Reduces payload sizes by $99.9\%$ (50 bytes vs 500 KB per keystroke), eliminating network bottlenecks. |
| **Storage Optimization** | 2-Tier Minor/Major Reconciliation | Permanent Snapshot per Save | Consolidates thousands of minor keystrokes into clean major versions, pruning ephemeral snapshots from S3. |
| **Presence / Cursor Tracking**| Ephemeral Redis State with TTL | Relational Database Logging | High-frequency cursor coordinates bypass disk storage completely; auto-expires disconnected users in 10s. |
