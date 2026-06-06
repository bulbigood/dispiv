---
pipeline_id: DISPIV-CART-042
version: 1
created: 2026-06-07
author: planner-agent/v3
specs:
  - id: CART-SPEC-v3.0
    path: docs/specs/cart-spec-v3.0.md
  - id: INVENTORY-SPEC-v1.2
    path: docs/specs/inventory-spec-v1.2.md
adr: docs/adrs/042-cart-v3-migration.md
strategy: expand-migrate-contract
dag:
  - id: t1_dto
    file: task-1.md
    emc_phase: expand
    depends_on: []
    estimated_tokens: 2000
  - id: t2_db
    file: task-2.md
    emc_phase: expand
    depends_on: []
    estimated_tokens: 3000
  - id: t3_api
    file: task-3.md
    emc_phase: migrate
    depends_on: [t1_dto, t2_db]
    breaks_compilation: true
    affected_files:
      - src/main/java/com/shop/cart/controller/CartController.java
      - src/main/java/com/shop/cart/service/LegacyCartService.java
    estimated_tokens: 5000
  - id: t4_cleanup
    file: task-4.md
    emc_phase: contract
    depends_on: [t3_api]
    estimated_tokens: 1500
---

# Cart API v3.0 — Implementation Plan

## Gap Analysis Summary

| Area | Spec v3.0 | Current Implementation | Delta | EMC Phase |
|:---|:---|:---|:---|:---|
| **REST API** | `POST /api/v3/cart/items` | `POST /api/v1/cart/add` | Rename + new payload | Migrate |
| **State** | `CartState` (ACTIVE, PAID) | Boolean `isPaid` | Replace + enforce | Expand → Contract |
| **Storage** | Database Table `cart_items` | In-memory `ConcurrentHashMap` | Migrate to PostgreSQL | Expand → Migrate |
| **DTO** | `CartItemRequest` | `AddToCartForm` | New class, deprecate old | Expand |

## Execution Graph

```mermaid
flowchart LR
    T1[T1: DTOs<br/>expand] --> T3[T3: API Refactor<br/>migrate]
    T2[T2: DB Layer<br/>expand] --> T3
    T3 --> T4[T4: Cleanup<br/>contract]
    
    style T1 fill:#2d5016,stroke:#4a7c23
    style T2 fill:#2d5016,stroke:#4a7c23
    style T3 fill:#b8860b,stroke:#daa520
    style T4 fill:#8b0000,stroke:#dc143c
```

## Compile-Safe Ordering & Execution Strategy

| Step | Task | EMC Phase | Breaks? | Parallel Group | Rollback |
|:---|:---|:---|:---|:---|:---|
| 1 | T1: DTOs | Expand | No | A | Delete new files |
| 1 | T2: DB Layer | Expand | No | A | Delete new files |
| 2 | T3: API Refactor | Migrate | **Yes** | B | Revert T3, restore old API |
| 3 | T4: Cleanup | Contract | No | C | N/A (irreversible) |

> **⚠️ Group B Warning:** После T3 проект не компилируется до завершения T3. 
> Оркестратор должен выполнить T3 атомарно. При провале — откат к состоянию после Group A.