# System Design: Google Drive / Dropbox (Distributed Cloud Storage)

> **Source**: [Interview with Bunny - Episode 18](https://youtu.be/Dnp_-xZjNRo)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/18](https://www.interviewwithbunny.com/systemdesign/18)  
> **Topic**: Designing a Distributed Cloud Storage Platform (Google Drive / Dropbox)

---

## 1. Step 1: Requirements Gathering

### 1.1 Functional Requirements
1. **User Onboarding / Authentication**: Users can create an account and log in securely.
2. **File & Folder Operations (Upload / Download)**:
   - Users can upload files, videos, documents, and folders of arbitrary sizes.
   - Users can download any previously uploaded file or directory from any location worldwide.
3. **Automatic Cross-Device Synchronization**:
   - Multiple devices connected to the same account (e.g., iPhone, iPad, Mac/Windows PC) must automatically synchronize changes made to any synced file or folder.
4. **Sharing & Access Control (Permissions)**:
   - Users can share specific files or entire folders with other users (friends, family, teammates) with granular permissions (`READ_ONLY`, `READ_WRITE`).
5. **Directory Management (Full CRUD)**:
   - Hierarchical folder structure support (folders and nested subfolders).
   - Users can create, rename, move, and delete files and directories.
6. **Storage Quota Enforcement**:
   - Users get a free tier quota (e.g., 15 GB).
   - System must strictly enforce quota limits before allowing chunk uploads, with upgrade paths for premium users.

---

### 1.2 Non-Functional Requirements
1. **High Availability**: The service must be operational 24/7 with zero downtime for viewing and downloading files.
2. **Extreme Durability & Reliability (Zero Data Loss)**:
   - Files uploaded must never be lost or corrupted ($99.999999999\%$ / 11 9's durability using distributed block storage replication).
3. **Consistency Trade-offs**:
   - **Strong Consistency for Metadata**: File names, directory trees, permissions, and quota tracking must be strictly consistent (ACID) to prevent corrupted directory hierarchies.
   - **Eventual Consistency for Cross-Device Sync**: A nominal sync lag (milliseconds to a few seconds) across secondary devices is completely acceptable.
4. **Low Latency & Bandwidth Optimization**:
   - Efficient network bandwidth utilization using **chunking**, **block-level deduplication**, and **delta sync** (only uploading modified blocks).

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+       +--------------------+       +--------------------+
|        User        |       |     Directory      |       |    FileMetadata    |
+--------------------+       +--------------------+       +--------------------+
| id (UUID)          | 1   * | id (UUID)          | 1   * | id (UUID)          |
| email (VARCHAR)    |<----->| user_id (UUID)     |<----->| dir_id (UUID)      |
| storage_used (BIG) |       | name (VARCHAR)     |       | name (VARCHAR)     |
| storage_quota (BIG)|       | parent_id (UUID)   |       | size (BIGINT)      |
+--------------------+       +--------------------+       | latest_version_id  |
                                                          +--------------------+
                                                                     | 1
                                                                     | *
+--------------------+       +--------------------+       +--------------------+
|     ChunkBlock     | *   * |  FileVersionChunk  | *   1 |    FileVersion     |
+--------------------+<----->+--------------------+<----->+--------------------+
| chunk_hash (SHA256)|       | version_id (UUID)  |       | id (UUID)          |
| size (INT)         |       | chunk_hash (SHA256)|       | file_id (UUID)     |
| s3_blob_url (TEXT) |       | chunk_index (INT)  |       | version_num (INT)  |
+--------------------+       +--------------------+       | status (COMMITTED) |
                                                          +--------------------+
```

1. **User**: Account credentials, billing tier, storage quota used vs. allocated limit.
2. **Directory / Folder**: Hierarchical node representing folders (`id`, `parent_dir_id`, `user_id`, `path`).
3. **File Metadata**: High-level file details (`id`, `name`, `directory_id`, `total_size`, `mime_type`, `owner_id`).
4. **File Version**: Represents immutable point-in-time snapshots of a file (`version_id`, `file_id`, `version_number`, `status`).
5. **Chunk Block**: Physical block unit (e.g., 4MB) identified by its cryptographic hash (`chunk_hash`, `s3_blob_url`, `size`).
6. **File-Version-Chunk Mapping**: Ordered manifest linking a specific file version to its constituent chunks (`version_id`, `chunk_index`, `chunk_hash`).
7. **Permission / Share**: Access control mapping (`id`, `entity_type: FILE|FOLDER`, `entity_id`, `shared_with_user_id`, `permission_type: VIEW|EDIT`).

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Upload Initiation (Pre-Flight & Quota Check)
```http
POST /api/v1/files/upload/initiate
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "name": "annual_presentation.mp4",
  "total_size": 104857600,   // 100 MB
  "parent_dir_id": "dir_789a-bc12",
  "total_chunks": 25,
  "chunk_hashes": [
    "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb",
    ...
  ]
}

Response: 200 OK
{
  "upload_id": "upl_9812-45af",
  "file_id": "file_4421-9988",
  "recommended_chunk_size": 4194304, // 4MB
  "missing_chunk_hashes": [
    "ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb"
  ],
  "existing_chunk_hashes": [
    "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
  ]
}
```
> **Bunny's Optimization**: The server immediately compares hashes against existing blocks in S3. Chunks that already exist across the system (**Deduplication**) do *not* need to be uploaded!

---

### 3.2 Request Pre-signed S3 Upload URLs
```http
POST /api/v1/files/upload/signed-urls
Authorization: Bearer <jwt_token>

Request Body:
{
  "upload_id": "upl_9812-45af",
  "chunk_hashes": ["ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb"]
}

Response: 200 OK
{
  "urls": [
    {
      "chunk_hash": "ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb",
      "signed_s3_url": "https://s3.amazonaws.com/drive-blobs/ca9781...?signature=xyz&expires=3600"
    }
  ]
}
```

---

### 3.3 Final Commit Upload
```http
POST /api/v1/files/upload/commit
Authorization: Bearer <jwt_token>

Request Body:
{
  "upload_id": "upl_9812-45af",
  "file_id": "file_4421-9988"
}

Response: 201 Created
{
  "file_id": "file_4421-9988",
  "version_id": "ver_0012",
  "status": "COMMITTED",
  "synced_at": "2026-10-18T12:00:00Z"
}
```

---

### 3.4 Download File
```http
GET /api/v1/files/{file_id}/download?version=latest
Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "file_id": "file_4421-9988",
  "name": "annual_presentation.mp4",
  "total_size": 104857600,
  "manifest": [
    {
      "chunk_index": 0,
      "chunk_hash": "e3b0c442...",
      "signed_url": "https://s3.amazonaws.com/drive-blobs/e3b0c4...?sig=..."
    },
    {
      "chunk_index": 1,
      "chunk_hash": "ca978112...",
      "signed_url": "https://s3.amazonaws.com/drive-blobs/ca9781...?sig=..."
    }
  ]
}
```

---

### 3.5 Directory & Sharing Management
- `POST /api/v1/directories` $\to$ Create folder with `{"name": "Projects", "parent_id": "root"}`.
- `PUT /api/v1/directories/{id}` $\to$ Rename or Move folder.
- `DELETE /api/v1/directories/{id}` $\to$ Soft delete folder hierarchy.
- `POST /api/v1/shares` $\to$ `{"entity_type": "FOLDER", "entity_id": "...", "shared_with": "friend@gmail.com", "permission": "READ_WRITE"}`.

---

## 4. Step 4: High-Level Architecture

![High-Level Architecture](https://ik.imagekit.io/interviewwithbunny/616889478_26113765334874098_4968132749202016461_n.jpg?updatedAt=1768506884752)
*Figure 4.1: High-Level Architecture Diagram (Source: [Interview with Bunny - System Design 18](https://www.interviewwithbunny.com/systemdesign/18))*

### Component Interaction Topology
```
[ Client A (Desktop) ]                  [ Client B (Mobile Device) ]
  ├── Watcher                             ├── Sync Engine
  ├── Chunker (4MB)                       └── Local SQLite DB
  ├── Local SQLite DB
  └── Sync Engine
         │ (1. Init / Commit)                    ▲ (8. Sync Notification)
         ▼                                       │
┌────────────────────────────────────────────────────────────────┐
│                   API Gateway & Load Balancer                  │
└────────────────────────────────────────────────────────────────┘
       │                          │                     ▲
       ▼                          ▼                     │
┌──────────────┐          ┌────────────────┐     ┌────────────────┐
│ User & Quota │          │ File Uploader  │     │  Sync Engine   │
│   Service    │          │    Service     │     │   (WebSocket/  │
└──────────────┘          └────────────────┘     │  Long Polling) │
       │                          │              └────────────────┘
       │                          ▼                     ▲
       │                  ┌────────────────┐            │
       │                  │  Redis Cluster │            │ (7. New Version
       │                  │ (TTL Sessions) │            │     Published)
       │                  └────────────────┘            │
       │                          │                     │
       ▼                          ▼                     ▼
┌────────────────────────────────────────────────────────────────┐
│                     Metadata Service & DB                      │
│                  (PostgreSQL / Relational ACID)                │
└────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────────┐
│                   Object Storage (AWS S3 / GCS)                 │
│         [Chunk 1]  [Chunk 2]  [Chunk 3] ... (Deduplicated)      │
└────────────────────────────────────────────────────────────────┘
       ▲
       │ (Direct Parallel Chunk Upload via Pre-signed URLs)
[ Client A ] ──────────────────────────────────────────┘
```

---

## 5. Step 5: Deep Dive & Low-Level Design

### 5.1 Client-Side Components
Bunny emphasizes that a cloud drive system cannot be designed solely from the server side. The client is a sophisticated micro-system:
1. **Watcher**: Runs background OS-level file monitoring hooks (`ReadDirectoryChangesW` on Windows, `FSEvents` on macOS, `inotify` on Linux) to detect file creation, modification, or deletion instantly.
2. **Chunker**: Splits large files into fixed-size chunks (e.g., 4MB). Computes a cryptographic checksum (SHA-256) for each chunk.
3. **Local SQLite Database**:
   - Stores the local state: `(file_path, chunk_hash, chunk_index, version_id, sync_status)`.
   - Allows the client to detect local modifications without round-tripping to the server.
4. **Sync Engine**: Handles network retry backoff, multi-threaded parallel chunk transfers, and listening for remote update events.

---

### 5.2 Block-Level Deduplication & Space Optimization
- Instead of uploading duplicate copies when multiple users upload the exact same document or video, the system deduplicates at the **Chunk level**:
  - The chunk's SHA-256 hash serves as its unique S3 Object Key: `s3://drive-blobs/{sha256_hash}`.
  - If 1,000 users upload the same 1 GB operating system ISO or popular video, S3 stores the blocks **only once**.
  - The database simply maps each user's `file_version_chunks` table to the existing `chunk_hash` records.

---

### 5.3 Upload Session Management with Redis (TTL)
- **Why Redis?**
  - Tracking upload progress across 50 chunks in PostgreSQL would cause excessive write churn on high-load transactional tables.
  - When `/initiate` is called, an upload session key is created in Redis with a 24-hour TTL:
    ```redis
    upload_session:{upload_id} -> {
      "file_id": "file_4421",
      "user_id": "u_101",
      "expected_chunks": 25,
      "uploaded_chunks": ["chunk_0", "chunk_1"]
    }
    ```
- **Direct S3 Upload via Pre-Signed URLs**:
  - The client uploads chunks **directly to AWS S3 / GCS**, completely bypassing application servers. This offloads gigabytes of network I/O from backend microservices.

---

### 5.4 Commit-Time Batch Validation vs. Per-Chunk Validation
Bunny highlights a critical interview trap around chunk validation:
- **The Naive Mistake**: Validating SHA-256 synchronously on every single chunk upload to S3 via S3 event notifications.
  - *Why it fails*: For a 15–20 GB video file sliced into 4MB chunks, there are 4,000–5,000 chunks. Firing synchronous validation on every chunk consumes all CPU and network capacity of the uploader service.
- **The Optimal Solution (Validator Service)**:
  - Validation occurs **only once upon final commit** (`POST /api/v1/files/upload/commit`).
  - The `File Uploader Service` acknowledges the commit and delegates to an asynchronous **Validator Service**.
  - The Validator Service verifies that all required chunk keys exist in S3 and validates integrity.
  - *Real-world connection*: This is why when you upload a large file to Google Drive or Dropbox, the progress bar hits 100% and then shows **"Processing..."** for a few seconds before becoming accessible.

---

### 5.5 Asynchronous Post-Processing Pipeline (Kafka + Workers)
Once the commit is verified, the file enters an asynchronous background worker pipeline orchestrated via Kafka:
1. **Deduplication Service**:
   - Compares uniform chunk hashes (e.g., 5MB blocks) against global chunk tables.
   - If a duplicate chunk exists, it updates the reference pointer to the existing S3 blob and prunes redundant data.
   - **Why Uniform Chunking Matters**: If files are split into arbitrary chunk sizes, two identical files will produce completely different hash boundaries. Enforcing a strict uniform chunk size (e.g., 5MB for all chunks except the final tail chunk) guarantees identical cryptographic hashes for identical file segments.
2. **Sanitization & Virus Scanning Service**:
   - Scans uploaded blobs for malware/trojans before allowing them to be shared publicly.
3. **Thumbnail & Preview Transcoding Service**:
   - Generates low-resolution previews for images and transcode profiles (1080p, 720p, 480p) for video streaming.

---

### 5.6 Delta Syncing & Network Bandwidth Optimization
- If a user modifies 2 paragraphs in a 500 MB PDF or presentation:
  - The local Chunker re-hashes the file.
  - Only 1 of the 125 chunks will have a mismatched SHA-256 hash.
  - **Only that 1 changed chunk (4 MB) is transmitted over the network**, saving 99% of bandwidth and upload time.

---

### 5.7 Cross-Device Sync Notification Engine
- How does the user's secondary laptop know that a file was updated on their phone?
  1. Once the commit transaction finishes in PostgreSQL, the Metadata Service publishes a `FileUpdatedEvent` to a message broker (Kafka or Redis Pub/Sub).
  2. The **Notification / Sync Service** maintains persistent connections (WebSockets or HTTP Long-Polling) to active client devices.
  3. Client B receives an event: `{"file_id": "file_4421", "new_version": 2}`.
  4. Client B queries the Metadata Service for the chunk manifest of Version 2, compares it with its local SQLite DB, and downloads only the newly changed chunk.

---

### 5.8 Conflict Resolution
- If two devices edit the same file simultaneously while offline:
  - Both devices try to commit a new version based on Version 1.
  - The database uses **Optimistic Concurrency Control**:
    ```sql
    UPDATE file_metadata 
    SET latest_version = 2 
    WHERE id = :file_id AND latest_version = 1;
    ```
  - The first device succeeds.
  - The second device's update fails with a version conflict. 
  - The server handles this by creating a **Conflicted Copy** (the standard Dropbox approach):
    `annual_presentation (Device B's conflicted copy 2026-10-18).mp4`, preserving both users' work without data loss.

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: How do you implement chunking and deduplication (Fixed-Size vs. Rolling Hash / Rabin Fingerprints)?

Chunking and deduplication trade-offs in cloud storage:

- **Fixed-size chunking (e.g. 4MB chunks)**:
  Simple to implement. Hash each 4MB block using SHA-256. The fatal flaw: If a user inserts 1 byte at the beginning of a 1GB file, every chunk boundary shifts by 1 byte. The SHA-256 hash of every subsequent chunk changes completely, causing 100% deduplication failure and forcing a full 1GB re-upload.

- **Content-Defined Chunking (CDC) via Rabin Fingerprints / FastCDC**:
  A sliding window computes a rolling polynomial hash over the byte stream. When the lower N bits of the hash match a condition (e.g. hash & 0x0FFF == 0), a chunk boundary is declared (averaging 4MB).

- **Shift-resilience**:
  If a byte is inserted at byte 0, only the first chunk boundary changes; all downstream 4MB chunks retain identical boundaries and identical SHA-256 hashes.

- **Global deduplication**:
  Before uploading, the client sends a list of chunk hashes to the Metadata Service. The server responds with which chunks already exist in S3 (e.g. identical OS images, videos, common documents). The client only uploads the truly unique missing chunks, saving up to 60-70% bandwidth and storage.

---

### Q2: How do you handle resumable uploads for multi-gigabyte files over unstable networks?

Resumable upload state machine and S3 multipart orchestration:

- **Initialization**:
  Client initiates upload via POST /api/v1/files/upload-session with {file_name, total_size, total_chunks, file_hash}. Backend generates an upload_session_id and pre-allocates an AWS S3 multipart upload (InitiateMultipartUpload).

- **Presigned URL generation**:
  Server returns presigned S3 PUT URLs for each chunk index, allowing clients to stream blocks directly to object storage without choking API gateway servers.

- **Parallel chunk streaming**:
  Client uploads chunks concurrently (e.g. 4 parallel workers). If the mobile connection drops at chunk 47/100, the client pauses.

- **Resume handshake**:
  On reconnection, client calls GET /api/v1/files/upload-session/{session_id}/status. Server queries S3 ListParts to return the list of already-committed part ETags. Client resumes uploading only parts 48 to 100.

- **Finalization**:
  Once all chunks land in S3, client calls POST /api/v1/files/upload-session/{session_id}/complete. Server calls S3 CompleteMultipartUpload, updates file status in PostgreSQL metadata DB to 'ACTIVE', and broadcasts sync events.

---

### Q3: How do you maintain consistency between Relational Metadata DB and S3 Block Storage under concurrent edits?

Decoupled consistency and optimistic concurrency control:

- **S3 Immutability**:
  S3 object storage is strictly write-once, read-many (WORM). Blocks are keyed by their content hash (s3://bucket/chunks/{sha256_hash}), meaning chunk objects are immutable and never updated in place.

- **Optimistic Concurrency Control (OCC) in Metadata DB**:
  Each file record contains a monotonically increasing version integer: file_metadata (file_id, version, root_chunk_manifest_id, updated_at).

- **Atomic Commit**:
  When Client A and Client B concurrently edit file_id 101, both upload new chunks to S3. When committing metadata, Client A executes: UPDATE file_metadata SET version = version + 1, manifest_id = :new_manifest WHERE file_id = 101 AND version = 5.

- **Conflict Resolution**:
  Client A's update affects 1 row and succeeds. When Client B attempts the same with version = 5, 0 rows are affected. Client B's transaction aborts with 409 Conflict. Client B fetches version 6, performs a 3-way merge or generates a conflicted copy ('Document (Alice's conflicted copy).docx').

- **Garbage Collection**:
  Any orphaned chunks uploaded to S3 that are never referenced by an active metadata manifest are cleaned up by an asynchronous Mark-and-Sweep GC worker after 7 days.

---

### Q4: How does the client synchronization engine propagate file updates across millions of connected devices in real time?

Real-time client synchronization pipeline:

- **Long-Polling vs. WebSockets vs. SSE**:
  WebSockets maintain full-duplex overhead; Server-Sent Events (SSE) or HTTP/2 Push are preferred because file sync is 99% unidirectional notifications from server to client.

- **Connection Gateway**:
  Client devices (laptop, mobile, web) maintain a persistent HTTP/2 connection to a dedicated Sync Notification Gateway.

- **Change Capture**:
  When an edit commits to PostgreSQL, an event is emitted via CDC/Kafka topic 'file-change-events' with {user_id, file_id, new_version, modified_chunk_hashes}.

- **Device Filtering**:
  Sync gateway checks which client devices belong to user_id (and shared folder collaborators), pushing a lightweight sync packet: {event: 'REMOTE_UPDATE', file_id: 101, version: 6}.

- **Delta Download**:
  The receiving client queries GET /api/v1/files/{file_id}/delta?base_version=5. Server returns only the 1 modified chunk hash. Client downloads the single 4MB chunk from CDN/S3 and reconstructs the file locally on disk without re-downloading the remaining 996MB.

---

## 7. Interview Summary & Key Takeaways

| Dimension | Bunny's Architectural Decision | Justification |
| :--- | :--- | :--- |
| **Storage Architecture** | Separation of Metadata (PostgreSQL) and Raw Blobs (AWS S3) | Relational DB handles complex tree structures and ACID; S3 provides 11 9's durability at low cost. |
| **Upload Pipeline** | Pre-signed S3 URLs with Client Chunking | Prevents backend application servers from bottlenecking on raw file I/O. |
| **Storage Optimization**| SHA-256 Block-level Deduplication | Eliminates duplicate chunk storage globally across millions of accounts. |
| **Sync Latency** | Delta Sync + WebSocket / Long Polling | Minimizes network footprint; keeps secondary devices updated within seconds. |
| **Resilience & State** | In-memory Redis tracking with TTL | Survives client connection drops; enables seamless resumable uploads without polluting persistent tables. |

