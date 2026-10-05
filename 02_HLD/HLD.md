# SALESTORM — High-Level Design

## 1. Architecture Overview

SALESTORM is designed as a highly scalable, fault-tolerant flash-sale platform capable of handling:

- 10,000+ concurrent purchase requests
- 100 available units
- Zero overselling
- Duplicate request protection
- Reliable payment processing
- Event-driven order processing
- Horizontal scaling
- Multi-AZ availability

The architecture is divided into five major areas:

1. Client & Edge Layer
2. Core Commerce Services
3. Event-Driven Processing
4. Data & Infrastructure
5. External Integrations

---

## 2. High-Level Architecture

```mermaid
flowchart TB

%% =========================================================
%% CLIENT + EDGE
%% =========================================================

subgraph EDGE["CLIENT & EDGE LAYER"]

    direction LR

    CLIENT["Customers<br/>Web / Mobile"]

    CDN["CDN"]

    WAF["WAF<br/>DDoS Protection"]

    TRAFFIC["Traffic Control<br/>Rate Limiting"]

    QUEUE["Flash Sale<br/>Waiting Room"]

    LB["Load Balancer"]

    API["API Gateway"]

    CLIENT --> CDN
    CDN --> WAF
    WAF --> TRAFFIC

    TRAFFIC --> QUEUE
    TRAFFIC --> LB

    QUEUE --> LB
    LB --> API

end


%% =========================================================
%% CORE COMMERCE
%% =========================================================

subgraph CORE["CORE COMMERCE SERVICES"]

    direction LR

    CATALOG["Product<br/>Catalog"]

    SEARCH["Search"]

    CART["Cart"]

    SALE["Sale /<br/>Promotion"]

    INVENTORY["INVENTORY &<br/>RESERVATION<br/><br/>CRITICAL CONSISTENCY<br/>BOUNDARY"]

    CHECKOUT["Checkout"]

    PAYMENT["Payment"]

    ORDER["Order"]

end


%% =========================================================
%% ASYNC
%% =========================================================

subgraph ASYNC["EVENT-DRIVEN PROCESSING"]

    direction LR

    OUTBOX["Transactional<br/>Outbox"]

    BUS["Durable Event Bus"]

    FULFILMENT["Fulfilment"]

    SHIPMENT["Shipment"]

    NOTIFICATION["Notification"]

end


%% =========================================================
%% DATA
%% =========================================================

subgraph DATA["DATA & CACHING"]

    direction LR

    CACHE[("Distributed<br/>Cache")]

    PRODUCTDB[("Product DB")]

    INVENTORYDB[("Inventory<br/>Ledger")]

    RESERVATIONDB[("Reservation<br/>Store")]

    PAYMENTDB[("Payment<br/>Ledger")]

    ORDERDB[("Order DB")]

end


%% =========================================================
%% EXTERNAL
%% =========================================================

subgraph EXTERNAL["EXTERNAL SYSTEMS"]

    direction LR

    PAYGATEWAY["Payment<br/>Gateway"]

    CARRIER["Shipping<br/>Partner"]

    MSG["Notification<br/>Provider"]

end


%% =========================================================
%% MAIN REQUEST FLOW
%% =========================================================

API --> CATALOG
API --> SEARCH
API --> CART
API --> SALE

CATALOG --> CACHE
SEARCH --> CACHE
CART --> CACHE
SALE --> CACHE

CATALOG --> PRODUCTDB


%% =========================================================
%% PURCHASE FLOW
%% =========================================================

CART --> SALE

SALE --> INVENTORY

INVENTORY --> CHECKOUT

CHECKOUT --> PAYMENT

PAYMENT --> ORDER


%% =========================================================
%% INVENTORY CONSISTENCY
%% =========================================================

INVENTORY --> INVENTORYDB
INVENTORY --> RESERVATIONDB


%% =========================================================
%% PAYMENT
%% =========================================================

PAYMENT --> PAYMENTDB
PAYMENT --> PAYGATEWAY

PAYGATEWAY --> PAYMENT


%% =========================================================
%% EVENT FLOW
%% =========================================================

INVENTORY --> OUTBOX
PAYMENT --> OUTBOX
ORDER --> OUTBOX

OUTBOX --> BUS

BUS --> FULFILMENT
BUS --> SHIPMENT
BUS --> NOTIFICATION


%% =========================================================
%% DOWNSTREAM
%% =========================================================

FULFILMENT --> SHIPMENT

SHIPMENT --> CARRIER

NOTIFICATION --> MSG


%% =========================================================
%% ORDER DATA
%% =========================================================

ORDER --> ORDERDB


%% =========================================================
%% STYLING
%% =========================================================

classDef client fill:#E8F1FF,stroke:#2563EB,stroke-width:2px,color:#0F172A

classDef edge fill:#F8FAFC,stroke:#475569,stroke-width:2px,color:#0F172A

classDef service fill:#FFFFFF,stroke:#334155,stroke-width:1.5px,color:#0F172A

classDef critical fill:#FFF1F2,stroke:#DC2626,stroke-width:3px,color:#991B1B

classDef event fill:#F5F3FF,stroke:#7C3AED,stroke-width:2px,color:#4C1D95

classDef data fill:#EFF6FF,stroke:#2563EB,stroke-width:1.5px,color:#1E3A8A

classDef external fill:#F0FDF4,stroke:#16A34A,stroke-width:2px,color:#14532D


class CLIENT client

class CDN,WAF,TRAFFIC,QUEUE,LB,API edge

class CATALOG,SEARCH,CART,SALE,CHECKOUT,PAYMENT,ORDER service

class INVENTORY critical

class OUTBOX,BUS,FULFILMENT,SHIPMENT,NOTIFICATION event

class CACHE,PRODUCTDB,INVENTORYDB,RESERVATIONDB,PAYMENTDB,ORDERDB data

class PAYGATEWAY,CARRIER,MSG external
