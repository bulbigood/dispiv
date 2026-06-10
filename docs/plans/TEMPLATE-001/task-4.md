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

# Задача: Рефакторинг Сервиса (Переход на Outbox)

## Цель
Перевести `UserRegistrationService` с синхронного вызова `EmailGateway` на паттерн Transactional Outbox. Вместо прямой отправки письма необходимо формировать доменное событие и публиковать его через `OutboxEventPublisher`.

## Бизнес-инварианты (Из Спецификации)
> **INV-03 (Атомарность Outbox):**
> Создание записи `User` и события `UserRegisteredEvent` происходит в одной БД-транзакции. Метод должен быть обернут в `@Transactional`.

## Критерии приёмки (Acceptance Criteria)
1. Удалить внедрение зависимости `EmailGateway` из `UserRegistrationService`.
2. Внедрить зависимость `OutboxEventPublisher`.
3. В методе `registerUser` заменить прямой вызов отправки письма на формирование `UserRegisteredEvent` и вызов `outboxEventPublisher.publish()`.