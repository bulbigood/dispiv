---
id: USER-REGISTRATION-PLAN-001
strategy: expand-migrate-contract
affected_by_specs:
  - path: /docs/specs/TEMPLATE.spec.md
dag:
  - id: t1_db_schema
    file: task-1.md
    emc_phase: expand
    spec_invariants: [INV-01,INV-02]
    depends_on: []
    sandbox_policy:
      allow_read:
        - src/main/java/com/app/db/FlywayService.java
      allow_write:
        - src/main/java/com/app/db/FlywayService.java
  - id: t2_kafka_dtos
    file: task-2.md
    emc_phase: expand
    spec_invariants: [INV-01,INV-02]
    depends_on: []
    sandbox_policy:
      allow_read:
        - src/main/java/com/app/kafka/dto/User.java
      allow_write:
        - src/main/java/com/app/kafka/dto/User.java
  - id: t3_outbox_publisher
    file: task-3.md
    emc_phase: expand
    spec_invariants: [INV-03,INV-04]
    depends_on: [t1_db_schema, t2_kafka_dtos]
    sandbox_policy:
      allow_read:
        - src/main/java/com/app/service/UserPublisher.java
      allow_write:
        - src/main/java/com/app/service/UserPublisher.java
  - id: t4_service_refactor
    file: task-4.md
    emc_phase: migrate
    spec_invariants: [INV-03]
    depends_on: [t3_outbox_publisher]
    sandbox_policy:
      allow_write:
        - src/main/java/com/app/service/UserRegistrationService.java
      allow_read:
        - src/main/java/com/app/gateway/EmailGateway.java
        - src/main/java/com/app/domain/OutboxEventPublisher.java
  - id: t5_cleanup
    file: task-5.md
    emc_phase: contract
    spec_invariants: [INV-03]
    depends_on: [t4_service_refactor]
    sandbox_policy:
      allow_read:
        - src/main/java/com/app/service/UserRegistrationService.java
      allow_write:
        - src/main/java/com/app/service/UserRegistrationService.java
      allow_delete:
        - src/main/java/com/app/gateway/EmailGateway.java
---

# Implementation Plan: Outbox Migration for User Registration

> **Pipeline Roles**
>
> * 🤖 **Orchestrator:** Reads only the YAML header to build the DAG and launch agents.
> * 🧠 **Weak AI (Agents):** Do not read this manifest. They receive only their assigned `task-X.md` file.
> * 👁️ **Human:** Reviews this document to approve the architectural migration.

---

<details>
<summary><b>Architectural Delta (Gap Analysis)</b></summary>

| Responsibility Area | Target State (Spec)         | Current State                     | EMC Phase |
| ------------------- | --------------------------- | --------------------------------- | --------- |
| **Database Schema** | `outbox_events` table added | Only `users` table exists         | Expand    |
| **Events**          | `UserRegisteredEvent` DTO   | Not implemented                   | Expand    |
| **Integration**     | `OutboxEventPublisher`      | Not implemented                   | Expand    |
| **Business Logic**  | Persist event into Outbox   | Synchronous `EmailGateway` call   | Migrate   |
| **Cleanup**         | Direct dependency removed   | Service depends on `EmailGateway` | Contract  |

</details>

<details>
<summary><b>Execution Graph Topology (Mermaid)</b></summary>

```mermaid
flowchart LR
    subgraph Group A [Parallel Phase: Expand]
        direction LR
        T1[t1: Database Schema]
        T2[t2: Event DTOs]
    end

    T1 --> T3
    T2 --> T3

    T3[t3: Publisher] --> T4[t4: Service Refactoring ⚠️ Migrate]
    T4 --> T5[t5: Legacy Cleanup<br/>Contract]
```

</details>
