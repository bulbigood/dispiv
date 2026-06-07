---
module: USER-REGISTRATION
version: "1.1.0"
status: Active
owner: "@alice"
idea_ref: "/docs/adrs/005-async-email-verification.md"
dependencies:
  - "EMAIL-GATEWAY.spec.md"
tags: ["auth", "core"]
changelog:
  - version: "1.1.0"
    date: 2026-06-07
    changes: "Переход на асинхронную отправку писем через Transactional Outbox (см. ADR-005)."
  - version: "1.0.0"
    date: 2026-05-10
    changes: "Первоначальная версия с синхронной отправкой."
---

# User Registration — Спецификация v1.0

> **Scope**: Регистрация новых пользователей, валидация данных, верификация email.
> **Вне scope**: Аутентификация (Login), сброс пароля, OAuth.
> **Целевая аудитория**: Человек, Strong AI (Plan/Verify), Weak AI (Implement).

---

## 1. Назначение и границы (Scope)

`UserRegistrationService` отвечает за:
- Прием и валидацию регистрационных данных.
- Создание неактивной учетной записи и генерацию токена.
- Подтверждение email по токену.

**Не отвечает** за:
- Выдачу JWT-токенов (это `AuthService`).
- Физическую отправку писем (делегировано `EmailGateway`).

---

## 2. Контракты и Модели Данных (Contracts)

### 2.1. Входящие команды и Результаты

```java
public record RegisterUserCommand(
    String email, 
    String rawPassword, 
    String displayName
) {}

public record RegistrationResult(
    UUID userId, 
    String verificationToken
) {}

public record VerifyEmailCommand(String token) {}
```

### 2.2. Исключения (Domain Exceptions)

```java
public class DuplicateEmailException extends DomainException { ... }
public class ValidationException extends DomainException {
    private final ErrorCode code; // WEAK_PASSWORD, INVALID_EMAIL
}
public class ExpiredTokenException extends DomainException { ... }
```

---

## 3. Инварианты и Бизнес-правила (Invariants)

| ID | Правило | Формальное описание / Логика |
|:---|:---|:---|
| **INV-01** | **Уникальность Email** | Email проверяется без учета регистра: `email.toLowerCase()`. |
| **INV-02** | **Требования к паролю** | Минимум 8 символов, хотя бы одна цифра. Хранится только BCrypt-хэш (cost=12). |
| **INV-03** | **Атомарность регистрации** | Запись `User` и `VerificationToken` создаются в одной транзакции. |
| **INV-04** | **Токен одноразовый** | После успешной верификации токен удаляется. Повторный вызов с тем же токеном возвращает ошибку `InvalidToken`. |

---

## 4. Требования к хранению (Storage)

| Сущность | Хранилище | Constraints & Indexes | Специфика |
|:---|:---|:---|:---|
| `User` | PostgreSQL | PK: `id` (UUID). UNIQUE: `email_lower`. | Поле `is_verified` (boolean, default false). |
| `VerificationToken` | Redis | Key: `reg_tkn:{token}` | TTL: строго 24 часа. Value: `userId`. |

*Примечание для Plan-агента: Redis используется для токенов из-за встроенного TTL и высокой скорости чтения. При недоступности Redis регистрация блокируется (Circuit Breaker).*

---

## 5. Тестовые сценарии (Test Sketches)

### `registerUser(RegisterUserCommand cmd)`
* **happy path** `[INV-01, INV-03]`: email уникальный, пароль валидный → создается `User` (is_verified=false), генерируется токен, сохраняется в Redis (TTL 24h). **[SideEffect]**: асинхронно вызывается `EmailGateway.send()`.
* **duplicate email** `[INV-01]`: email уже есть в БД → `DuplicateEmailException`. **[SideEffect]**: `EmailGateway` НЕ вызывается.
* **weak password** `[INV-02]`: пароль < 8 символов → `ValidationException(WEAK_PASSWORD)`. БД и Redis не затрагиваются.

### `verifyEmail(VerifyEmailCommand cmd)`
* **happy path** `[INV-04]`: токен найден в Redis → `user.is_verified = true`, токен удаляется из Redis.
* **expired token** `[INV-05]`: токен не найден в Redis (истек TTL) → `ExpiredTokenException`.
* **already verified**: если пользователь уже верифицирован → идемпотентный ответ `200 OK` (без исключения).

---

## 6. Миграция и Версионирование

- **Обратная совместимость**: Добавление новых полей в `RegisterUserCommand` (например, `phoneNumber`) должно быть опциональным (`Optional<String>`).
- **Expand-Migrate-Contract**: Если потребуется изменить алгоритм хэширования паролей с BCrypt на Argon2, это потребует трехфазного плана (поддержка обоих хэшей на чтение -> миграция данных -> удаление BCrypt).