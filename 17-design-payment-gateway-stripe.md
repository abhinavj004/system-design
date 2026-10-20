# System Design 17: Design Payment Gateway System (Stripe / Razorpay / PayPal)

> **Source**: [Interview with Bunny - Episode 17](https://www.youtube.com/watch?v=7EnqCwQOddY)  
> **Companion Reference**: [interviewwithbunny.com/systemdesign/17](https://www.interviewwithbunny.com/systemdesign/17)  
> **Topic**: Designing a Secure, Highly Consistent, Multi-Tenant Payment Gateway (Stripe, Razorpay, Adyen, PayPal)

---

## 1. Step 1: Requirements Gathering & Estimations

### 1.1 Domain Clarification: Payment Gateway vs. Payment Processor
Before designing a payment infrastructure, Bunny stresses an essential architectural distinction:
- **Payment Gateway (Our System)**: The customer-facing traffic controller and orchestration platform (e.g., Stripe, Razorpay, PayPal Checkout). It captures customer card credentials via a secure hosted checkout environment, tokenizes sensitive cardholder data to satisfy **PCI-DSS Level 1** compliance, manages payment sessions, routes transactions across multiple processors, enforces idempotency, and reconciles transactions.
- **Payment Processor (Downstream Partner)**: Financial network entities and acquiring bank gateways (e.g., Chase Paymentech, First Data/Fiserv, Worldpay, Visa/Mastercard networks). Processors interact directly with issuing and acquiring banks to execute authorization, currency capture, clearing, and interbank settlement.

```
[ Merchant Web / Mobile App ]
             │
             ▼
┌────────────────────────────────────────┐
│     Payment Gateway (Our System)       │ ─── Orchestration, Tokenization,
│ (Stripe / Razorpay / PayPal Checkout)  │     Session Management, Idempotency
└────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│           Payment Processor            │ ─── Financial Networks, Bank Rails,
│ (First Data / Worldpay / Visa / MC)    │     Funds Movement, Clearing
└────────────────────────────────────────┘
             │
             ▼
[ Issuing Bank ] <=======> [ Acquiring Bank ]
```

---

### 1.2 Functional Requirements (FR)
1. **Payment Intent Creation**: Merchants can initiate a payment transaction for an order (specifying amount, currency, merchant ID, customer ID).
2. **Hosted Checkout Session Management**: The gateway generates short-lived, secure checkout sessions (10-minute TTL) and hosts an isolated payment web page (`https://gateway.com/pay/{session_id}`) or iframe.
3. **PCI-DSS Compliant Card Ingestion & Tokenization**: End users enter sensitive Primary Account Numbers (PAN), CVVs, and expiry dates directly into the gateway's isolated PCI zone. The gateway tokenizes the card into an opaque token using a **Hardware Security Module (HSM)**. The merchant application never touches, transmits, or stores raw card data.
4. **Smart Multi-Processor Orchestration**: The orchestrator evaluates merchant routing rules and processor health, transforms payloads via the **Adapter Pattern**, and dispatches payment authorization calls to downstream processors (e.g., Razorpay, PayU, Stripe).
5. **Asynchronous Webhook & Status Callbacks**: Ingest real-time authorization callbacks from external processor gateways and forward cryptographically signed webhooks to merchant endpoints.
6. **Double-Entry Ledger & Financial Reconciliation**: An offline automated reconciliation engine tallies internal transactions against downstream bank settlement files (T+1/T+2 days) into an immutable double-entry ledger.

#### Explicitly Out of Scope
- Partial payments / split tenders.
- Physical Point-of-Sale (POS) terminal hardware.
- Chargeback arbitration disputes.

---

### 1.3 Non-Functional Requirements (NFR)
1. **High Throughput**: Support peak loads of **10,000 Transactions Per Second (TPS)**.
2. **Strict Consistency over Availability (CAP Theorem - Strictly CP)**:
   - In financial platforms, **Strong Consistency (CP)** is mandatory.
   - Returning HTTP 500 or failing a checkout attempt is vastly preferable to charging a customer twice or corrupting the financial ledger.
   - Strict ACID transactions and atomic distributed locks for state mutations.
3. **Low Latency Orchestration**:
   - Gateway tokenization, session validation, and processor dispatch must complete in **$< 200\text{ms}$** (excluding external bank processing time).
4. **Strict Idempotency**:
   - The platform must guarantee **Exactly-Once Semantics** from a business perspective. Retried network requests bearing the same `X-Idempotency-Key` must never result in duplicate bank charges.
5. **Zero-Trust Security & PCI-DSS Compliance**:
   - Dedicated, air-gapped PCI-DSS Level 1 certified enclave for the Card Vault.
   - End-to-end TLS 1.3 encryption in transit; AES-256 HSM envelope encryption at rest.

---

### 1.4 Capacity Estimations (Back-of-the-Envelope Math)

#### Throughput
- **Peak Load**: $10,000 \text{ TPS}$ (Global Black Friday / Cyber Monday flash sale).
- **Average Load**: $3,000 \text{ TPS}$.
- Daily volume: $3,000 \times 86,400 \approx 259 \text{ Million transactions / day}$.

#### Storage Growth (5-Year Horizon)
- Transaction Record size:
  - `transaction_id`: 16 bytes (UUID)
  - `intent_id`: 16 bytes
  - `merchant_id`: 16 bytes
  - `amount`: 8 bytes (Stored as integer in cents/paise to prevent floating-point precision errors)
  - `currency`: 3 bytes (ISO 4217, e.g., `USD`)
  - `status`: 4 bytes
  - `card_token`: 32 bytes
  - `processor_ref`: 32 bytes
  - `metadata`: 200 bytes
  - Total per transaction $\approx 350 \text{ bytes}$.
- **Daily Storage Growth**:
  $$259 \times 10^6 \times 350 \text{ bytes} \approx 90.6 \text{ GB / day}$$
- **Annual Storage**: $90.6 \text{ GB} \times 365 \approx 33 \text{ TB / year}$.
- Fits comfortably in a partitioned relational database cluster (PostgreSQL with read replicas and Citus sharding).

#### Memory / Caching (Active Payment Sessions)
- Active session TTL: 10 minutes ($600\text{ seconds}$).
- Concurrently active sessions at 10,000 TPS:
  $$\text{Concurrent Sessions} = 10,000 \times 600 = 6,000,000 \text{ sessions in RAM}$$
- Payload per session in Redis $\approx 500 \text{ bytes}$.
- Total Redis RAM required: $6,000,000 \times 500 \text{ bytes} \approx 3 \text{ GB RAM}$.
- Negligible memory footprint; easily handled by a 32 GB Redis cluster.

---

## 2. Step 2: Identification of Core Entities

```
+--------------------+        +--------------------+        +--------------------+
|      Merchant      |        |   PaymentIntent    |        |  CheckoutSession   |
+--------------------+        +--------------------+        +--------------------+
| merchant_id(PK,UUID| 1    * | intent_id (PK,UUID)| 1    1 | session_id (PK,VAR)|
| business_name(VAR) |<------>| merchant_id (FK)   |<------>| intent_id (FK)     |
| api_key (VARCHAR)  |        | amount_cents (BIG) |        | redis_ttl (INT)    |
| webhook_url (TEXT) |        | currency (CHAR 3)  |        | redirect_url (TEXT)|
| status (ACTIVE)    |        | status (REQUIRES_  |        | created_at (TIME)  |
+--------------------+        |         PAYMENT_M) |        +--------------------+
         │ 1                  | idempotency_key    |
         │                    +--------------------+
         ▼ *                            │ 1
+--------------------+                  │
| MerchantPreference |                  ▼ *
+--------------------+        +--------------------+        +--------------------+
| id (PK, UUID)      |        |    Transaction     |        |    LedgerEntry     |
| merchant_id (FK)   |        +--------------------+        +--------------------+
| primary_proc (VAR) |        | tx_id (PK, UUID)   | 1    * | entry_id (PK, UUID)|
| fallback_proc (VAR)|        | intent_id (FK)     |<------>| tx_id (FK)         |
| routing_rules(JSON)|        | card_token (FK)    |        | account_type (ENUM)|
+--------------------+        | processor_name(VAR)|        | debit_cents (BIG)  |
                              | processor_tx_id    |        | credit_cents (BIG) |
                              | status (AUTHORIZED)|        | created_at (TIME)  |
                              | updated_at (TIME)  |        +--------------------+
                              +--------------------+
```

---

## 3. Step 3: API Design (Public REST Endpoints)

### 3.1 Create Payment Intent (Merchant Server $\rightarrow$ Gateway)
Merchant initiates the payment flow server-to-server:
```http
POST /api/v1/payment_intents
Authorization: Bearer <merchant_secret_key>
X-Idempotency-Key: idemp_ord_998124_v1
Content-Type: application/json

Request Body:
{
  "amount": 4999,                     // $49.99 stored in integer cents
  "currency": "USD",
  "order_id": "ord_amz_9918237",
  "customer_id": "cust_4412",
  "payment_method_types": ["card", "upi"],
  "metadata": {
    "cart_items": 3
  }
}

Response: 201 Created
{
  "intent_id": "pi_991823-a1b2-c3d4",
  "client_secret": "pi_991823_secret_77af9912",
  "amount": 4999,
  "currency": "USD",
  "status": "REQUIRES_PAYMENT_METHOD",
  "created_at": "2026-10-18T15:00:00Z"
}
```

---

### 3.2 Initialize Checkout Session (Create Hosted Redirect Page)
Merchant creates an ephemeral session for the user's browser:
```http
POST /api/v1/checkout/sessions
Authorization: Bearer <merchant_secret_key>
Content-Type: application/json

Request Body:
{
  "intent_id": "pi_991823-a1b2-c3d4",
  "success_url": "https://merchant.com/order/success?id=ord_amz_9918237",
  "cancel_url": "https://merchant.com/order/cancel",
  "session_ttl_seconds": 600          // 10 minutes
}

Response: 200 OK
{
  "session_id": "cs_test_8812401",
  "redirect_url": "https://checkout.gateway.com/pay/cs_test_8812401",
  "expires_at": "2026-10-18T15:10:00Z"
}
```

---

### 3.3 Execute Payment (Hosted Checkout Frontend $\rightarrow$ Checkout Backend)
User clicks "Pay Now" on the gateway-hosted page. Raw card credentials go exclusively to the gateway:
```http
POST /api/v1/checkout/pay
Content-Type: application/json

Request Body:
{
  "session_id": "cs_test_8812401",
  "cardholder_name": "Alex Mercer",
  "card_number": "4111111111114444",
  "exp_month": 12,
  "exp_year": 2028,
  "cvv": "123"
}

Response: 202 Accepted
{
  "status": "PROCESSING",
  "transaction_id": "tx_4412-9988",
  "poll_url": "https://checkout.gateway.com/api/v1/checkout/status/tx_4412-9988"
}
```

---

### 3.4 Query Transaction Status
```http
GET /api/v1/transactions/{transaction_id}
Authorization: Bearer <merchant_secret_key>

Response: 200 OK
{
  "transaction_id": "tx_4412-9988",
  "intent_id": "pi_991823-a1b2-c3d4",
  "amount": 4999,
  "currency": "USD",
  "status": "SETTLED",                // "AUTHORIZED" | "SETTLED" | "FAILED"
  "card_details": {
    "brand": "VISA",
    "last4": "4444",
    "bin": "411111",
    "exp_month": 12,
    "exp_year": 2028
  },
  "settled_at": "2026-10-18T15:00:03Z"
}
```

---

## 4. Step 4: High-Level Architecture

```
[ Customer Browser ]                     [ Merchant Server (Amazon) ]
        │                                             │
        │ 2. Redirect to Hosted URL                   │ 1. POST /payment_intents
        │                                             │    POST /checkout/sessions
        ▼                                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API Gateway & Load Balancer                         │
│               (SSL Termination, Auth, Ingress Rate Limiting)                │
└─────────────────────────────────────────────────────────────────────────────┘
   │                           │                               │
   │ (Host Web Page)           │ (Pay: Card Raw Data)          │ (Create Intent/Session)
   ▼                           ▼                               ▼
┌──────────────────┐  ┌──────────────────┐           ┌──────────────────┐
│ Checkout Frontend│  │ Checkout Backend │           │ Payment Intent & │
│ (Hosted UI HTML) │  │ Service          │           │ Session Service  │
└──────────────────┘  └──────────────────┘           └──────────────────┘
                               │                               │
                      Validate │                               ▼
                       Session └──────────────────────> ┌──────────────┐
                                                        │ Redis Cache  │
                                                        │(Session TTL) │
                                                        └──────────────┘
                               │
               (PCI Zone)      ▼ (mTLS TLS 1.3)
         ┌──────────────────────────────────────────────┐
         │          Card Tokenization Vault             │
         │  - Card Fingerprint Generator (BIN + Last4)  │
         │  - Hardware Security Module (HSM Key Mgmt)   │
         │  - Token Store DB (Encrypted PAN)            │
         └──────────────────────────────────────────────┘
                               │ (Returns: tok_card_4444)
                               ▼
         ┌──────────────────────────────────────────────┐
         │         Payment Orchestrator Service         │
         │  - Check Merchant Routing Preferences        │
         │  - Save Initial State: Transaction DB (ACID) │
         └──────────────────────────────────────────────┘
                               │
             ┌─────────────────┴─────────────────┐
             ▼                                   ▼
┌─────────────────────────┐         ┌─────────────────────────┐
│  Razorpay Connector     │         │     PayU Connector      │
│  (Adapter Pattern)      │         │   (Adapter Pattern)     │
└─────────────────────────┘         └─────────────────────────┘
             │                                   │
             └─────────────────┬─────────────────┘
                               ▼
            ┌───────────────────────────────────────┐
            │  External Payment Processor Gateway   │
            │     (Visa / Mastercard / Banks)       │
            └───────────────────────────────────────┘
                               │
                               ▼ (Async Callbacks)
            ┌───────────────────────────────────────┐
            │     Callback Collector Service        │
            └───────────────────────────────────────┘
                               │
                               ▼
            ┌───────────────────────────────────────┐
            │         Kafka Message Broker          │
            │  - Topic: 'payment.callback.status'   │
            │  - Topic: 'payment.settlement.daily'  │
            └───────────────────────────────────────┘
                     │                     │
                     ▼                     ▼
          ┌─────────────────────┐ ┌─────────────────────┐
          │ Status Update Worker│ │Reconciliation Worker│
          │ (Updates Tx DB)     │ │(Double-Entry Ledger)│
          └─────────────────────┘ └─────────────────────┘
```

---

## 5. Step 5: Low-Level Design & Deep Dives

---

### 5.1 Deep Dive 1: The 3-Step Lifecycle & PCI-DSS Scope Reduction

A major focus of Bunny's breakdown is why modern payment gateways strictly separate payment execution into **three distinct phases**:
1. **Phase 1: Intent Creation**: Merchant requests permission to collect money. Gateway records intent metadata.
2. **Phase 2: Session Establishment**: Gateway provisions a secure temporary session in Redis with a 10-minute TTL and returns a unique hosted URL.
3. **Phase 3: Card Execution**: The customer's browser loads the gateway-hosted page directly.

#### Why the Merchant Must Never Host the Card Input Fields
- Under **PCI-DSS (Payment Card Industry Data Security Standard)** regulations, any server that touches, processes, or transmits raw 16-digit Primary Account Numbers (PANs) is subject to extensive, multi-million-dollar annual security audits, forensic logging, and air-gapped physical server requirements.
- By rendering an **isolated hosted web page** (or Stripe-style `Elements` iframe injected from the gateway domain), the merchant application is completely isolated from the raw credit card data.
- **The Result**: Merchant compliance drops to a simple self-assessment questionnaire (**SAQ-A**), drastically lowering security risk and operational overhead.

---

### 5.2 Deep Dive 2: Card Tokenization & Hardware Security Module (HSM) Vault

When raw credit card data arrives at the Checkout Backend, it enters the **PCI-DSS Enclave**:

```
[ Raw Card Payload: "4111 1111 1111 4444", EXP: 12/28, CVV: 123, Name: "Alex Mercer" ]
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PCI-DSS Isolated Vault                             │
│                                                                                 │
│ 1. Card Fingerprinting:                                                         │
│    SHA-256(BIN: "411111" + Last4: "4444" + Exp: "1228" + Name: "Alex Mercer")   │
│    Enables duplicate card detection and velocity checks without decrypting PAN. │
│                                                                                 │
│ 2. Hardware Security Module (HSM) Envelope Encryption:                          │
│    - Master Key resides inside tamper-resistant physical HSM chip.              │
│    - Card PAN encrypted with unique Data Encryption Key (DEK) via AES-256-GCM.  │
│    - CVV is NEVER stored anywhere (PCI-DSS violation); discarded post-auth.     │
│                                                                                 │
│ 3. Token Generation:                                                            │
│    Generates opaque UUID token: "tok_card_9981240192"                           │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                      Returns: "tok_card_9981240192"
```

#### Token Vault Database Schema (PostgreSQL in Isolated Subnet)
```sql
CREATE TABLE card_vault (
    card_token VARCHAR(64) PRIMARY KEY,
    card_fingerprint VARCHAR(64) NOT NULL UNIQUE,
    encrypted_pan BYTEA NOT NULL,             -- Encrypted via HSM AES-256-GCM
    dek_encrypted_by_kek BYTEA NOT NULL,      -- Key encryption key
    bin_number VARCHAR(6) NOT NULL,           -- First 6 digits (Bank Routing)
    last4 VARCHAR(4) NOT NULL,                -- Last 4 digits (Receipt display)
    exp_month SMALLINT NOT NULL,
    exp_year SMALLINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_vault_fingerprint ON card_vault (card_fingerprint);
```

---

### 5.3 Deep Dive 3: Idempotency Engine & Double-Charge Prevention

In distributed payment systems, network timeouts between merchant, gateway, and bank are common. If a network drops during transmission, the client automatically retries.
Without strict idempotency, **the customer would be debited twice**.

```
[ Inbound Request with X-Idempotency-Key: "idemp_99812" ]
                         │
                         ▼
┌─────────────────────────────────────────────────┐
│ Redis Distributed Atomic Lock:                  │
│ SET idempotency:idemp_99812 "LOCKED" NX EX 120  │
└─────────────────────────────────────────────────┘
       /                                     \
(Lock Acquired)                       (Key Already Exists)
      │                                       │
      ▼                                       ▼
Execute Transaction                   Read Transaction Record
Write result to DB                    from PostgreSQL DB
Release Redis Lock with               Return Cached Response
cached response payload               (HTTP 200 OK)
```

#### Redis Lua Script for Atomic Idempotency Check
```lua
-- KEYS[1]: idempotency key, ARGV[1]: lock_value, ARGV[2]: ttl_seconds
if redis.call("EXISTS", KEYS[1]) == 1 then
    return redis.call("GET", KEYS[1]) -- Return existing status/payload
else
    redis.call("SET", KEYS[1], ARGV[1], "EX", ARGV[2])
    return "ACQUIRED"
end
```

---

### 5.4 Deep Dive 4: Multi-Processor Smart Routing & Adapter Design Pattern

Different merchants negotiate different fee structures across processors. For example:
- Merchant A routes domestic transactions through Razorpay (1.5% fee).
- Merchant A routes international USD transactions through Stripe (2.9% fee).
- If Razorpay health checks drop below 90%, fallback automatically routes traffic to PayU.

```
                   ┌──────────────────────────────────┐
                   │    Payment Orchestrator Core     │
                   └──────────────────────────────────┘
                                    │
                         Consults Routing Rules:
                         (Currency, Merchant, Health)
                                    │
                                    ▼
                 ┌──────────────────────────────────────┐
                 │       <<interface>> ProcessorAdapter │
                 │  + authorize(PaymentPayload): Result │
                 │  + capture(PaymentPayload): Result   │
                 │  + refund(PaymentPayload): Result    │
                 └──────────────────────────────────────┘
                     ▲              ▲              ▲
                     │              │              │
      ┌──────────────┴───┐  ┌───────┴──────┐  ┌────┴─────────────┐
      │ RazorpayAdapter  │  │ PayUAdapter  │  │  StripeAdapter   │
      └──────────────────┘  └──────────────┘  └──────────────────┘
```

#### Adapter Implementation (Java/Go Mental Model)
```go
type PaymentRequest struct {
    AmountCents  int64
    Currency     string
    CardToken    string
    MerchantRef  string
}

type ProcessorAdapter interface {
    Authorize(ctx context.Context, req PaymentRequest) (*AuthorizationResult, error)
    Capture(ctx context.Context, txId string) (*CaptureResult, error)
}
```

---

### 5.5 Deep Dive 5: Asynchronous Callback Pipeline & Reconciliation Ledger

A payment gateway **never assumes a transaction is finalized on the synchronous HTTP response**.
Network drops and bank delays mean authorization is merely an intent to move funds.

```
                                    [ Downstream Processor / Bank ]
                                                   │
                                     Asynchronous  │ Webhook Callback
                                                   ▼
                                    [ Callback Collector Service ]
                                                   │
                                                   ▼
                                    ┌─────────────────────────────┐
                                    │  Kafka: 'payment.callbacks' │
                                    └─────────────────────────────┘
                                                   │
                                                   ▼
                                    ┌─────────────────────────────┐
                                    │   Status Update Consumer    │
                                    └─────────────────────────────┘
                                                   │
                                                   ▼
                                    ┌─────────────────────────────┐
                                    │ PostgreSQL Transactions DB  │
                                    │ UPDATE status = 'SETTLED'   │
                                    └─────────────────────────────┘
                                                   │
                                                   ▼
                                    [ Forward Merchant Webhook ]
```

#### Daily Settlement & Double-Entry Bookkeeping Ledger
Every night (T+1), acquiring banks and card processors export settlement settlement batch files (CSV/ISO 8583).
A **Reconciliation Worker** matches processor settlement lines against the internal `transactions` table.
To ensure mathematical auditing integrity, all funds movements are recorded in an **Immutable Double-Entry Ledger**:

$$\sum \text{Debits} - \sum \text{Credits} = 0$$

```sql
-- Sample Ledger entry for a $100.00 purchase with $2.00 gateway fee
BEGIN;
-- 1. Debit Cash in Transit (Asset increases)
INSERT INTO ledger_entries (tx_id, account, debit_cents, credit_cents)
VALUES ('tx_9981', 'ASSET_CASH_IN_TRANSIT', 10000, 0);

-- 2. Credit Merchant Payable (Liability increases)
INSERT INTO ledger_entries (tx_id, account, debit_cents, credit_cents)
VALUES ('tx_9981', 'LIABILITY_MERCHANT_PAYABLE', 0, 9800);

-- 3. Credit Fee Revenue (Equity/Revenue increases)
INSERT INTO ledger_entries (tx_id, account, debit_cents, credit_cents)
VALUES ('tx_9981', 'REVENUE_GATEWAY_FEES', 0, 200);
COMMIT;
```

---

## 6. Common Interview Follow-Ups & Cross-Questions (Deep Dive)


### Q1: Why use tokenization instead of storing card numbers?

PCI-DSS compliance requirement. Storing raw card data requires Level 1 PCI compliance which is extremely expensive and complex. Tokenization:

- **Point 1**: Reduces PCI scope - only tokenization service needs full compliance,

- **Point 2**: Security - tokens are useless if leaked, real card data stays in HSM-protected vault,

- **Point 3**: Flexibility - same token works across processors. Tokens are one-way mapped, cannot be reversed without HSM access.

---

### Q2: How do you handle concurrent payment attempts with the same card?

Use database-level locking with idempotency keys. Flow:

- **Point 1**: Client generates unique idempotency key (UUID),

- **Point 2**: Backend checks UNIQUE constraint on idempotency_key column,

- **Point 3**: First request acquires row lock and processes,

- **Point 4**: Concurrent requests with same key either wait or get cached response,

- **Point 5**: After processing, response cached in Redis for 24h. This prevents double-charging even if user clicks 'Pay' button multiple times.

---

### Q3: What happens if payment processor API fails midway?

Multi-layer failure handling:

- **Point 1**: Detection - timeout (30s) or HTTP 5xx triggers failure,

- **Point 2**: Circuit Breaker - after 5 consecutive failures, route to backup processor for 1 minute,

- **Point 3**: Transaction State - mark as 'pending_verification' instead of 'failed',

- **Point 4**: Reconciliation - daily job queries processor for transaction status,

- **Point 5**: Fallback - if processor down, queue payment in Kafka for later processing,

- **User Experience - show 'Payment pending, will update via email' instead of hard failure. Recovery SLA**:
  99% resolved within 1 hour via automated reconciliation.

---

### Q4: How to ensure payment data durability and prevent loss?

Multiple durability layers:

- **Point 1**: Database - PostgreSQL with synchronous replication to 2 standby replicas (RPO=0),

- **Point 2**: WAL Archival - continuous archiving to S3 with point-in-time recovery,

- **Point 3**: Kafka - payment events stored with replication factor=3, retention=7 days,

- **Point 4**: Processor Records - external processors maintain independent records for reconciliation,

- **Point 5**: Backup - daily encrypted snapshots to cold storage (30 day retention). Even if entire primary datacenter fails, zero transaction data loss guaranteed.

---

### Q5: How does the reconciliation process work?

Daily batch reconciliation:

- **Point 1**: Fetch - pull previous day's transactions from internal DB (status=succeeded),

- **Point 2**: Download - fetch settlement files from each processor (Visa, Mastercard, Razorpay),

- **Point 3**: Match - join on processor_txn_id,

- **Point 4**: Compare - check amount, status, timestamp differences,

- **Detect - flag discrepancies**:
  missing in processor file, amount mismatch (>1%), status conflict,

- **Point 6**: Alert - notify finance team for manual review,

- **Resolve - automatic retry for missing transactions after 24h. Typical match rate**:
  99.8%, discrepancies resolved within 48h.

---

### Q6: What if the database goes down during payment processing?

High availability setup:

- **Point 1**: Primary-Standby - PostgreSQL with streaming replication, automatic failover via Patroni in <30s,

- **Point 2**: Health Checks - load balancer pings DB every 2s, routes to standby if primary unhealthy,

- **Point 3**: In-Flight Transactions - payments in 'processing' state are idempotent, can be retried safely,

- **Point 4**: Kafka Buffer - if DB fully down, queue payment requests in Kafka (retention=7 days) for processing when DB recovers,

- **Graceful Degradation - return 'Payment queued, will process shortly' instead of hard error. RTO**:
  30 seconds, RPO: 0 seconds (no data loss).

---

### Q7: How do you handle quota/rate limits per merchant?

Multi-tier rate limiting:

- **Point 1**: API Gateway - token bucket algorithm, 1000 req/min per merchant API key stored in Redis,

- **Point 2**: Circuit Breaker - if merchant hits limit, return 429 with Retry-After header,

- **Point 3**: Quota Enforcement - monthly transaction volume limits checked before processing (stored in merchant config table),

- **Point 4**: Soft Limits - warning at 80% quota via email, hard stop at 100%,

- **Sliding Window - use Redis sorted sets with timestamp scoring for accurate window-based counting. Implementation**:
  Zadd merchant:{id}:requests {timestamp} {request_id}, Zremrangebyscore to remove old entries, Zcard to count current window.

---

### Q8: What's the optimal session timeout and why?

30 minutes with reasoning:

- **Point 1**: User Behavior - 95% of checkouts complete within 5 minutes, 30 min covers edge cases (phone interruptions, OTP delays),

- **Point 2**: Security - limits window for session hijacking attacks,

- **Point 3**: Memory - Redis eviction policy (LRU) prevents unbounded growth,

- **UX Trade-off - too short (5 min) = user frustration on slow forms, too long (2 hours) = stale cart data, security risk. Implementation**:
  Redis TTL set on session creation, auto-extends by 10 min on each activity (view payment page, enter card), maximum 30 min from first creation. Alternative: Shorter timeout (10 min) for high-value transactions >$1000.

---

### Q9: How do you prevent replay attacks on payment APIs?

Multiple layers:

- **Point 1**: Idempotency Keys - each payment request must include unique key, stored with TTL=24h, replayed requests return cached response without charging,

- **Point 2**: Timestamp Validation - request must include timestamp, reject if >5 minutes old,

- **Point 3**: HMAC Signature - client signs request with secret key, server validates signature to ensure integrity,

- **Point 4**: Nonce - one-time random value included in signature, stored in Redis (TTL=10 min) to prevent reuse,

- **HTTPS - TLS 1.3 prevents MITM attacks. Full signature scheme**:
  HMAC-SHA256(api_key + timestamp + nonce + request_body) sent in X-Signature header.

---

### Q10: How do you handle refunds and partial refunds?

Refund workflow:

- **Point 1**: Validation - check original payment status=succeeded, refund amount ≤ remaining balance,

- **Point 2**: Idempotency - refund requests also use idempotency keys to prevent double-refunds,

- **Point 3**: Processor Call - call processor's refund API with original transaction_id,

- **Point 4**: State Update - create Refund record linked to Transaction, update remaining_refundable_amount,

- **Point 5**: Settlement - refunds processed in next day's settlement cycle, not instant,

- **Partial Support - track refund history**:
  original_amount=100, refunded=[30, 20] = remaining=50. Typical refund processing time: 5-10 business days (processor dependent).

---

## 7. Architectural Trade-Off Analysis

| Dimension | Chosen Strategy | Alternative Considered | Trade-Off & Justification |
| :--- | :--- | :--- | :--- |
| **Card Data Entry** | Hosted Gateway URL / Isolated iframe | Merchant Captures Card Direct | Reduces merchant PCI scope from complex SAQ-D to minimal SAQ-A; isolates card credentials entirely inside gateway enclave. |
| **Consistency vs. Avail** | Strong Consistency (CP) | Eventual Consistency (AP) | Double charging customers or losing transaction records destroys business trust; strict relational ACID required. |
| **Session Storage** | In-Memory Redis Cluster (TTL 10m) | Relational Database Table | Redis provides sub-millisecond session validation; auto-expiring keys eliminate expensive database deletion sweeps. |
| **Processor Integration**| Adapter Design Pattern | Hardcoded Direct API Calls | Allows zero-downtime additions of new processors and dynamic failover without modifying core payment logic. |
| **Accounting Model** | Double-Entry Immutable Ledger | Single Balance Column Increment | Double-entry guarantees cryptographic mathematical balance; provides full audit trail for regulatory compliance. |

---

## 8. Failure Modes & Resiliency Strategies

1. **Processor Outage During Authorization**:
   - The Orchestrator monitors downstream error rates via a **Circuit Breaker** (Netflix Hystrix / Resilience4j pattern).
   - If Razorpay returns consecutive 5xx errors for $> 10\text{ seconds}$, the circuit opens, automatically rerouting inflight requests to PayU.
2. **Double Click / Thundering Client Retries**:
   - The UI disables the "Pay" button immediately upon click.
   - At the backend, Redis atomic distributed locks reject any concurrent payload bearing the same `session_id` or `idempotency_key` with HTTP `409 Conflict`.
3. **Database Network Partition During Authorization**:
   - If the relational database connection drops before committing the authorization state, the orchestrator immediately triggers an automated **Reversal / Void** API call to the downstream processor, preventing customer accounts from being held in pending debit without an order.
