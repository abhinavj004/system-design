# System Design: Medicine Development & Research Platform (Pharma / Biotech)

> **Source Reference**: [Blink Health / Biotech System Design Interview Experience]  
> **Methodology**: Formatted using the [Master 5-Step Architectural Blueprint](./00-system-design-blueprint-pattern.md) (*Interview with Bunny* FAANG Standard).  
> **Topic**: Designing an End-to-End Drug Discovery & Development Platform with Polyglot Persistence, Distributed Compute Workflows, and AI/ML Integrations.

---

## ⚡ Last-Minute Interview Cram Box (3-Minute Revision)

> **The 30-Second Mental Model**: Deconstruct drug development into 5 domain-driven microservices. Biological pathways and molecular graphs are stored in a **Graph Database (Neo4j / Amazon Neptune)**, clinical trial records reside in an **ACID Relational Database (PostgreSQL)**, and raw cryo-EM / genomic files reside in **Object Storage (S3)**. Long-running in-silico simulations (molecular docking, protein folding) are orchestrated using a stateful DAG engine (**Temporal / Nextflow**) feeding an auto-scaling **GPU Worker Fleet (AWS Batch / Slurm)**. Service-to-service communication uses **synchronous gRPC** for low latency and **Apache Kafka** for domain events. AI is integrated at 4 key milestones: **Target GNNs**, **Generative Diffusion for de novo molecules**, **Predictive ADMET toxicity**, and **Clinical Trial cohort matching**.

| Metric / Dimension | Production Target / Value (Biotech Scale) |
| :--- | :--- |
| **Compound Library** | **10,000,000+ chemical compounds**; 100,000+ biological targets |
| **Raw Scientific Data** | Terabytes to Petabytes of cryo-EM images and genomic reads (S3) |
| **Simulation Runtime** | Hours to days per job (requires checkpointing and resume) |
| **Inter-Service Latency**| Synchronous queries $< 5\text{ ms}$ (gRPC Protobuf) |
| **Regulatory Standards**| Strict **FDA 21 CFR Part 11** immutable audit logs & **HIPAA** compliance |

### Top 3 Interviewer Traps & Must-Say Defenses
1. **The Single Database Trap (Relational vs Graph vs Lakehouse)**:
   - *Trap*: *"Why not store everything in PostgreSQL with JSONB columns?"*
   - *Defense*: Chemical structures and biological targets require multi-hop relationship traversals (*Target $\to$ Pathway $\to$ Disease*), where relational joins degrade exponentially. We use **Graph DB** for knowledge graphs, **PostgreSQL** for strict ACID clinical trial cohorts, and **S3 / Parquet Lakehouse** for massive raw genomic files.
2. **The "Just Use Kafka" Long-Running Task Trap**:
   - *Trap*: *"Can't we just push a simulation job to a Kafka topic and let a worker process it?"*
   - *Defense*: Kafka is an append-only event stream, not a workflow orchestrator. Molecular docking and folding run for 10–24 hours across GPU clusters. If a node fails at hour 9, Kafka cannot manage step-level state machines, retries, or checkpointing. We must use a **Workflow Orchestrator (Temporal / Nextflow)** that manages DAG dependencies and resumes from checkpoints.
3. **FDA 21 CFR Part 11 Compliance Trap**:
   - *Trap*: *"How do you handle audit logging when scientists update experimental findings?"*
   - *Defense*: Standard application logs are insufficient. Under FDA regulations, scientific records must have an **immutable, append-only cryptographic audit trail** with electronic signatures, timestamps, and justification reasons. Previous records are never updated in-place.

> **Golden Architectural Rule**: Never run heavy scientific simulations inside synchronous HTTP request threads. Use gRPC for fast metadata lookups, Kafka for domain events, and stateful workflow DAGs for multi-hour GPU simulations.

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Functional Requirements (FR)
1. **Target & Compound Management**: Scientists can search, register, and link biological targets (proteins, genes) and chemical molecules.
2. **In-Silico Simulation Workflows**: Scientists can configure and launch multi-step computational experiments (virtual screening, molecular docking, protein folding).
3. **Clinical Trial Tracking**: Coordinators can manage trial phases (Phase I/II/III), participant cohorts, dosage logs, and adverse event reports.
4. **Sample & Reagent Inventory**: Track physical freezer storage, compound aliquot concentrations, shelf-life, and chain of custody.
5. **Regulatory Compliance & Electronic Signatures**: Enforce FDA 21 CFR Part 11 compliant audit trails for all data mutations and experiment approvals.

---

### 1.2 Non-Functional Requirements (NFR)
1. **Data Integrity & Auditability (Highest Priority)**: Regulatory data must be tamper-proof, ACID-compliant, and fully auditable.
2. **Compute Fault Tolerance**: Long-running simulations running across spot GPU instances must survive worker crashes and resume from intermediate checkpoints.
3. **Low-Latency Inter-Service Queries**: Core metadata and inventory checks must respond in $< 5\text{ ms}$ over internal networks.
4. **Scalability**: Capable of screening libraries of **10+ Million compounds** against target proteins.

---

### 1.3 Capacity Estimations (Back-of-the-Envelope Math)

#### Data & Storage Sizing
- **Compound Library**: $10,000,000$ chemical compounds.
  - Molecular metadata (SMILES string, 3D coordinates, properties) $\approx 2\text{ KB}$ per molecule $\implies \mathbf{20\text{ GB}}$ (in-memory / graph accessible).
  - Molecular embeddings (512-dim float vectors for similarity search) $\approx 2\text{ KB}$ per molecule $\implies \mathbf{20\text{ GB}}$ in Vector DB.
- **Raw Scientific Assay & Imaging Data**:
  - High-content microscopy and cryo-EM images: $\approx 50\text{ GB}$ per experiment run.
  - 100 runs/week $\implies 5\text{ TB/week} \approx \mathbf{260\text{ TB/year}}$ (stored in AWS S3 with Glacier tiering).
- **Clinical Trial Data**:
  - 10,000 trial participants across 5 active studies; telemetry logged daily $\approx 500\text{ MB/year}$ (relational, fits easily in PostgreSQL).

#### Compute & Simulation Workload
- **Virtual Screening Burst**: Screening 100,000 candidate ligands against 1 target pocket:
  - Average docking time per ligand $\approx 10\text{ seconds}$ on an NVIDIA GPU core.
  - Total compute required:
    $$\text{Total Compute} = 100,000 \times 10\text{ s} = 1,000,000\text{ GPU seconds} \approx \mathbf{277\text{ GPU hours}}$$
  - Running on an autoscaling cluster of **64 GPU instances** (e.g. AWS `g5.2xlarge`), screening finishes in $\approx \mathbf{4.3\text{ hours}}$.

---

## 2. Step 2: Identification of Core Entities & Polyglot Data Modeling

Because drug discovery spans chemistry, biological networks, clinical trials, and compliance, **a single database technology is an anti-pattern**.

```
┌─────────────────────────────────┬───────────────────────────────┬────────────────────────────────────────────────────────┐
│ Domain / Microservice           │ Database Technology           │ Architectural Justification                            │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ Target & Lead Discovery         │ Graph DB (Neo4j / Neptune) +  │ Graph DB models multi-hop biological pathways          │
│ (Knowledge Graph)               │ Vector DB (Milvus / Pinecone) │ (Gene ──▶ Protein ──▶ Pathway ──▶ Disease).            │
│                                 │                               │ Vector DB enables sub-10ms chemical similarity search. │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ Clinical Trial Management       │ Relational (PostgreSQL)       │ Strict ACID compliance, foreign keys for participant   │
│                                 │                               │ cohorts, double-blind visits, and adverse events.      │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ Raw Assay & Microscopy Files    │ Cloud Object Storage (S3/GCS) │ Petabyte-scale, cost-effective storage for multi-GB    │
│                                 │ + Apache Iceberg / Parquet    │ raw imaging and sequencing data (FASTQ/BAM).           │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ Regulatory Audit & Signatures   │ Append-Only Immutable Log     │ FDA 21 CFR Part 11 mandates tamper-evident historical  │
│ (21 CFR Part 11)                │ (PostgreSQL Audit / QLDB)     │ logs with cryptographic hashes and change reasons.     │
├─────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────┤
│ Chemical Inventory & Freezers   │ PostgreSQL                    │ Barcode lookups, optimistic concurrency control for    │
│                                 │                               │ concurrent sample aliquoting and freezer slot checks.  │
└─────────────────────────────────┴───────────────────────────────┴────────────────────────────────────────────────────────┘
```

### Core Entities & Relationships

```
┌──────────────────┐               ┌──────────────────┐               ┌──────────────────┐
│ BiologicalTarget │   Interacts   │ ChemicalCompound │   Allocated   │  PhysicalSample  │
├──────────────────┤───────────────├──────────────────┤───────────────├──────────────────┤
│ target_id (PK)   │◀─────────────▶│ compound_id (PK) │◀─────────────▶│ sample_id (PK)   │
│ gene_symbol      │   (Graph)     │ smiles_string    │  (Relational) │ compound_id (FK) │
│ protein_name     │               │ molecular_weight │               │ freezer_slot_id  │
│ organism         │               │ vector_embedding │               │ concentration_um │
└──────────────────┘               └──────────────────┘               │ volume_ul        │
                                            │                         └──────────────────┘
                                            │ Evaluated In
                                            ▼
                                   ┌──────────────────┐
                                   │  ClinicalTrial   │
                                   ├──────────────────┤
                                   │ trial_id (PK)    │
                                   │ compound_id (FK) │
                                   │ phase (I,II,III) │
                                   │ status (ACTIVE)  │
                                   └──────────────────┘
```

---

## 3. Step 3: API & Inter-Service Protocol Design

### 3.1 Synchronous Internal Interface (gRPC / Protobuf)
Internal service calls use **gRPC over HTTP/2** for multiplexing, binary serialization, and sub-5ms latency:

```protobuf
syntax = "proto3";
package pharma.inventory.v1;

service InventoryService {
  rpc ReserveCompoundSample (ReserveSampleRequest) returns (ReserveSampleResponse);
  rpc GetSampleAvailability (GetSampleRequest) returns (SampleAvailability);
}

message ReserveSampleRequest {
  string compound_id = 1;
  double volume_ul = 2;
  string experiment_id = 3;
  string researcher_id = 4;
}

message ReserveSampleResponse {
  string sample_id = 1;
  string barcode = 2;
  string freezer_location = 3;
  bool success = 4;
}
```

### 3.2 External RESTful Endpoints (Researcher Web Portal)
```http
POST /api/v1/experiments/simulations
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
  "target_id": "tgt_HER2_human",
  "compound_ids": ["cmp_88192", "cmp_88193", "cmp_88194"],
  "simulation_type": "MOLECULAR_DOCKING",
  "parameters": {
    "exhaustiveness": 8,
    "num_modes": 9
  }
}

Response: 202 Accepted
{
  "workflow_id": "wf_dock_991823",
  "status": "QUEUED",
  "estimated_runtime_seconds": 1800,
  "telemetry_stream": "/api/v1/workflows/wf_dock_991823/events"
}
```

---

## 4. Step 4: High-Level Architecture (HLD)

### Component Interaction Topology

```
[ Research Scientists & Clinicians ]
                │
                ▼ (HTTPS / REST & WebSockets)
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              API Gateway (Kong / Envoy)                                │
│        (JWT Authentication, Role-Based Access Control, Rate Limiting, Audit Tagging)   │
└────────────────────────────────────────────────────────────────────────────────────────┘
       │                                 │                                │
       ▼ (gRPC)                          ▼ (gRPC)                         ▼ (REST)
┌──────────────┐                  ┌──────────────┐                 ┌──────────────┐
│ Target & Lead│                  │ Chemical     │                 │ Clinical     │
│ Discovery    │                  │ Inventory    │                 │ Trials       │
└──────────────┘                  └──────────────┘                 └──────────────┘
       │ (Read/Write)                    │ (Read/Write)                   │ (Read/Write)
       ▼                                 ▼                                ▼
┌──────────────┐                  ┌──────────────┐                 ┌──────────────┐
│ Graph DB     │                  │ PostgreSQL   │                 │ PostgreSQL   │
│ (Neo4j)      │                  │ (Inventory)  │                 │ (Clinical)   │
└──────────────┘                  └──────────────┘                 └──────────────┘
       │                                 │                                │
       └────────────────────────┬────────┴────────────────────────────────┘
                                │ Domain Events (e.g. CompoundSynthesized, BatchApproved)
                                ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 Apache Kafka Event Bus                                 │
└────────────────────────────────────────────────────────────────────────────────────────┘
         │                                                        │
         ▼ (Triggers Long Compute)                                ▼ (Audit Consumer)
┌─────────────────────────────────────────┐             ┌────────────────────────────────┐
│      Simulation Compute Orchestrator    │             │   Compliance & Audit Service   │
│         (Temporal.io / Nextflow)        │             │   (FDA 21 CFR Part 11 Engine)  │
└─────────────────────────────────────────┘             └────────────────────────────────┘
         │                                                               │
         ├─── Dispatches Tasks                                           ▼
         ▼                                                      ┌────────────────────────┐
┌─────────────────────────────────────────┐                     │ Immutable Audit Store  │
│  AWS Batch / Slurm GPU Worker Fleet     │                     │ (Append-Only / QLDB)   │
│  (NVIDIA A100/H100 - Auto-scaling)      │                     └────────────────────────┘
└─────────────────────────────────────────┘
         │
         ▼ Writes Raw Trajectories & Checkpoints
┌─────────────────────────────────────────┐
│     Object Storage (S3 / Lakehouse)     │
└─────────────────────────────────────────┘
```

---

### 4.1 Step-by-Step Architectural Evolution (Whiteboard Script)

#### Phase 1: The Monolith & Shared Relational DB
- **Whiteboard**: `Client` ──► `Monolithic API Server` ──► `Single PostgreSQL DB`.
- **Explanation**: *"We start with a single service storing compounds, inventory, and experiments in relational tables. When a scientist clicks 'Run Simulation', the server runs the docking script directly in a worker thread."*
- **Bottlenecks**:
  1. *Join Explosion*: Querying chemical pathways requires recursive self-joins that freeze the DB.
  2. *Worker Starvation*: A 6-hour docking simulation locks application worker threads, causing HTTP request timeouts for all other users.

#### Phase 2: Domain-Driven Microservices & Polyglot Persistence
- **Whiteboard**: Split into **Discovery Service (Graph DB)**, **Inventory Service (PostgreSQL)**, and **Clinical Trial Service (PostgreSQL)**.
- **Explanation**: *"We decouple the domains. Complex molecular graphs and biological targets move to Neo4j for $O(1)$ multi-hop traversals. Clinical trial and inventory records stay in PostgreSQL for ACID transactions."*

#### Phase 3: Synchronous gRPC for Fast Internal Metadata
- **Whiteboard**: Add **gRPC** between Discovery and Inventory.
- **Explanation**: *"Before a compound can be used in an experiment, Discovery queries Inventory for sample availability. Using gRPC over HTTP/2 with binary Protobuf provides sub-5ms low latency."*

#### Phase 4: Decouple Long Simulations via Workflow DAGs (Temporal)
- **Whiteboard**: Introduce **Temporal / Nextflow** + **AWS Batch GPU Cluster**.
- **Explanation**: *"Simulations cannot run inside HTTP lifecycles. The API returns `HTTP 202 Accepted`. Temporal orchestrates the multi-step DAG: fetch ligand $\to$ dock $\to$ score $\to$ notify. If a GPU spot instance drops at hour 5, Temporal resumes from the last checkpoint without restarting."*

#### Phase 5: Event-Driven Kafka Bus & Immutable Compliance
- **Whiteboard**: Connect all services to **Apache Kafka**, feeding an **Audit & Compliance Service (21 CFR Part 11)**.
- **Explanation**: *"Pharma regulations require an unalterable history. When an experiment completes or a trial stage advances, an event publishes to Kafka. The Compliance service writes a cryptographically signed, append-only audit record."*

#### Phase 6: Layering AI/ML into Discovery and Screening
- **Whiteboard**: Embed AI inference workers into the screening and simulation pipeline.
- **Explanation**: *"We add machine learning surrogate models (GNNs, Diffusion, and XGBoost) to predict toxicity and binding affinity in milliseconds before spending expensive GPU hours on physical physics simulations."*

---

## 5. Step 5: Low-Level Design & Deep Dives

### 5.1 Deep Dive 1: Asynchronous Execution for Long-Running Tasks

How do we reliably execute simulations that take 12+ hours across GPU clusters?

```
[ Client: POST /simulations ] ──▶ Returns HTTP 202 Accepted (workflow_id)
             │
             ▼
┌────────────────────────────────────────────────────────┐
│                 Temporal.io Orchestrator               │
│   ┌────────────────────────────────────────────────┐   │
│   │ Step 1: Preprocess PDB Structure (CPU Worker)  │   │
│   │ Step 2: AutoDock Vina / DiffDock (GPU Worker)  │ ◀─┼── Checkpoint Saved to S3
│   │ Step 3: Molecular Dynamics Energy (GPU Worker) │   │
│   │ Step 4: Aggregate Binding Affinities           │   │
│   └────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
             │                                    │
             ▼ Push Progress Event                ▼ Checkpoints & Results
┌─────────────────────────┐             ┌─────────────────────────┐
│ Redis Pub/Sub ──▶ SSE   │             │   Object Storage (S3)   │
│ (Live UI Progress Bar)  │             └─────────────────────────┘
```

1. **Stateful DAG Orchestration**:
   - Workflows are authored as code using **Temporal** or **Nextflow**.
   - Each simulation step is an independent activity. Temporal persists the state of every completed activity to disk.
2. **Checkpointing & Spot Instance Recovery**:
   - In-silico simulations output intermediate coordinate checkpoints (e.g. `.dcd` trajectory frames) to S3 every 15 minutes.
   - If AWS terminates an economical GPU Spot instance, the replacement instance automatically downloads the last checkpoint and resumes without data loss.
3. **Real-Time Telemetry**:
   - The worker reports progress percentages to **Redis Pub/Sub**, which pushes updates to the researcher's browser over **Server-Sent Events (SSE)**.

---

### 5.2 Deep Dive 2: AI & Machine Learning Integration Points

Where does AI integrate into the medicine development lifecycle?

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Target Discovery │ ──▶ │ Lead Generation  │ ──▶ │ In-Silico ADMET  │ ──▶ │ Clinical Trials  │
│  (GNN / BioBERT) │     │(Diffusion Models)│     │(Gradient Boosted)│     │(Clinical NLP/LLM)│
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘
```

1. **Target Identification (Graph Neural Networks)**:
   - *Model*: Graph Neural Networks (GNNs) on biological knowledge graphs.
   - *Impact*: Predicts novel protein-disease associations and link prediction across 100,000 biological nodes.
2. **De Novo Molecule Generation (Equivariant Diffusion)**:
   - *Model*: Diffusion models (e.g., DiffDock, RFdiffusion, AlphaFold 3).
   - *Impact*: Generates novel 3D molecular candidates specifically tailored to bind to a target protein's active pocket.
3. **Predictive ADMET Profiling (Pre-filtering)**:
   - *Model*: XGBoost & Graph Attention Networks (GATs).
   - *Impact*: Predicts **ADMET** (*Absorption, Distribution, Metabolism, Excretion, Toxicity*) in $< 10\text{ ms}$. Eliminates molecules with high predicted liver toxicity or hERG cardiac risk before physical wet-lab synthesis.
4. **Clinical Trial Optimization (NLP & Stratification)**:
   - *Model*: Clinical LLMs & survival models.
   - *Impact*: Parses unstructured clinical notes to match eligible patients to trials and builds **Synthetic Control Arms** from historical patient datasets.

---

### 5.3 Deep Dive 3: Regulatory Compliance (FDA 21 CFR Part 11 & HIPAA)

How do we guarantee that clinical and experimental records cannot be altered or falsified?

1. **Immutable Append-Only Audit Trail**:
   - Update and delete operations are strictly prohibited on experimental findings tables.
   - Every mutation is recorded as a new audit event:
     ```sql
     CREATE TABLE experiment_audit_log (
         audit_id BIGSERIAL PRIMARY KEY,
         entity_type VARCHAR(64) NOT NULL,
         entity_id VARCHAR(64) NOT NULL,
         action VARCHAR(20) NOT NULL,            -- INSERT, AMEND, APPROVE
         changed_by VARCHAR(64) NOT NULL,
         change_reason TEXT NOT NULL,            -- Mandatory justification
         previous_state JSONB,
         new_state JSONB,
         cryptographic_hash VARCHAR(64) NOT NULL, -- SHA-256(prev_hash + new_state + timestamp)
         timestamp TIMESTAMPTZ DEFAULT NOW()
     );
     ```
2. **Cryptographic Chaining**:
   - Each audit log entry includes the hash of the preceding record (merkle-chain structure). Any retroactive database tampering invalidates downstream hashes.
3. **HIPAA Safe Harbor De-Identification**:
   - Direct patient identifiers (Names, SSNs, phone numbers) are stripped at the API Gateway.
   - The clinical database stores only anonymized subject IDs (`SUBJ-9921`) using salt-hashed identifiers.

---

### 5.4 Deep Dive 4: Chemical Sample Inventory & Freezer Slot Locking

When two researchers simultaneously request the last 50 microliters ($\mu\text{L}$) of a rare compound from a $-80^\circ\text{C}$ cryogenic freezer:

```sql
-- Optimistic Concurrency Control (OCC) with Versioning
UPDATE sample_vials
SET available_volume_ul = available_volume_ul - :requested_volume,
    version = version + 1
WHERE sample_id = :sample_id
  AND version = :expected_version
  AND available_volume_ul >= :requested_volume;
```
- If two allocation requests hit simultaneously:
  - Researcher A's update modifies 1 row $\implies$ **Confirmed**.
  - Researcher B's update modifies 0 rows $\implies$ **Rejected with `HTTP 409 Conflict: Insufficient Volume`**.
  - Eliminates sample over-allocation and physical freezer slot conflicts.

---

## 6. Interview Summary & Key Takeaways

| Architectural Dimension | Chosen Solution | Alternative Considered | Trade-off Justification |
| :--- | :--- | :--- | :--- |
| **Data Modeling** | Polyglot: Graph DB (Neo4j) + Relational (Postgres) | Single Global Relational Database | Relational joins cannot scale across multi-hop biological pathways. Neo4j executes graph traversals $100\times$ faster. |
| **Simulation Compute** | Stateful DAG Engine (Temporal) + AWS Batch GPU | Standard Message Queue (Kafka) Worker | Kafka cannot handle multi-hour state machines, step retries, or spot instance checkpointing. Temporal resumes interrupted jobs. |
| **Inter-Service Calls** | gRPC over HTTP/2 with Protobuf | Synchronous REST HTTP/1.1 | gRPC multiplexing and binary serialization cut internal inter-service latency down to $< 5\text{ ms}$. |
| **Regulatory Audit** | Append-Only Cryptographic Audit Log | Standard Application Log Files (ELK) | FDA 21 CFR Part 11 requires immutable audit records with mandatory justification reasons and tamper-evident hashes. |
| **AI Integration** | Fast Surrogate ML for ADMET Pre-filtering | Running full physics docking on all molecules | Screening 10M molecules via physics docking takes years. ML filters out 99% of toxic molecules in minutes. |

---

### Top 4 Interview Follow-Up Questions (Rapid-Fire Defenses)

### Q1: What happens if a simulation fails midway through an 8-hour execution?
- **Answer**: The workflow orchestrator (**Temporal**) detects worker heartbeat loss. It provisions a replacement worker, retrieves the latest intermediate coordinate checkpoint from **S3**, and resumes execution from that specific activity step rather than restarting the entire 8-hour simulation.

### Q2: How do you handle patient data privacy between research teams and clinical trial coordinators?
- **Answer**: Enforce strict **Role-Based Access Control (RBAC)** at the API Gateway. Clinical trial coordinators access de-identified patient data via dedicated HIPAA-compliant services. Research scientists only have access to aggregated cohort statistics; direct and indirect identifiers are never exposed to discovery services.

### Q3: Why not use a Graph Database for everything, including Clinical Trials?
- **Answer**: Graph databases trade ACID transaction guarantees and relational schema enforcement for relationship traversal speed. Clinical trial management requires strict foreign-key constraints, financial auditing, and row-level locking for patient cohort enrollment, which PostgreSQL handles natively.

### Q4: How do you perform sub-10ms chemical similarity searches across 10 Million compounds?
- **Answer**: Pre-calculate **Morgan chemical fingerprints** or neural molecular embeddings for all compounds. Index them into a **Vector Database (Milvus / Pinecone)** using Hierarchical Navigable Small World (**HNSW**) graphs to execute sub-10ms cosine similarity queries.
