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
    Maximum 100 successful
       reservations
            ↓
      No overselling