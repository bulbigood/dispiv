---
task_id: t4_service_refactor
sandbox_policy:
  allow_write:
    - src/main/java/com/app/service/UserRegistrationService.java
  allow_read:
    - src/main/java/com/app/gateway/EmailGateway.java
    - src/main/java/com/app/domain/OutboxEventPublisher.java
on_blocker:
  artifact: docs/plans/USER-REGISTRATION-PLAN-001/blockers/t4-blocker.md
  notify: plan_id: USER-REGISTRATION-PLAN-001
---

# Task: Migrate UserRegistrationService to the Transactional Outbox Pattern

## Goal

Migrate `UserRegistrationService` from a synchronous `EmailGateway` invocation to the Transactional Outbox pattern.

Instead of sending emails directly, the service must create a domain event and publish it through `OutboxEventPublisher`.

## Business Invariants (From the Specification)

> **INV-03 (Outbox Atomicity):**
>
> Creation of the `User` record and the corresponding `UserRegisteredEvent` must occur within the same database transaction.
>
> The method must be wrapped with `@Transactional`.

## Acceptance Criteria

1. Remove the `EmailGateway` dependency from `UserRegistrationService`.
2. Inject the `OutboxEventPublisher` dependency.
3. In the `registerUser` method, replace the direct email-sending call with:

   * creation of a `UserRegisteredEvent`;
   * invocation of `outboxEventPublisher.publish()`.
