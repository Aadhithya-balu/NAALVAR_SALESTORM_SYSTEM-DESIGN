# SALESTORM — Requirements & Assumptions Document
### High-Scale E-Commerce Flash Sale
**SYSCRAFTERS 2026 — Design-First, AI-Assisted System Design Hackathon**

---

## Document Control & Metadata

| Attribute | Details |
| :--- | :--- |
| **Document Title** | SALESTORM — Requirements & Assumptions Document |
| **Project** | SYSCRAFTERS 2026 Hackathon (Design-First, AI-Assisted System Design) |
| **Document Version** | 1.0.0 |
| **Status** | Approved Architectural Baseline |
| **Target Audience** | System Architects, Lead Engineers, Domain Reviewers |
| **Downstream Deliverables** | `02_HLD`, `03_LLD`, `04_Database`, `05_API`, `08_Scalability_Reliability`, `09_Security_Observability`, `10_ADR` |

---

## 1. Business Problem

### 1.1 Context & Core Business Question
SALESTORM is an enterprise-grade, high-volume e-commerce platform designed to orchestrate high-velocity flash-sale events. During these events, a massive influx of concurrent customers converges on an extremely scarce catalog of high-demand items.

The central architectural and business challenge is formulated as:
> *"How can we handle thousands of simultaneous purchase requests for limited inventory without overselling, while keeping payment and order processing reliable, consistent, and idempotent under adverse network and service failure conditions?"*

### 1.2 The Critical Flash-Sale Scenario
The system design is anchored around a benchmark critical scenario:
* **Concurrent Contenders:** $10,000$ active buyers submitting purchase requests concurrently.
* **Available Stock:** Exactly $100$ physical inventory units available for allocation.
* **Inventory Invariant:** Zero overselling permitted under any concurrency or failure condition ($\text{Total Reserved} \le 100$).
* **Transaction Safety:** Absolute prevention of duplicate reservations, double charges, and duplicate orders.
* **Lifecycle Resilience:** Temporary inventory reservations must auto-expire upon customer abandonment or payment timeout, safely returning stock to the pool.
* **Fault Tolerance:** End-to-end consistency across inventory, payment, and order state machines even when intermediate downstream services experience prolonged outages.

### 1.3 High-Level Business Pipeline
The customer journey follows an end-to-end business pipeline spanning discovery through fulfillment:

```
Customer
  │
  ▼
[ Product Discovery ] ──► [ Cart Management ]
                               │
                               ▼
                    [ Inventory Check ]
                               │
                               ▼
                 [ Inventory Reservation ]
                               │
                               ▼
                      [ Checkout Flow ]
                               │
                               ▼
                     [ Payment Gateway ]
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
     (Payment Succeeded)                   (Payment Failed/Timeout)
            │                                     │
            ▼                                     ▼
     [ Order Creation ]                   [ Release Reservation ]
            │                                     │
            ▼                                     ▼
     [ Fulfillment ]                       (Stock Restored)
            │
            ▼
     [ Shipment & Logistics ]
            │
            ▼
     [ Customer Notification ]
            │
            ▼
     [ Delivery Tracking ]
```

---

## 2. System Objectives

The SALESTORM platform must achieve the following eleven core architectural objectives:

1. **Traffic Surge Elasticity:** Ingest and arbitrate massive traffic spikes gracefully without system degradation or cascading outages.
2. **Strict Inventory Allocation:** Guarantee mathematical non-overselling ($\text{Stock} \ge 0$) under extreme write contention.
3. **Deterministic Temporary Reservations:** Implement time-bounded inventory holding to balance high conversion with stock starvation prevention.
4. **Automated Compensation & Reclamation:** Reclaim reserved stock reliably upon checkout abandonment, client timeouts, or payment failures.
5. **Universal Transaction Idempotency:** Guarantee that repeated invocations across network retries or client double-submissions produce exactly one side effect.
6. **Duplicate Transaction Immunity:** Prevent concurrent or sequential duplicate reservation attempts and duplicate payment attempts from the same principal.
7. **Deterministic Order Lifecycle:** Maintain strict, auditable state machine transitions across all order stages.
8. **Asynchronous Fault Recovery:** Recover state smoothly from partial network partitions, downstream payment timeouts, and service outages without human intervention or data corruption.
9. **Horizontal Scalability:** Ensure compute and read tiers scale horizontally across nodes without introducing shared-memory bottlenecks.
10. **Zero-Trust Observability & Defense:** Ensure end-to-end distributed tracing, metrics, audit trails, mutual TLS, and rate-limiting at network perimeters.
11. **Production-Grade Blueprint Clarity:** Provide an architectural design that is unambiguous, operationally defensible, and directly implementable by engineering teams.

---

## 3. Functional Requirements

### 3.1 Customer & Product Discovery
* **FR-DISC-001 (Catalog Access):** The system shall allow unauthenticated and authenticated users to browse, search, and retrieve product catalog details.
* **FR-DISC-002 (Product Detail Retrieval):** The system shall return metadata including title, description, imagery, price, and active sale window schedules.
* **FR-DISC-003 (Availability Visibility):** The system shall expose stock availability indicators (e.g., `IN_STOCK`, `LOW_STOCK`, `SOLD_OUT`). Availability data in the discovery path may be eventually consistent with bounded staleness to decouple catalog browsing from core transaction databases.

### 3.2 Cart Management
* **FR-CART-001 (Cart Mutation):** The system shall enable authenticated customers to add, modify quantity of, or remove catalog items from their persistent cart.
* **FR-CART-002 (Cart State Persistence):** The system shall maintain cart state across client sessions.
* **FR-CART-003 (Soft Decoupling):** Adding an item to the shopping cart shall *not* reserve inventory. Cart state represents customer intent, not an inventory reservation.

### 3.3 Inventory Management
* **FR-INV-001 (Atomic Stock Check):** The system shall evaluate real-time available inventory prior to executing reservation requests.
* **FR-INV-002 (Atomic Stock Reservation):** The system shall execute atomic reservation decrements against verified stock balances.
* **FR-INV-003 (Permanent Reservation Confirmation):** The system shall permanently convert a temporary reservation to a finalized sold state upon receipt of a verified payment confirmation.
* **FR-INV-004 (Explicit Reservation Release):** The system shall release reserved stock back into the available pool immediately upon payment rejection or client cancellation.
* **FR-INV-005 (Automated Expiry Release):** The system shall detect unconfirmed reservations whose time-to-live (TTL) has elapsed and reclaim the stock.
* **FR-INV-006 (Strict Non-Negative Invariant):** Under no condition—including race conditions, concurrent node operations, or replay attacks—shall the inventory balance drop below zero ($Inventory \ge 0$).

### 3.4 Reservation Lifecycle
The system shall enforce a deterministic reservation state machine with unambiguous valid transitions and rigid failure paths.

```
       [ AVAILABLE ]
             │
             │ (1) Reserve Stock (Atomic Decrement)
             ▼
        [ RESERVED ]
             │
             │ (2) Initiate Payment Handshake
             ▼
    [ PAYMENT_PENDING ]
             │
      ┌──────┴──────────────────────────────┐
      │ (3a) Payment Succeeded             │ (3b) Payment Failed / Timeout / Expired
      ▼                                     ▼
 [ CONFIRMED ]                         [ RELEASED ]
      │                                     │
      │ (4) Fulfillment Initiated           │ (Stock Returned to Available Pool)
      ▼                                     ▼
   [ SOLD ]                           [ AVAILABLE ]
```

#### Valid State Transitions:
1. `AVAILABLE` $\rightarrow$ `RESERVED`: Initiated when customer enters checkout with valid stock available.
2. `RESERVED` $\rightarrow$ `PAYMENT_PENDING`: Initiated when checkout dispatches the payment authorization request to the payment gateway.
3. `PAYMENT_PENDING` $\rightarrow$ `CONFIRMED`: Triggered when the payment provider issues a cryptographically verified success notification.
4. `CONFIRMED` $\rightarrow$ `SOLD`: Triggered when the confirmed order transitions into the physical fulfillment pipeline.

#### Valid Failure & Compensation Transitions:
* **Failure Path A (Payment Failure):** `RESERVED` / `PAYMENT_PENDING` $\rightarrow$ `PAYMENT_FAILED` $\rightarrow$ `RELEASED` $\rightarrow$ `AVAILABLE`.
* **Failure Path B (TTL Expiration / Timeout):** `RESERVED` / `PAYMENT_PENDING` $\rightarrow$ `TIMEOUT` $\rightarrow$ `RELEASED` $\rightarrow$ `AVAILABLE`.

#### Prohibited Transitions:
* `RELEASED` $\rightarrow$ `CONFIRMED` (Illegal: Expired or released stock cannot be confirmed).
* `SOLD` $\rightarrow$ `AVAILABLE` (Illegal: Sold inventory cannot be reclaimed without a dedicated return/refund business process).
* `AVAILABLE` $\rightarrow$ `CONFIRMED` (Illegal: Direct confirmation without active reservation bypasses concurrency controls).

### 3.5 Checkout Flow
* **FR-CHK-001 (Checkout Session Initiation):** The system shall create an immutable Checkout Session linking customer ID, product items, pricing snapshots, and shipping addresses.
* **FR-CHK-002 (Active Reservation Binding):** The system shall verify that an active, non-expired reservation token is bound to the Checkout Session prior to initiating payment.
* **FR-CHK-003 (Downstream Payment Dispatch):** The system shall generate a secure, idempotent payment transaction token and hand off execution to the Payment Gateway interface.

### 3.6 Payment Processing
* **FR-PAY-001 (Payment Success Handling):** Upon receiving synchronous or asynchronous payment authorization success, the system shall atomically mark the payment record as `SETTLED` and trigger order confirmation.
* **FR-PAY-002 (Payment Failure Handling):** Upon receiving definitive payment rejection (e.g., insufficient funds, fraud flag), the system shall immediately mark the payment as `FAILED` and trigger inventory release.
* **FR-PAY-003 (Payment Timeout Resolution):** In the event of gateway timeouts or missing callbacks, the payment status shall enter `UNKNOWN_PENDING` and initiate automated polling or webhook reconciliation.
* **FR-PAY-004 (Deterministic Retry Policy):** Retries against payment gateways shall reuse the identical idempotency key to prevent double charging.
* **FR-PAY-005 (Payment Reconciliation):** An out-of-band reconciliation mechanism shall audit lingering `PAYMENT_PENDING` records against external payment gateway settlement logs.
* **FR-PAY-006 (Duplicate Payment Prevention):** Concurrent or sequential payment execution requests bearing the same checkout or transaction identifier shall be rejected or deduplicated at the boundary.

### 3.7 Order Lifecycle
The system shall manage the customer order via an auditable, append-only or transition-checked order state machine:

```
[ CREATED ] ──► [ PAYMENT_PENDING ] ──► [ CONFIRMED ] ──► [ PROCESSING ]
                                              │
                                              ▼
                                         [ SHIPPED ] ──► [ OUT_FOR_DELIVERY ] ──► [ DELIVERED ]
```

* **FR-ORD-001 (Creation):** Orders are created in `CREATED` status upon checkout submission.
* **FR-ORD-002 (Payment Binding):** Order advances to `PAYMENT_PENDING` while awaiting gateway settlement.
* **FR-ORD-003 (Order Confirmation):** Order advances to `CONFIRMED` only upon verified payment confirmation and confirmed inventory allocation.
* **FR-ORD-004 (Downstream State Progression):** Downstream logistics advance the order strictly through `PROCESSING` $\rightarrow$ `SHIPPED` $\rightarrow$ `OUT_FOR_DELIVERY` $\rightarrow$ `DELIVERED`.
* **FR-ORD-005 (State Transition Guards):** Any out-of-sequence event (e.g., receiving `SHIPPED` before `CONFIRMED`, or moving `CANCELLED` to `DELIVERED`) shall be rejected with an audit alert.

### 3.8 Fulfillment & Shipment
* **FR-FUL-001 (Fulfillment Trigger):** The fulfillment pipeline shall be triggered strictly upon receipt of an immutable `OrderConfirmedEvent`.
* **FR-FUL-002 (Shipment Lifecycle Tracking):** The system shall track external logistics carrier milestones (label generated, picked up, in-transit, out for delivery, delivered).
* **FR-FUL-003 (Asynchronous Decoupling):** Fulfillment and shipment processing shall operate asynchronously from the core purchase checkout loop.

### 3.9 Notifications
* **FR-NOTIF-001 (Event-Driven Triggers):** The notification service shall publish customer-facing communications (Email, SMS, Push) for key lifecycle milestones (`Order Confirmed`, `Payment Failed`, `Item Shipped`, `Out for Delivery`).
* **FR-NOTIF-002 (Non-Blocking Guarantee):** All notification processing shall be strictly asynchronous and decoupled via message broker topics. Notification delivery delays or vendor failures must have zero impact on inventory reservation, payment processing, or order creation.

### 3.10 Universal Idempotency
* **FR-IDEM-001 (Idempotency Key Specification):** All mutating requests across the purchase flow (Reservation, Checkout, Payment, Order Creation) must mandate a client-supplied or gateway-generated unique idempotency token (`Idempotency-Key`).
* **FR-IDEM-002 (Deduplication Enforcement):** The receiving service boundary shall record processed idempotency keys. Re-executing an operation with an active or already processed key shall return the original cached response without re-executing business logic or state mutations.
* **FR-IDEM-003 (Key Scope & Collision Avoidance):** Idempotency keys shall be scoped by tenant/user and operation type, with deterministic expiration windows matching transaction lifecycles.

---

## 4. Non-Functional Requirements (NFRs)

The following measurable matrix defines the non-functional criteria governing the SALESTORM platform:

| Category | Requirement ID | Metric / Target | Architectural Constraint & Measurement Context |
| :--- | :--- | :--- | :--- |
| **Scalability** | `NFR-SCALE-001` | **Base Throughput:** 10,000 req/sec | System handles steady-state browsing, cart additions, and account activity with zero performance degradation. |
| **Scalability** | `NFR-SCALE-002` | **Peak Flash Throughput:** Up to 500,000 req/sec | Edge, ingress routing, caching, and queue buffering tiers must absorb up to 500k req/sec peak surge during flash launch. |
| **Scalability** | `NFR-SCALE-003` | **Horizontal Scaling:** Linear compute elasticity | Stateless application services must scale horizontally by adding instances behind load balancers with no shared memory dependency. |
| **Concurrency** | `NFR-CONC-001` | **Simultaneous Contenders:** 10,000 requests | The system must arbitrate 10,000 simultaneous purchase requests hitting the same SKU without resource starvation or deadlocks. |
| **Concurrency** | `NFR-CONC-002` | **Allocation Boundary:** Exactly $\le 100$ units | Across 10,000 concurrent attempts, exactly and only 100 units can be reserved; remaining 9,900 requests receive graceful sold-out responses. |
| **Consistency** | `NFR-CONS-001` | **Inventory Non-Negative:** Invariant ($Stock \ge 0$) | Strong transactional consistency at the reservation boundary. Zero tolerance for negative stock balances under any failure mode. |
| **Consistency** | `NFR-CONS-002` | **Eventual Consistency:** Downstream tiers | Catalog browsing, analytics, and notification projections may exhibit bounded eventual consistency ($< 2$ seconds). |
| **Reliability** | `NFR-REL-001` | **Payment Gateway Resilience** | 100% of payment timeouts and gateway drops must route to automated reconciliation or compensations; zero silent losses. |
| **Reliability** | `NFR-REL-002` | **Downstream Service Outage Resilience** | System buffers confirmed transactions if Order Service suffers a 30-second outage, recovering cleanly without data loss. |
| **Reliability** | `NFR-REL-003` | **Automated Stock Reclamation** | 100% of expired reservations must be returned to available inventory via reliable background sweeps or event TTLs. |
| **Availability** | `NFR-AVAIL-001` | **Critical Path Availability:** 99.99% | Discovery and ingress remain operational under surge; inventory boundary prioritizes correctness over raw write acceptance. |
| **Performance** | `NFR-PERF-001` | **Reservation Latency Target** | P99 latency target of $< 150\text{ ms}$ for inventory reservation operations under peak contention (design target, not SLA guarantee). |
| **Performance** | `NFR-PERF-002` | **Catalog Read Latency Target** | P95 latency target of $< 30\text{ ms}$ via multi-tier edge and distributed read caching. |
| **Security** | `NFR-SEC-001` | **Network & Channel Security** | Mandatory TLS 1.3 for all client-to-server and mTLS for all inter-service mesh communications. |
| **Security** | `NFR-SEC-002` | **Authentication & Authorization** | Cryptographic JWT verification, OAuth2/OIDC, and role-based access control (RBAC) across administrative and checkout APIs. |
| **Security** | `NFR-SEC-003` | **Perimeter Defense & Abuse Protection** | IP and user-based token bucket rate limiting, bot protection, and WAF rules at ingress to prevent script scraping and DDoS. |
| **Security** | `NFR-SEC-004` | **Payment Data Protection** | Strict PCI-DSS compliance boundaries. Cardholder data is tokenized; zero raw cardholder data stored on SALESTORM servers. |
| **Observability** | `NFR-OBS-001` | **Telemetry & Metrics** | Real-time monitoring of ingestion rates, P50/P90/P99 latencies, reservation drop rates, payment failure ratios, and queue depths. |
| **Observability** | `NFR-OBS-002` | **Distributed Tracing & Structured Logs** | End-to-end W3C distributed trace propagation (`traceparent`) linking edge ingress, reservation, payment, and order records. |

---

## 5. Strict Guarantees vs. Engineering Targets

To prevent architectural ambiguity, the table below establishes a strict boundary between non-negotiable invariants and operational design targets:

| Dimension | Strict Guarantees (Non-Negotiable Invariants) | Targets / Engineering Goals (Best-Effort Operational Goals) |
| :--- | :--- | :--- |
| **Inventory Allocation** | • **Zero Overselling:** Under no circumstances shall total confirmed + active reservations exceed available stock.<br>• **Non-Negative Stock:** Available inventory balance can never drop below zero ($Balance \ge 0$). | • High reservation conversion rate.<br>• Sub-second customer-facing rejection notices when stock reaches zero. |
| **Transaction Processing** | • **Universal Payment Deduplication:** An idempotency key can never initiate more than one charge on an external gateway.<br>• **Single Order per Checkout:** Duplicate order submissions produce the identical order record without duplicate fulfillment. | • Rapid payment round-trip processing.<br>• Minimal customer checkout abandonment. |
| **Lifecycle Integrity** | • **State Machine Determinism:** Illegal order/reservation transitions are blocked and logged.<br>• **Guaranteed Reclamation:** Expired or abandoned reservations must eventually be released back to the pool. | • Reservation expiry sweep latency within seconds of TTL breach. |
| **Fault Recovery** | • **No Payment Abandonment:** Confirmed payments are never dropped, even during downstream Order Service downtime.<br>• **Safe Inventory Compensation:** Failed payments never lock reserved inventory permanently. | • Automated recovery of backlog within 60 seconds of downstream service restoration. |
| **Scale & Traffic** | *None (System cannot guarantee unlimited traffic intake without perimeter shedding).* | • Sustain ~10,000 req/sec base traffic.<br>• Ingress architecture reasons about absorbing up to 500,000 req/sec flash spikes.<br>• Horizontal scale-out of stateless application nodes. |
| **Latency & Performance** | *None (Network physics and downstream banking gateways prevent absolute latency guarantees).* | • Sub-150ms P99 target for reservation write contention.<br>• Sub-30ms P95 target for cached product discovery. |
| **Availability** | • **Correctness Over Blind Availability:** Under extreme partition, the inventory boundary chooses consistency over accepting unverified writes. | • 99.99% uptime for public-facing discovery endpoints.<br>• Graceful degradation with queuing under extreme overload. |

---

## 6. Architecture Assumptions

The design of the SALESTORM platform is founded upon the following explicit architectural assumptions:

1. **Definitive Inventory Source of Truth:** A single, authoritative data store (or bounded transactional partition) holds ownership of real-time inventory counts and reservation states.
2. **Consistency Boundary Isolation:** Concurrency control and consistency enforcement occur strictly at the Inventory/Reservation service boundary, shielding downstream services from write contention.
3. **Deterministic Expiry Mechanism:** Every reservation possesses an explicit TTL. The architecture assumes an active or passive revocation mechanism will release unconfirmed allocations.
4. **Client Retries & Network Flakiness:** In high-traffic scenarios, clients will aggressively click buttons multiple times, and network drops will trigger automated client retries.
5. **Duplicate Ingress Traffic:** Network packet retransmissions and browser retries will introduce duplicate requests across the entire pipeline.
6. **External Payment Gateway Latency & Flakiness:** The external payment gateway is a third-party dependency subject to network latency, transient timeouts, intermittent rate limiting, and 5xx errors.
7. **Downstream Service Instability:** Downstream services (e.g., Order Service, Fulfillment Service, Notification Service) are assumed to experience transient outages (e.g., 30-second crash or restart loops).
8. **Asynchronous Decoupling for Durability:** Message-oriented middleware and asynchronous event streaming can be leveraged to buffer and decouple post-payment workflows safely.
9. **Stateless Service Tier:** All API routing and business logic services are stateless, externalizing state to distributed caches, transactional stores, and queues.
10. **Storage Transactional Guarantees:** The underlying persistence engine chosen for inventory and payments must support atomic compare-and-swap, ACID transactions, or distributed serializable isolation levels.
11. **Idempotency Everywhere:** Idempotency is not an optional optimization; it is a foundational prerequisite implemented at every state-mutating boundary.

---

## 7. System Constraints

The architecture must operate within the following boundaries:

* **Extreme Contention on Single SKU:** In flash-sale scenarios, contention is localized to a minuscule fraction of catalog keys (e.g., $10,000$ clients competing for $1$ SKU with $100$ items). Distributed partition sharding by SKU does not relieve hot-key contention on that single item.
* **Hard Upper-Bound Inventory:** Exactly 100 physical items exist in the warehouse for the critical scenario. No backordering, pre-ordering, or buffer buffers are permitted.
* **Zero Overselling Tolerance:** Overselling results in legal liability, brand damage, and expensive customer service compensations.
* **External Payment Black Box:** The system cannot control external payment gateway internal latency, internal queue backlogs, or bank clearing speeds.
* **Design-First Scope:** Hackathon deliverables prioritize architectural completeness, rigorous trade-off evaluations, mathematical correctness proofs, and state machine designs over raw boilerplate code generation.
* **Prototype Validation Rule:** Any exploratory code or prototype created in later phases must serve solely to benchmark or validate a specific architectural hypothesis (e.g., lock contention benchmarks, idempotency collision rates).

---

## 8. Traffic & Load Model

### 8.1 Load Profiles
The platform must transition smoothly between three operating regimes:

```
[ Base Load: ~10,000 req/sec ]
              │
              │  (Flash Sale Commences: 50x Surge)
              ▼
[ Ingress Flash Load: Up to 500,000 req/sec ]
              │
              │  (Perimeter Filtering, Edge Cache, Rate Limiting)
              ▼
[ Critical Inventory Contention Boundary: 10,000 concurrent req / single SKU ]
              │
              ▼
[ Successful Allocations: EXACTLY 100 ] ──► [ 9,900 Graceful Rejections ]
```

* **Base Traffic (Steady State):** $\approx 10,000\text{ req/sec}$ across catalog browsing, search, user profiles, and cart management. Read-to-write ratio $\approx 95:5$.
* **Flash-Sale Surge (Edge Ingress):** Architecture must reason about ingesting, filtering, and shedding traffic for surges reaching up to $500,000\text{ req/sec}$ at the edge gateway. Read-to-write ratio $\approx 80:20$.
* **Critical Flash Event (Hot-Key Boundary):** $10,000$ concurrent buyers execute checkout simultaneously against a single SKU with exactly $100$ units in stock.

### 8.2 The Contention Bottleneck
The core challenge in flash sales is not simple network bandwidth or stateless compute capacity; it is **hot-row write serialization**. 

When 10,000 requests attempt to decrement the identical counter concurrently, conventional row-level locking causes severe lock contention, thread starvation, connection pool exhaustion, and cascading database failure. The architecture must decouple request intake from the atomic inventory serialization boundary.

### 8.3 The Mathematical Invariants
Let:
* $I_0$ be the initial available inventory ($I_0 = 100$).
* $R(t)$ be the cumulative count of successful inventory reservations granted up to time $t$.
* $C(t)$ be the cumulative count of confirmed orders settled via payment up to time $t$.
* $E(t)$ be the cumulative count of expired or cancelled reservations reclaimed up to time $t$.
* $A(t)$ be the current available stock balance at time $t$.

The system strictly enforces the following invariant for all $t$:

$$A(t) = I_0 - [R(t) - E(t)] \ge 0$$

$$R(t) - E(t) \le I_0$$

$$C(t) \le I_0$$

For the benchmark critical scenario ($I_0 = 100$):

$$\text{Successful Active Reservations} \le 100$$
$$\text{Total Sold Items } (C) \le 100$$
$$\text{Total Oversold Items } \equiv 0$$

---

## 9. Critical Business Invariants

The following nine invariants must be maintained across all system states, failures, and network partitions:

1. **Inventory Non-Negativity:** Physical inventory balances must never drop below zero under any concurrency or race condition.
2. **Bounded Reservation Volume:** Total active reservations plus settled orders can never exceed the configured stock threshold.
3. **Singular Idempotency Mapping:** Exactly one business state change can be associated with a single idempotency key. Duplicate submissions must return identical output with zero side effects.
4. **Zero Double-Billing:** A customer transaction can never be submitted to the payment gateway more than once for a single checkout session.
5. **Single Order Guarantee:** A single checkout session and payment authorization cannot generate more than one confirmed order record.
6. **Guaranteed Stock Reclamation:** Any inventory held under a reservation that is abandoned, timed out, or associated with a failed payment must be returned to the available inventory pool.
7. **Strict State Machine Validity:** Order and reservation records can only transition through formally defined valid paths. Out-of-order and illegal transitions must fail closed.
8. **Durable Payment Settlement:** Once payment authorization succeeds, the order state must not be lost, even if the Order Service or database crashes immediately post-payment.
9. **No Ghost Inventory Locks:** Unsuccessful or aborted payment attempts must never permanently lock or isolate stock.

---

## 10. System Dependencies

### 10.1 Internal Service Dependencies

| Service / Component | Architectural Role | Critical Path Sync Dependency? | Failure Mode & Required Degradation Behavior |
| :--- | :--- | :--- | :--- |
| **Product Service** | Catalog metadata, media, and pricing. | **No** (Discovery only) | Serve from edge/regional read caches. If origin fails, serve stale cached metadata; disable price edits. |
| **Cart Service** | Holds pre-checkout customer intent. | **No** (Pre-checkout) | Client-side local storage backup; degraded cart save. Does not block direct flash-sale checkout. |
| **Sale Service** | Manages flash-sale schedules and eligibility. | **No** (Read-heavy) | Cached rule evaluation at edge/gateway. Fails closed if pricing or schedule validation cannot be verified. |
| **Inventory / Reservation Service** | Source of truth for atomic stock and reservations. | **YES (Critical Path)** | Highly resilient transactional boundary. If unavailable, checkout fails fast with "Try again" (never oversells). |
| **Checkout Service** | Orchestrates checkout validation and session tokens. | **YES (Critical Path)** | Returns structured error. Fails fast without executing partial transactions. |
| **Payment Service** | Interface to payment gateways and tokenization. | **YES (Critical Path)** | Enforces strict timeouts and idempotent retries. Never reports false success. |
| **Order Service** | Manages order creation and lifecycle state. | **NO (Asynchronous Post-Payment)** | If down, payment completion events are buffered in persistent message queues; recovered upon restart. |
| **Shipment Service** | Interfaces with 3PL logistics carriers. | **No** (Asynchronous) | Buffered in message broker; retryable background jobs. Zero impact on customer purchase flow. |
| **Notification Service** | Customer messaging (email, SMS, push). | **No** (Asynchronous) | Buffered in message broker; dropped or retried without impacting transactions. |
| **Primary Database** | Authoritative transactional persistence. | **YES (Critical Path)** | High availability with primary-replica failover. Fails closed on total loss. |
| **Distributed Cache** | Edge/read acceleration and hot-key buffering. | **Conditional** | Cache-aside for reads; if cache fails, system sheds load to protect database origin. |
| **Message Broker** | Asynchronous durable messaging between services. | **Conditional (Post-Payment)** | High-availability distributed log. If unavailable, publishers buffer locally or reject new checkouts safely. |

### 10.2 External Dependencies

| Dependency | Purpose | Critical Path Sync Dependency? | Failure Behavior & Mitigation |
| :--- | :--- | :--- | :--- |
| **Payment Gateway** | Card processing, digital wallets, bank authorization. | **YES** | Gateway timeout / outage handled via circuit breakers, idempotent status polling, and automated reconciliation. |
| **Logistics Carrier / 3PL** | Shipping label generation, manifest dispatch. | **No** | Asynchronous batch polling or webhook integration. Isolated from purchase path. |
| **Third-Party Notification Provider** | SMS/Email dispatch gateways (e.g., Twilio, SendGrid). | **No** | Asynchronous delivery queues with dead-letter queue (DLQ) support. |

---

## 11. Failure Modes & Required System Behavior

The requirements specify explicit, deterministic system responses for each failure mode:

### 11.1 Payment Failure
* **Scenario:** External payment gateway explicitly rejects the charge (e.g., insufficient funds, card declined).
* **Required Behavior:** 
  1. Payment Service marks transaction as `FAILED`.
  2. Inventory Service receives immediate compensation event to transition reservation from `PAYMENT_PENDING` $\rightarrow$ `RELEASED`.
  3. Available inventory is incremented back by the reserved amount.
  4. Checkout UI notifies customer with clear failure reason; cart remains intact for retry with alternative payment method.

### 11.2 Payment Timeout
* **Scenario:** Payment Service dispatches authorization request to payment gateway, but the connection drops or times out after $X$ seconds with no response.
* **Required Behavior:** 
  1. Transaction state transitions to `PAYMENT_UNKNOWN_TIMEOUT`.
  2. System initiates an idempotent background verification/status query to the gateway.
  3. If payment was settled, order is marked `CONFIRMED`.
  4. If payment was never processed, payment is formally cancelled and reservation is reclaimed.
  5. Customer is notified of pending confirmation rather than double-charged.

### 11.3 Duplicate Payment Request
* **Scenario:** Client browser or malicious script sends multiple identical payment authorization requests for the same checkout session.
* **Required Behavior:** 
  1. Payment Service verifies the unique `Idempotency-Key` or `CheckoutSessionId`.
  2. Active in-flight lock rejects concurrent identical requests with HTTP `409 Conflict`.
  3. Subsequent completed queries return the exact cached result of the first authorization without re-contacting the gateway.

### 11.4 Duplicate Buy / Reservation Request
* **Scenario:** User aggressively double-clicks "Buy Now", or automated bots fire concurrent reservation requests for the same user session.
* **Required Behavior:** 
  1. Ingress/Reservation boundary validates `(UserId, FlashSaleId, SKU)`.
  2. Only the first atomic acquisition succeeds; the duplicate invocation is recognized as a duplicate and either bound to the existing reservation or gracefully rejected.
  3. A single user cannot consume multiple units unless explicitly permitted by business policy.

### 11.5 Order Service 30-Second Outage Post-Payment
* **Scenario:** Payment gateway successfully authorizes and settles payment, but the internal Order Service crashes or undergoes a 30-second restart before the order record is written.
* **Required Behavior:** 
  1. Payment Service publishes an immutable `PaymentSettledEvent` to a durable, persistent distributed message log.
  2. The message remains unacknowledged in the broker while the Order Service is offline.
  3. Upon Order Service recovery (at $t = 30\text{s}$), the consumer resumes processing the queue, writes the `CONFIRMED` order, and acknowledges the message.
  4. Zero customer payments are lost or orphaned.

### 11.6 Database Partition / Primary Failure
* **Scenario:** Primary database node experiences hardware failure or network split during peak reservation operations.
* **Required Behavior:** 
  1. Automated health checks initiate failover to standby replica.
  2. During the split-brain / failover window, in-flight write operations fail fast with HTTP `503 Service Unavailable`.
  3. Under no condition does the system fall back to an uncoordinated secondary that permits duplicate reservations or dirty reads. Consistency takes absolute precedence over write availability.

### 11.7 Message Processing Failure (Consumer Crash)
* **Scenario:** Downstream consumer crashes mid-processing of an event (e.g., during fulfillment notification).
* **Required Behavior:** 
  1. Unacknowledged message is redelivered to an alternative consumer instance after visibility timeout.
  2. Consumer enforces consumer-side idempotency using the event's unique message ID.
  3. Repeatedly failing messages are routed to a Dead Letter Queue (DLQ) with alert triggers after max retry threshold.

### 11.8 External Dependency Failure (Complete Gateway Down)
* **Scenario:** External payment provider suffers an extended regional outage.
* **Required Behavior:** 
  1. Circuit breaker trips open after reaching configured failure threshold.
  2. System gracefully informs customers entering checkout that payment processing is temporarily degraded.
  3. System halts new reservations for that payment method rather than locking inventory in indefinitely pending states.

### 11.9 Reservation Expiry (TTL Elapsed)
* **Scenario:** Customer reserves stock, receives a reservation token, but closes the browser or abandons the payment window.
* **Required Behavior:** 
  1. Reservation time-to-live expires.
  2. Automated scheduler or reactive reconciliation identifies the expired reservation.
  3. State updates from `RESERVED` $\rightarrow$ `RELEASED`.
  4. Available inventory is atomically restored.
  5. Stale client attempting payment post-expiry is rejected with an `EXPIRED_RESERVATION` error.

### 11.10 Massive Traffic Spike (500k req/sec Surge)
* **Scenario:** 500,000 requests hit the platform in the first 5 seconds of the flash sale.
* **Required Behavior:** 
  1. Edge CDN and API Gateway absorb and rate-limit anomalous IPs and unauthorized traffic.
  2. Static catalog assets served from distributed cache.
  3. Fair queue or token-bucket rate limiter admits only manageable batches to the transactional reservation boundary.
  4. Overflow requests receive structured, friendly "In Waiting Room" or "Sale Sold Out" responses without crashing upstream core databases.

---

## 12. Out of Scope for This Document

To preserve the integrity of the design-first methodology, the following technical and implementation decisions are **strictly out of scope** for this document:

* **Specific Database Engine Selection:** No final determination of PostgreSQL, MySQL, CockroachDB, Cassandra, DynamoDB, or Spanner.
* **Specific Message Broker Selection:** No final determination of Apache Kafka, RabbitMQ, AWS SQS/SNS, or NATS.
* **Specific Caching & In-Memory Store Technology:** No final determination of Redis, Memcached, Dragonfly, or Hazelcast.
* **Specific Concurrency Implementation Mechanics:** No final commitment to Redis Lua scripts, distributed Redlock, optimistic locking (`SELECT FOR UPDATE`), or saga orchestrators.
* **Specific Cloud Vendor & Orchestration Tools:** No binding to AWS, GCP, Azure, Kubernetes, or Serverless.
* **Detailed Code & Implementation Artifacts:** No concrete programming language syntax, ORM entity definitions, or service class implementations.
* **Concrete REST/gRPC/GraphQL Schemas:** Exact JSON/Protobuf schemas will be defined in `05_API/`.
* **Database DDL & Relational Schemas:** Specific table schemas, indexing strategies, and partitioning rules will be defined in `04_Database/`.

*All above concerns are reserved for subsequent HLD, LLD, Database, API, and Architecture Decision Records (ADR).*

---

## 13. Requirement Traceability Matrix

The following matrix establishes end-to-end traceability for 28 core requirements across priority, guarantee classification, and downstream design artifacts:

| Requirement ID | Domain | Requirement Description | Priority | Guarantee Type | Downstream Design Artifact |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FR-DISC-001** | Discovery | Support browse and catalog retrieval | High | Target | `02_HLD`, `05_API` |
| **FR-DISC-003** | Discovery | Expose stock availability indicators | High | Target | `02_HLD`, `05_API` |
| **FR-CART-001** | Cart | Maintain persistent user cart state | Medium | Target | `03_LLD`, `04_Database` |
| **FR-INV-001** | Inventory | Real-time atomic inventory balance checks | Critical | **Strict Guarantee** | `02_HLD`, `03_LLD`, `04_Database` |
| **FR-INV-002** | Inventory | Atomic inventory reservation decrements | Critical | **Strict Guarantee** | `02_HLD`, `03_LLD`, `10_ADR` |
| **FR-INV-003** | Inventory | Confirm reservation upon verified payment | Critical | **Strict Guarantee** | `03_LLD`, `04_Database` |
| **FR-INV-004** | Inventory | Release reservation on payment rejection | Critical | **Strict Guarantee** | `03_LLD`, `08_Scalability_Reliability` |
| **FR-INV-005** | Inventory | Automatic reclamation of expired reservations | Critical | **Strict Guarantee** | `03_LLD`, `08_Scalability_Reliability` |
| **FR-INV-006** | Inventory | Guarantee inventory balance never negative | Critical | **Strict Guarantee** | `02_HLD`, `04_Database`, `10_ADR` |
| **FR-RES-001** | Reservation | Deterministic reservation state machine | Critical | **Strict Guarantee** | `03_LLD`, `04_Database` |
| **FR-RES-002** | Reservation | Bound reservation duration via explicit TTL | Critical | Target | `03_LLD`, `10_ADR` |
| **FR-CHK-001** | Checkout | Immutable Checkout Session creation | High | Target | `03_LLD`, `05_API` |
| **FR-CHK-002** | Checkout | Enforce active reservation prior to payment | Critical | **Strict Guarantee** | `03_LLD`, `05_API` |
| **FR-PAY-001** | Payment | Idempotent payment authorization handling | Critical | **Strict Guarantee** | `03_LLD`, `05_API`, `10_ADR` |
| **FR-PAY-002** | Payment | Automated payment reconciliation workflow | High | **Strict Guarantee** | `08_Scalability_Reliability` |
| **FR-ORD-001** | Order | Strict order state machine transitions | Critical | **Strict Guarantee** | `03_LLD`, `04_Database` |
| **FR-ORD-002** | Order | Zero payment-order drop during service downtime| Critical | **Strict Guarantee** | `02_HLD`, `08_Scalability_Reliability` |
| **FR-FUL-001** | Fulfillment | Event-driven fulfillment initiation | Medium | Target | `02_HLD`, `03_LLD` |
| **FR-NOTIF-001**| Notification | Asynchronous non-blocking customer alerts | Low | Target | `02_HLD`, `08_Scalability_Reliability` |
| **FR-IDEM-001** | Idempotency | Universal Idempotency-Key support on writes | Critical | **Strict Guarantee** | `03_LLD`, `05_API`, `10_ADR` |
| **NFR-SCALE-001**| Scalability | Sustain 10k req/sec base traffic | High | Target | `02_HLD`, `08_Scalability_Reliability` |
| **NFR-SCALE-002**| Scalability | Absorb up to 500k req/sec peak flash surge | High | Target | `02_HLD`, `08_Scalability_Reliability` |
| **NFR-CONC-001** | Concurrency | Arbitrate 10k concurrent requests on 100 units| Critical | **Strict Guarantee** | `02_HLD`, `03_LLD`, `10_ADR` |
| **NFR-CONS-001** | Consistency | Strong consistency at reservation boundary | Critical | **Strict Guarantee** | `04_Database`, `10_ADR` |
| **NFR-REL-001** | Reliability | Automated recovery from payment timeouts | Critical | **Strict Guarantee** | `08_Scalability_Reliability` |
| **NFR-REL-002** | Reliability | Survive 30s downstream Order Service outage | High | **Strict Guarantee** | `02_HLD`, `08_Scalability_Reliability` |
| **NFR-SEC-001** | Security | Tokenized payment handling (PCI-DSS) | Critical | **Strict Guarantee** | `09_Security_Observability` |
| **NFR-OBS-001** | Observability | End-to-end distributed tracing across boundary| High | Target | `09_Security_Observability` |

---

## 14. Acceptance Criteria

The system architecture and its subsequent designs shall be deemed acceptable if and only if they demonstrate clear, verifiable mechanisms satisfying the following twelve criteria:

1. **Massive Concurrency Arbitration:** The architecture demonstrates how 10,000 concurrent purchase requests enter the system without causing connection pool depletion, server crashes, or unbounded memory usage.
2. **Absolute Overselling Prevention:** In the benchmark scenario (10,000 requests for 100 units), the architecture guarantees that no more than 100 reservations are granted.
3. **Mathematical Invariant Preservation:** The system demonstrates how inventory balances are physically prevented from dipping below zero ($Stock \ge 0$).
4. **Idempotency Across Replays:** Repeated submissions of identical reservation, payment, or order creation requests return identical responses with zero duplicate state mutations.
5. **Prompt Compensation on Failure:** When a payment fails or is rejected, the architecture guarantees the associated reservation is reclaimed and restored to stock.
6. **Robust Timeout Reconciliation:** A gateway timeout or unacknowledged payment response is automatically resolved via an auditable reconciliation path.
7. **Downstream Outage Survival:** A simulated 30-second complete outage of the Order Service during payment completion results in zero lost orders and complete recovery upon service restart.
8. **Exhaustion Handling:** When stock reaches zero, all subsequent reservation attempts are immediately rejected with clean, low-latency "Sold Out" signals.
9. **Automated Expiry Reclamation:** Abandoned reservations with expired TTLs have their inventory restored reliably to the available pool.
10. **Stateless Horizontal Elasticity:** The design demonstrates how application service instances can scale out linearly behind load balancers.
11. **Comprehensive Observability:** Every transaction is traceable via a unique correlation ID from ingress edge to database commit and downstream messaging.
12. **Rigorous Security Boundaries:** Architecture defines authentication, authorization, rate limiting, and tokenized payment boundaries adhering to zero-trust principles.

---

## 15. Architectural Design Principles

The design of SALESTORM in all downstream documents (`02_HLD` through `10_ADR`) must adhere to these ten architectural principles:

1. **Correctness Over Blind Availability at the Reservation Boundary:** Under extreme write contention, preserving inventory integrity ($Stock \ge 0$) is paramount. Dropping or shedding excessive traffic is preferable to overselling stock.
2. **Universal Idempotency for Mutating Operations:** Every state-altering API call must be designed under the assumption that network drops will trigger client and system retries.
3. **Single Ownership of Bounded Business State:** Each aggregate root (Inventory, Order, Payment) is owned by exactly one bounded context and service. Direct cross-database writes are prohibited.
4. **Synchronous Processing Only for Immediate Correctness:** Synchronous blocking request-response cycles are restricted solely to operations requiring atomic validation (e.g., inventory reservation decrement).
5. **Asynchronous Processing for Resilience & Decoupling:** All post-reservation and post-payment workflows (order dispatch, fulfillment, notifications, analytics) must be decoupled via durable message brokers.
6. **Statelessness in the Compute Tier:** No application node shall maintain session-affinity or in-memory transaction states that prevent arbitrary instance termination or horizontal auto-scaling.
7. **Explicit Failure & Compensation Modeling:** Every happy-path transaction must have an explicitly documented compensation, timeout, and recovery flow.
8. **Full-Spectrum Observability by Design:** Logs, metrics, and distributed trace headers must be first-class architectural components, not operational afterthoughts.
9. **Defensible Trade-Offs for Major Decisions:** Every architectural decision (e.g., lock-free counters vs. ACID transactions, cache invalidation vs. TTLs) must be justified via an Architecture Decision Record (ADR) analyzing pros, cons, and alternatives.
10. **Clarity & Implementability:** The architecture must be lucid, precise, and easily understood by the entire engineering organization, providing clear guidelines for implementation.

---
*End of Document — Proceed to `02_HLD/` for High-Level Architectural Design.*
