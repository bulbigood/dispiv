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
        - src/main/java/com/app/db/UserRegistrationService.java
      allow_write:
        - src/main/java/com/app/db/UserRegistrationService.java
  - id: t2_kafka_dtos
    file: task-2.md
    emc_phase: expand
    spec_invariants: [INV-01,INV-02]
    depends_on: []
    sandbox_policy:
      allow_read:
        - src/main/java/com/app/kafka/KafkaService.java
      allow_write:
        - src/main/java/com/app/kafka/KafkaService.java
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
    depends_on: [t4_service_refactor]
    sandbox_policy:
      allow_read:
        - src/main/java/com/app/service/UserRegistrationService.java
      allow_write:
        - src/main/java/com/app/service/UserRegistrationService.java
      allow_delete:
        - src/main/java/com/app/gateway/EmailGateway.java
---

# План реализации: Outbox Migration для User Registration

> **Роли в пайплайне:**
>
> - 🤖 **Оркестратор**: Читает только YAML-заголовок (строит DAG и запускает агентов).
> - 🧠 **Weak AI (Агенты)**: НЕ читают этот манифест. Им передается исключительно файл `task-X.md`.
> - 👁️ **Человек**: Читает этот документ для апрува архитектурного перехода.

---

<details>
<summary><b>Архитектурная дельта (Gap Analysis)</b></summary>

| Зона ответственности | Целевое состояние (Spec)          | Текущее состояние               | Фаза (EMC) |
| :------------------- | :-------------------------------- | :------------------------------ | :--------- |
| **Схема БД**         | Добавлена таблица `outbox_events` | Только `users`                  | Expand     |
| **События**          | DTO `UserRegisteredEvent`         | Отсутствует                     | Expand     |
| **Интеграция**       | `OutboxEventPublisher`            | Отсутствует                     | Expand     |
| **Бизнес-логика**    | Запись события в Outbox           | Синхронный вызов `EmailGateway` | Migrate    |
| **Очистка**          | Прямая зависимость удалена        | Бин зависит от `EmailGateway`   | Contract   |

</details>

<details>
<summary><b>Топология графа выполнения (Mermaid)</b></summary>

```mermaid
flowchart LR
    subgraph Group A [Параллельная фаза: Expand]
        direction LR
        T1[t1: Схема БД]
        T2[t2: DTO События]
    end

    T1 --> T3
    T2 --> T3

    T3[t3: Publisher] --> T4[t4: Рефакторинг Сервиса ⚠️ Migrate]
    T4 --> T5[t5: Очистка легаси<br/>Contract]
```
