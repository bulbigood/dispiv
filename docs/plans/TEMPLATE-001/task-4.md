---
task_id: t4_service_refactor
manifest_path: ./manifest.plan.md
requires:
  - t3_outbox_publisher
---

# Задача: Service Refactor (Phase: Migrate)

## Метаданные контекста
* **Target File (Update):** `src/.../service/UserRegistrationService.java`
* **Target File (Keep):** `src/.../gateway/EmailGateway.java` (Пока не удалять сам класс, убрать только вызовы из сервиса).
* **Preconditions:** Entity `OutboxEvent` и компонент `OutboxEventPublisher` уже созданы в предыдущих PR и доступны в проекте.

## Цель (Context & Goal)
Перевести `UserRegistrationService` с синхронного вызова `EmailGateway` на паттерн Transactional Outbox. Вместо прямой отправки письма необходимо формировать доменное событие и публиковать его через `OutboxEventPublisher`.

## Необходимые бизнес-правила (Из Spec)
*Так как ты выполняешь строго изолированную задачу, соблюдай следующие бизнес-инварианты:*

> **INV-03 (Атомарность Outbox):**
> Создание записи `User` и события `UserRegisteredEvent` происходит в одной БД-транзакции. Метод должен быть обернут в `@Transactional` (используй конфигурацию из `.dispiv/styleguide.md`).

## Acceptance Criteria (Ожидаемый результат AST)
1. Удалить внедрение зависимости `EmailGateway` из `UserRegistrationService`.
2. Внедрить зависимость `OutboxEventPublisher`.
3. В методе `registerUser` удалить прямой вызов отправки письма.
4. Сформировать `UserRegisteredEvent` (с `userId`, `email` и сгенерированным `token`) и передать в `publish()`.
5. ⚠️ **Важно:** Сохранение в Redis остается без изменений (TTL = 24h).