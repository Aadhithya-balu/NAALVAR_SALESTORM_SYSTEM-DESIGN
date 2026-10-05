# SALESTORM — Requirements & Assumptions

### High-Scale E-Commerce Flash Sale

**SYSCRAFTERS 2026 — Design-First, AI-Assisted System Design Hackathon**

---

## Document Information

| Attribute            | Details                                                                                                             |
| :------------------- | :------------------------------------------------------------------------------------------------------------------ |
| **Project**          | SALESTORM                                                                                                           |
| **Hackathon**        | SYSCRAFTERS 2026                                                                                                    |
| **Document Version** | 1.0.0                                                                                                               |
| **Status**           | Approved Architectural Baseline                                                                                     |
| **Purpose**          | Define the business requirements, system expectations, assumptions, constraints, and success criteria for SALESTORM |
| **Next Documents**   | HLD, LLD, Database Design, API Design, Scalability & Reliability, Security & Observability, ADRs                    |

---

# 1. Business Problem

## 1.1 What is SALESTORM?

SALESTORM is a high-scale e-commerce platform designed to handle flash-sale events where a very large number of customers compete for a very limited number of products.

The main challenge is simple to describe but difficult to solve:

> **How do we allow thousands of customers to purchase a limited-stock product at the same time without overselling, double-charging customers, or losing orders when parts of the system fail?**

For example, imagine a flash sale where:

* 10,000 customers try to purchase the same product at almost the same time.
* Only 100 units are available.
* All 10,000 requests may reach the system within seconds.
* Some customers will retry because of slow networks or duplicate clicks.
* The payment gateway may timeout or become temporarily unavailable.
* Internal services may fail after payment has already succeeded.

SALESTORM is designed around handling exactly these situations.

---

## 1.2 Critical Flash-Sale Scenario

The main benchmark used throughout the design is:

* **10,000 concurrent buyers**
* **100 units of inventory**
* **Zero overselling allowed**
* **Exactly one successful reservation per available unit**
* **No duplicate payments**
* **No duplicate orders**
* **Expired reservations must eventually return to inventory**
* **Successful payments must not be lost even if downstream services are temporarily unavailable**

The key inventory rule is:

**Confirmed orders + active reservations must never exceed available stock.**

For the benchmark scenario:

**Maximum successful allocations = 100**

The remaining requests must be rejected cleanly rather than allowing the system to oversell.

---

# 2. Customer Purchase Journey

The overall customer journey looks like this:

```text
Customer
   │
   ▼
Product Discovery
   │
   ▼
Cart
   │
   ▼
Inventory Check
   │
   ▼
Inventory Reservation
   │
   ▼
Checkout
   │
   ▼
Payment
   │
   ├─────────────── Payment Success ───────────────┐
   │                                               │
   ▼                                               ▼
Payment Failed / Timeout                     Order Creation
   │                                               │
   ▼                                               ▼
Release Reservation                         Fulfillment
   │                                               │
   ▼                                               ▼
Stock Available Again                    Shipment & Tracking
                                                   │
                                                   ▼
                                             Notifications
```

The important distinction is that **adding a product to a cart does not reserve stock**.

Inventory is reserved only when the customer actually enters the purchase flow.

---

# 3. System Objectives

SALESTORM should achieve the following goals:

### 1. Handle sudden traffic spikes

The platform should absorb large flash-sale traffic surges without allowing the spike to take down the core transaction services.

### 2. Protect inventory

Inventory is the most important consistency boundary. The system must never allocate more physical units than actually exist.

### 3. Use temporary reservations

Once a customer starts checkout, their inventory can be held temporarily so another customer cannot take it while payment is being completed.

### 4. Release abandoned inventory

If a customer abandons checkout, payment fails, or a reservation expires, the reserved stock must eventually become available again.

### 5. Make operations idempotent

Network retries and duplicate clicks are expected. Repeating the same request must not create another reservation, payment, or order.

### 6. Keep order and payment states consistent

Payment, reservation, and order states must follow clearly defined state machines.

### 7. Recover automatically from failures

Temporary failures should result in retries, reconciliation, or compensation rather than permanently stuck transactions.

### 8. Scale horizontally

Stateless services should be able to scale by adding more instances instead of relying on a single application server.

### 9. Protect the system from abuse

Rate limiting, authentication, authorization, bot protection, and secure service-to-service communication should protect the platform during normal and extreme traffic.

### 10. Make the architecture easy to implement

The final design should be detailed enough that an engineering team can use it as a practical implementation blueprint.

---

# 4. Functional Requirements

## 4.1 Product Discovery

Users should be able to browse and search the product catalog without going through the transactional inventory system.

The system should provide:

* Product name and description
* Images
* Price
* Flash-sale schedule
* Basic availability information

Availability shown during browsing does not have to be perfectly real-time. A small amount of delay is acceptable because the actual inventory check happens during reservation.

Possible availability states:

```text
IN_STOCK
LOW_STOCK
SOLD_OUT
```

---

## 4.2 Cart Management

Authenticated customers should be able to:

* Add products to their cart
* Change quantities
* Remove products
* Continue using the cart across sessions

A cart represents **customer intent**, not ownership of inventory.

Therefore:

> **Adding an item to a cart must not reserve inventory.**

---

# 5. Inventory & Reservation

Inventory is the most critical part of SALESTORM.

The system must perform the actual inventory decision at a strongly consistent boundary.

## 5.1 Reservation Rules

A reservation should only succeed when sufficient stock exists.

The reservation operation must be atomic:

```text
Check stock
     ↓
Reserve stock
     ↓
Create reservation
```

These operations must behave as one logical transaction.

Two customers must never be able to observe the same final unit as available and both successfully claim it.

---

## 5.2 Reservation Lifecycle

The reservation follows a controlled state machine:

```text
AVAILABLE
    │
    │ Reserve
    ▼
RESERVED
    │
    │ Start Payment
    ▼
PAYMENT_PENDING
    │
    ├──────── Payment Success ────────► CONFIRMED
    │                                      │
    │                                      │ Fulfillment
    │                                      ▼
    │                                    SOLD
    │
    └──── Payment Failed / Timeout ──► RELEASED
                                           │
                                           ▼
                                      AVAILABLE
```

### Successful path

```text
AVAILABLE
→ RESERVED
→ PAYMENT_PENDING
→ CONFIRMED
→ SOLD
```

### Payment failure

```text
RESERVED / PAYMENT_PENDING
→ PAYMENT_FAILED
→ RELEASED
→ AVAILABLE
```

### Reservation timeout

```text
RESERVED / PAYMENT_PENDING
→ TIMEOUT
→ RELEASED
→ AVAILABLE
```

---

## 5.3 Invalid Transitions

The system must reject invalid state changes.

For example:

```text
RELEASED → CONFIRMED
AVAILABLE → CONFIRMED
SOLD → AVAILABLE
```

A sold item cannot simply become available again. A return or refund process would be required for that scenario.

---

# 6. Checkout

When the customer starts checkout, SALESTORM creates a checkout session containing:

* Customer ID
* Selected products
* Quantity
* Price snapshot
* Shipping information
* Active reservation reference

The checkout session should be treated as an immutable record of what the customer is attempting to purchase.

Before payment begins, the system must verify that the reservation:

* Exists
* Belongs to the customer
* Has not expired
* Is still in a valid state

Only then should the payment request be sent.

---

# 7. Payment Processing

Payment is an external dependency, so SALESTORM must assume that it can be slow, unavailable, or uncertain.

## Payment success

When payment is confirmed:

1. The payment record becomes `SETTLED`.
2. The reservation is confirmed.
3. An order confirmation event is created.
4. The order continues through the asynchronous fulfillment pipeline.

## Payment failure

When the gateway clearly rejects the payment:

1. Payment becomes `FAILED`.
2. The reservation is released.
3. The inventory becomes available again.
4. The customer can retry with another payment method.

## Payment timeout

A timeout does **not** automatically mean payment failed.

Instead:

```text
PAYMENT_PENDING
       │
       │ Gateway timeout
       ▼
UNKNOWN_PENDING
       │
       ├── Payment found → CONFIRMED
       │
       └── Payment not found → FAILED → RELEASE
```

This prevents SALESTORM from accidentally charging a customer twice.

---

# 8. Idempotency

Idempotency is one of the core design principles of SALESTORM.

Customers can double-click buttons. Mobile networks can retry requests. Load balancers can retry failed connections. Services can restart while processing requests.

The system therefore uses an `Idempotency-Key` for state-changing operations.

Examples include:

* Reservation
* Checkout
* Payment
* Order creation

If the same operation is received again, the system should return the result of the original operation instead of performing it again.

For example:

```text
Request 1
Idempotency-Key: ABC123
       ↓
Reservation created

Request 2
Idempotency-Key: ABC123
       ↓
Return original reservation
       ↓
No second reservation
```

Idempotency keys should be scoped appropriately to the customer and operation and should have a lifecycle matching the transaction.

---

# 9. Order Lifecycle

Orders follow a controlled state machine:

```text
CREATED
   │
   ▼
PAYMENT_PENDING
   │
   ▼
CONFIRMED
   │
   ▼
PROCESSING
   │
   ▼
SHIPPED
   │
   ▼
OUT_FOR_DELIVERY
   │
   ▼
DELIVERED
```

The system must reject invalid transitions.

For example:

```text
CREATED → SHIPPED
CANCELLED → DELIVERED
PAYMENT_PENDING → SHIPPED
```

Every transition should be auditable so that the complete history of an order can be reconstructed.

---

# 10. Fulfillment & Notifications

Once an order is confirmed, fulfillment should happen asynchronously.

The checkout request should not wait for:

* Warehouse processing
* Shipping label creation
* Logistics providers
* Email delivery
* SMS delivery
* Push notifications

A confirmed order can instead produce an event such as:

```text
OrderConfirmedEvent
        │
        ├──► Fulfillment Service
        │
        ├──► Notification Service
        │
        └──► Analytics
```

If one of these services is temporarily unavailable, the purchase itself should remain unaffected.

---

# 11. Non-Functional Requirements

| Category                   | Target                                          |
| :------------------------- | :---------------------------------------------- |
| Base traffic               | ~10,000 requests/sec                            |
| Flash-sale edge traffic    | Up to 500,000 requests/sec                      |
| Concurrent buyers          | 10,000                                          |
| Available stock            | 100 units                                       |
| Maximum reservations       | 100                                             |
| Inventory invariant        | Stock must never become negative                |
| Reservation P99            | <150 ms design target                           |
| Catalog P95                | <30 ms design target                            |
| Downstream consistency     | <2 sec where eventual consistency is acceptable |
| Critical-path availability | 99.99% target                                   |
| Client communication       | TLS 1.3                                         |
| Service communication      | mTLS                                            |
| Authentication             | OAuth2/OIDC + JWT                               |
| Authorization              | RBAC                                            |
| Observability              | Metrics + logs + distributed tracing            |

The performance numbers are **engineering targets**, not absolute guarantees.

The most important guarantee remains inventory correctness.

---

# 12. Strict Guarantees vs. Targets

It is important to distinguish between what SALESTORM **must guarantee** and what it is simply **designed to achieve**.

## Strict guarantees

### Inventory

* No overselling
* Inventory cannot become negative
* Active reservations + confirmed orders cannot exceed available stock

### Payments

* One checkout cannot result in multiple charges
* Retries must use the same payment idempotency key
* Payment timeouts must be reconciled

### Orders

* One checkout produces at most one order
* Invalid state transitions are rejected
* Confirmed payments cannot disappear

### Reservations

* Expired reservations must eventually be released
* Failed payments must release their reservations

### Fault recovery

* A temporary Order Service outage must not lose successful payments
* Consumers must safely handle duplicate events

---

## Engineering targets

These are important goals but cannot be treated as absolute guarantees:

* 500k requests/sec at the edge
* <150 ms reservation P99
* <30 ms catalog P95
* 99.99% availability
* Recovery of normal operation within approximately 60 seconds after service restoration

---

# 13. Traffic Model

SALESTORM is designed around three major traffic levels.

```text
~10k req/sec
Base traffic
     │
     │ Flash sale begins
     ▼
~500k req/sec
Edge traffic
     │
     │ Filtering + caching + rate limiting
     ▼
10k concurrent requests
Hot SKU
     │
     ▼
100 successful reservations
     │
     ▼
9,900 rejected requests
```

The key point is that **the entire 500k req/sec load must not reach the inventory database**.

The edge, cache, rate limiter, waiting room, and queueing layers should absorb and filter traffic before requests reach the critical reservation boundary.

---

# 14. The Hot-Key Problem

The hardest technical problem in SALESTORM is not simply handling large network traffic.

It is handling thousands of writes against the **same inventory item**.

For example:

```text
10,000 buyers
      │
      ▼
   Same SKU
      │
      ▼
    100 units
```

Traditional database locking can cause:

* High lock contention
* Connection pool exhaustion
* Increased latency
* Thread starvation
* Cascading failures

Therefore, the architecture must protect the inventory boundary from uncontrolled concurrency while still preserving atomicity.

---

# 15. Inventory Invariants

Let:

* `I₀` = initial inventory
* `R(t)` = successful reservations
* `E(t)` = released or expired reservations
* `A(t)` = currently available inventory

Then:

```text
A(t) = I₀ - [R(t) - E(t)]
```

And the system must always maintain:

```text
A(t) >= 0
```

For the benchmark:

```text
I₀ = 100
```

Therefore:

```text
Active reservations <= 100
Confirmed orders <= 100
Oversold items = 0
```

These are core correctness properties of the system.

---

# 16. Architecture Assumptions

The design is based on the following assumptions:

### 1. Inventory has one authoritative source

A single transactional boundary owns the real-time inventory state.

### 2. Inventory consistency is isolated

The Inventory/Reservation service is responsible for concurrency control rather than allowing every service to modify stock directly.

### 3. Reservations have an explicit TTL

Every temporary reservation has an expiry time.

### 4. Duplicate requests are normal

Users and networks will retry requests. The architecture must expect this rather than treat it as an exceptional case.

### 5. Payment gateways can fail

Payment providers may timeout, return errors, become rate-limited, or temporarily go offline.

### 6. Internal services can fail

Order, fulfillment, notification, and other services may experience temporary outages.

### 7. Messaging can be used for durability

Durable asynchronous messaging is available for workflows that do not require an immediate response.

### 8. Application services are stateless

Session and transaction state is stored outside application instances.

### 9. The transactional store supports strong concurrency guarantees

The chosen persistence technology must support atomic operations and appropriate transactional isolation.

### 10. Idempotency is required throughout the transaction flow

Every state-changing boundary must be designed to safely handle retries.

---

# 17. System Constraints

SALESTORM operates under several hard constraints.

### Single hot SKU

The benchmark intentionally creates extreme contention on one product. Adding more database shards does not automatically solve contention when all requests target the same SKU.

### Fixed physical inventory

The benchmark has exactly 100 physical units.

There is:

* No backordering
* No pre-ordering
* No hidden inventory buffer

### Zero overselling tolerance

Selling more units than physically available is considered a system failure.

### External payment dependency

SALESTORM cannot control the payment gateway's internal processing time or availability.

### Design-first scope

The hackathon focuses primarily on architecture, correctness, trade-offs, and failure handling rather than generating a complete production application.

Any prototype code should be used to validate architectural assumptions, such as:

* Lock contention
* Reservation throughput
* Idempotency behavior
* Failure recovery

---

# 18. Service Dependencies

| Service              | Responsibility                | Checkout Critical? | Failure Handling                             |
| :------------------- | :---------------------------- | :----------------- | :------------------------------------------- |
| Product Service      | Catalog and pricing           | No                 | Serve cached data                            |
| Cart Service         | Customer cart                 | No                 | Degrade gracefully                           |
| Sale Service         | Sale schedule and eligibility | No                 | Cache rules; fail closed when required       |
| Inventory Service    | Reservations and stock        | **Yes**            | Fail closed to protect inventory             |
| Checkout Service     | Checkout orchestration        | **Yes**            | Reject safely if validation fails            |
| Payment Service      | Payment processing            | **Yes**            | Timeout, retry, reconcile                    |
| Order Service        | Order lifecycle               | No                 | Consume durable events after recovery        |
| Fulfillment Service  | Warehouse/shipping flow       | No                 | Queue and retry                              |
| Notification Service | Email/SMS/Push                | No                 | Queue and retry                              |
| Database             | Transactional source of truth | **Yes**            | Failover; reject writes during unsafe states |
| Cache                | Read acceleration             | Conditional        | Shed load if unavailable                     |
| Message Broker       | Durable events                | Conditional        | Buffer/retry or reject safely                |

---

# 19. Failure Scenarios

SALESTORM should be designed around failure rather than treating failure as an edge case.

## Payment failure

```text
Payment rejected
      ↓
Payment = FAILED
      ↓
Release reservation
      ↓
Return stock
      ↓
Customer can retry
```

---

## Payment timeout

```text
Gateway timeout
      ↓
Payment = UNKNOWN
      ↓
Reconcile with gateway
      │
      ├── Settled → Confirm order
      │
      └── Not settled → Release reservation
```

The system must never blindly retry a payment as a new transaction.

---

## Duplicate payment

Multiple requests for the same checkout should resolve to the same payment transaction.

```text
Request A ──┐
            ├──► Same Idempotency Key
Request B ──┘
                  │
                  ▼
             One payment
```

---

## Duplicate reservation

If the same customer repeatedly clicks Buy:

```text
Request 1 → Reservation created

Request 2 → Existing reservation returned

Request 3 → Existing reservation returned/rejected
```

The exact behavior can depend on the business policy, but it must never consume additional inventory unintentionally.

---

## Order Service outage

If payment succeeds while Order Service is unavailable:

```text
Payment Gateway
      ↓
Payment Service
      ↓
PaymentSettledEvent
      ↓
Durable Message Broker
      ↓
Order Service unavailable
      ↓
30 seconds later
      ↓
Order Service recovers
      ↓
Event consumed
      ↓
Order created
```

The event remains durable until it has been successfully processed.

---

## Database failure

If the primary database becomes unavailable:

```text
Database failure
      ↓
Failover / health check
      ↓
Unsafe writes rejected
      ↓
No uncontrolled secondary writes
      ↓
Recovery
      ↓
Normal processing resumes
```

During the failover window, rejecting a transaction is preferable to risking incorrect inventory.

---

## Consumer failure

If a service crashes while processing a message:

```text
Message
   ↓
Consumer
   ↓
Consumer crashes
   ↓
Message becomes available again
   ↓
Another consumer retries
```

Consumer-side idempotency ensures that processing the message again does not create duplicate side effects.

Repeated failures should eventually move the message to a Dead Letter Queue.

---

## Payment Gateway outage

If the payment provider is completely unavailable:

1. Circuit breaker opens.
2. New payment attempts are restricted.
3. Customers receive a clear temporary-unavailability message.
4. The system avoids holding inventory indefinitely.
5. Existing pending payments continue through reconciliation.

---

## Reservation expiry

When the reservation TTL expires:

```text
Reservation expires
       ↓
Expiry worker/reconciliation
       ↓
RESERVED → RELEASED
       ↓
Inventory restored
       ↓
AVAILABLE
```

A customer attempting to pay using an expired reservation must receive an `EXPIRED_RESERVATION` response.

---

## 500k req/sec flash spike

The edge layer should handle the majority of the surge before it reaches the core transactional system.

```text
500k req/sec
     ↓
CDN / WAF
     ↓
Rate Limiting
     ↓
Bot Protection
     ↓
Waiting Room / Queue
     ↓
Manageable traffic
     ↓
Reservation Service
```

Customers who cannot be admitted immediately should receive a controlled waiting-room or sold-out response rather than causing the core system to collapse.

---

# 20. Out of Scope

The following decisions are intentionally left for the later architecture documents:

* Exact database technology
* Exact message broker
* Exact caching technology
* Specific locking/concurrency mechanism
* Cloud provider
* Kubernetes or serverless platform
* Programming language
* ORM implementation
* Database DDL
* Exact REST/gRPC/GraphQL schemas
* Production application code

These decisions will be covered in the HLD, LLD, Database, API, Scalability, Security, and ADR documents.

---

# 21. Requirement Traceability

The major requirements will be traced through the downstream design documents.

| Requirement                   | Priority | Guarantee  | Main Design Area     |
| :---------------------------- | :------- | :--------- | :------------------- |
| Catalog access                | High     | Target     | HLD / API            |
| Persistent cart               | Medium   | Target     | LLD / Database       |
| Atomic inventory check        | Critical | **Strict** | HLD / LLD / Database |
| Atomic reservation            | Critical | **Strict** | HLD / LLD / ADR      |
| Reservation confirmation      | Critical | **Strict** | LLD / Database       |
| Reservation release           | Critical | **Strict** | LLD / Reliability    |
| Expiry reclamation            | Critical | **Strict** | LLD / Reliability    |
| Non-negative inventory        | Critical | **Strict** | HLD / Database / ADR |
| Reservation state machine     | Critical | **Strict** | LLD                  |
| Checkout validation           | Critical | **Strict** | LLD / API            |
| Payment idempotency           | Critical | **Strict** | LLD / API / ADR      |
| Payment reconciliation        | High     | **Strict** | Reliability          |
| Order state machine           | Critical | **Strict** | LLD / Database       |
| Payment-to-order durability   | Critical | **Strict** | HLD / Reliability    |
| Event-driven fulfillment      | Medium   | Target     | HLD / LLD            |
| Async notifications           | Low      | Target     | HLD / Reliability    |
| Universal idempotency         | Critical | **Strict** | LLD / API / ADR      |
| 10k req/sec base traffic      | High     | Target     | HLD / Scalability    |
| 500k req/sec edge surge       | High     | Target     | HLD / Scalability    |
| 10k concurrent buyers         | Critical | **Strict** | HLD / LLD / ADR      |
| Strong inventory consistency  | Critical | **Strict** | Database / ADR       |
| Payment failure recovery      | Critical | **Strict** | Reliability          |
| Order-service outage recovery | High     | **Strict** | HLD / Reliability    |
| Secure payment boundary       | Critical | **Strict** | Security             |
| Distributed tracing           | High     | Target     | Observability        |

---

# 22. Acceptance Criteria

The architecture will be considered successful if it can clearly demonstrate the following:

### 1. Handle 10,000 concurrent buyers

The system should absorb the requests without exhausting database connections, application memory, or server resources.

### 2. Never oversell

With 100 units and 10,000 buyers, no more than 100 reservations can succeed.

### 3. Preserve the inventory invariant

At every point:

```text
Stock >= 0
```

### 4. Handle retries safely

Repeated reservation, payment, and order requests must not create duplicate side effects.

### 5. Release failed reservations

When payment fails, the reserved inventory must eventually become available again.

### 6. Recover payment timeouts

Unknown payment states must be reconciled rather than blindly retried.

### 7. Survive downstream outages

A 30-second Order Service outage after successful payment must not result in a lost order.

### 8. Handle sold-out conditions

Once the 100 available units are allocated, subsequent requests should receive a clear and low-latency sold-out response.

### 9. Reclaim expired reservations

Abandoned reservations must eventually return their inventory.

### 10. Scale horizontally

Stateless services should be able to scale by adding more instances.

### 11. Provide end-to-end visibility

Each transaction should be traceable from the initial request through reservation, payment, order creation, and downstream events.

### 12. Maintain strong security boundaries

Authentication, authorization, rate limiting, service identity, and payment-tokenization boundaries must be clearly defined.

---

# 23. Architectural Principles

All subsequent SALESTORM architecture documents should follow these principles.

### 1. Correctness comes first

At the inventory boundary, rejecting excess traffic is always preferable to overselling.

### 2. Assume retries

Every state-changing operation should be designed with retries and duplicate requests in mind.

### 3. Give each service clear ownership

Inventory, Payment, and Order state should each have a clearly defined owner. Services should not directly modify another service's database.

### 4. Keep synchronous operations limited

Use synchronous communication where an immediate correctness decision is required, especially during inventory reservation.

### 5. Use asynchronous processing where possible

Fulfillment, notifications, analytics, and other downstream workflows should be event-driven.

### 6. Keep application services stateless

No application instance should depend on local memory for critical transaction state.

### 7. Design for failure

Every important workflow should define what happens when it succeeds, fails, times out, or is interrupted halfway through.

### 8. Build observability into the architecture

Logs, metrics, traces, audit events, and correlation IDs should be part of the design from the beginning.

### 9. Document important trade-offs

Major architectural choices should be captured through ADRs, including the alternatives considered and why a particular approach was selected.

### 10. Keep the design implementable

The final architecture should not only look good on a diagram. Every major component should have a clear responsibility and a practical path to implementation.

---

# Final Design Principle

The most important idea behind SALESTORM is simple:

> **When 10,000 customers compete for 100 products, the system does not need to make everyone successful. It needs to make the outcome correct.**

The architecture should therefore prioritize:

**Correct inventory → Safe payments → Durable orders → Reliable recovery → Scalability**

Everything else should support these goals.

---

**Next:** Proceed to `02_HLD/` for the High-Level Architecture.
