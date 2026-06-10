---
id: USER-REGISTRATION-SPEC
status: Active
affected_by_adrs:
  - path: /docs/adrs/TEMPLATE.adr.md
related_specs:
  - path: /docs/specs/EMAIL-GATEWAY.spec.md
tags: ["auth", "core"]
---

# Спецификация: User Registration

## 1. Область видимости (Scope)

Модуль `UserRegistrationService` **отвечает за**:

- Прием и строгую валидацию регистрационных данных.
- Создание неактивной учетной записи.
- Отправку асинхронного события для email-верификации.
- Подтверждение почты через одноразовый токен.

**Не входит в scope** (покрывается другими спецификациями):

- Аутентификация / Выдача JWT-токенов (`AuthService`).
- Восстановление или смена пароля.
- Физическая отправка писем по SMTP (`NotificationWorker`).

---

## 2. Бизнес-правила и Инварианты (Invariants)

_Это ключевые правила модуля. Они должны проверяться в изолированных Unit-тестах._

| ID         | Суть правила              | Формальная логика                                                                        |
| :--------- | :------------------------ | :--------------------------------------------------------------------------------------- |
| **INV-01** | **Уникальность Email**    | Проверяется строго без учета регистра: `email.toLowerCase()`.                            |
| **INV-02** | **Безопасность пароля**   | Не менее 8 символов, минимум одна цифра. Хранится только хэш (BCrypt, cost=12).          |
| **INV-03** | **Атомарность (Outbox)**  | Создание записи `User` и события `UserRegisteredEvent` происходит в одной БД-транзакции. |
| **INV-04** | **Жизненный цикл токена** | Токен имеет TTL $24 \text{ часа}$. После одного успешного применения токен уничтожается. |

---

## 3. Требования к хранению (Storage)

| Сущность            | Хранилище  | Ограничения (Constraints)                                                     |
| :------------------ | :--------- | :---------------------------------------------------------------------------- |
| `User`              | PostgreSQL | PK: `id` (UUID). UNIQUE: `email_lower`. Поле `is_verified` (default _false_). |
| `VerificationToken` | Redis      | Key: `reg_tkn:{token}`. Value: `userId`.                                      |

> **Для AI (Implement Phase):** Redis используется для токенов верификации из-за нативного TTL. Вызов Redis должен быть обернут в Circuit Breaker — при падении кэша регистрация отклоняется.

---

## 4. Тестовые сценарии (Test Sketches)

_Сокращенный маппинг для генерации тестов на этапе Plan. Формат: `[Задействованные инварианты] Входные условия ➔ Результат и Side Effects`_.

### Команда: `registerUser()`

- `[INV-01, 03]` Валидные данные ➔ Создан `User` (по умолчанию `is_verified=false`) **+** Сохранен токен в Redis **+** Опубликовано событие `UserRegisteredEvent` (атомарно).
- `[INV-01]` Дубликат Email ➔ Выброс `DuplicateEmailException`. Никаких событий не публикуется.
- `[INV-02]` Пароль < 8 символов ➔ Выброс `ValidationException(WEAK_PASSWORD)`. БД игнорируется.

### Команда: `verifyEmail()`

- `[INV-04]` Валидный токен ➔ `user.is_verified = true` **+** Токен удален из Redis.
- `[INV-04]` Токен не найден (или истек TTL) ➔ Выброс `ExpiredTokenException`.
- `[-]` Пользователь уже верифицирован ➔ Идемпотентный ответ `200 OK` без изменения БД.

---

<details>
<summary><b>5. Технические Контракты и DTO (Развернуть)</b></summary>

### Входящие команды (Inputs)

```java
public record RegisterUserCommand(
    String email,
    String rawPassword,
    String displayName
) {}

public record VerifyEmailCommand(String token) {}
```
