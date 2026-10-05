# SALESTORM — High-Scale E-Commerce Flash Sale System

> **SYSCRAFTERS 2026 | Design-First, AI-Assisted System Design Hackathon**

## 1. Problem Statement

SALESTORM is a high-scale e-commerce flash-sale platform designed to handle extreme traffic spikes when thousands of customers compete for limited inventory.

### Critical Scenario

- **Concurrent purchase requests:** 10,000
- **Available inventory:** 100 units
- **Payment success rate:** 95%
- **Payment failure rate:** 5%
- **Duplicate requests:** 2%
- **Order Service outage:** 30 seconds

### Core Challenge

> How can we handle thousands of simultaneous purchase requests for limited stock without overselling, while keeping payment and order processing reliable?

The primary invariant of the system is:

**Successful Reservations ≤ Available Inventory**

For the critical scenario:

```text
10,000 concurrent requests
            ↓
     100 available units
            ↓
       No overselling
```

## System Design Documentation Roadmap

1. [01_Requirements — Requirements & Assumptions](file:///c:/Users/aswin/OneDrive/Documents/NAALVAR_SALESTORM_SYSTEM-DESIGN/01_Requirements/README.md)
2. `02_HLD` — High-Level Architecture & End-to-End Flow
3. `03_LLD` — Low-Level Component Design & State Machines
4. `04_Database` — Data Models, Indexing & Concurrency Controls
5. `05_API` — API Specifications & Idempotency Handshakes
6. `06_SOLID` — Object-Oriented & Modular Architectural Principles
7. `07_Design_Patterns` — Distributed & Domain Design Patterns
8. `08_Scalability_Reliability` — Surge Handling, Queuing & Failure Recovery
9. `09_Security_Observability` — Zero-Trust, Distributed Tracing & Telemetry
10. `10_ADR` — Architectural Decision Records
11. `11_AI_Assisted_Validation` — Simulation & Validation Reports
12. `12_Presentation` — Executive Summary & Presentation Materials