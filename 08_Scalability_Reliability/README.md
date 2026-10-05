# Scalability & Reliability Design

## Scalability

<img width="2642" height="589" alt="Scalability" src="https://github.com/user-attachments/assets/1c888fcb-0431-4fbc-b987-0461d779c34a" />


* Horizontal scaling of stateless services
* Load Balancer distributes traffic
* Flash Sale Guard controls traffic spikes
* Redis Cache reduces database load
* Message Queue handles asynchronous processing
* Database remains the source of truth
* Supports high concurrent flash-sale traffic

## Reliability

<img width="3098" height="352" alt="Reliability" src="https://github.com/user-attachments/assets/4ae02909-837b-4813-8ed1-11493ca931a3" />

* Timeout and retry for transient failures
* Circuit breaker prevents cascading failures
* Idempotency prevents duplicate payments/orders
* Message Queue provides reliable async processing
* Dead Letter Queue handles failed messages
* Reservation expiry releases abandoned stock
* Reconciliation recovers inconsistent states
* Monitoring, logging and tracing for fault detection
