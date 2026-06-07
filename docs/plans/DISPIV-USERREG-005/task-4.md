---
module: USER-REGISTRATION
status: Active
dependencies:
  - "EMAIL-GATEWAY"
---

# Task 3: Service Refactor (Migrate to Outbox)

## Target Files

| File | Action | Status |
|:---|:---|:---|
| `src/.../service/UserRegistrationService.java` | Modify | Existing |
| `src/.../gateway/EmailGateway.java` | Keep (for now) | Existing (Will be removed in T4) |

## Context & Goal

Выполни фазу **Migrate**. Переведи `UserRegistrationService` с синхронного вызова `EmailGateway` на сохранение события в `outbox_events` через `OutboxEventPublisher`. 
Это гарантирует инвариант **INV-03** (Атомарность регистрации: User и событие создаются в одной транзакции БД).

## Preconditions (from depends_on)

- [x] T1: Таблица `outbox_events` и JPA Entity `OutboxEvent` существуют.
- [x] T2: Бин `OutboxEventPublisher` с методом `publish(DomainEvent event)` существует и инжектится.

## Interface & Signatures

### File: `UserRegistrationService.java` (Modify)

```java
// 1. REMOVE DEPENDENCY:
// private final EmailGateway emailGateway;

// 2. ADD DEPENDENCY:
private final OutboxEventPublisher outboxPublisher;

// 3. MODIFY METHOD: registerUser()
@Transactional
public RegistrationResult registerUser(RegisterUserCommand cmd) {
    // ... existing validation and user creation ...
    User user = userRepository.save(newUser);
    
    String token = tokenGenerator.generate();
    redisTemplate.opsForValue().set("reg_tkn:" + token, user.getId().toString(), 24, TimeUnit.HOURS);

    // REMOVE SYNC CALL:
    // emailGateway.sendVerification(user.getEmail(), token);

    // ADD OUTBOX PUBLISH:
    UserRegisteredEvent event = new UserRegisteredEvent(
        user.getId(), 
        user.getEmail(), 
        token, 
        Instant.now()
    );
    outboxPublisher.publish(event);

    return new RegistrationResult(user.getId(), token);
}
```

## Logical Specification & Invariants Check

| Invariant | How this task satisfies it |
|:---|:---|
| **INV-03** (Атомарность) | Метод помечен `@Transactional`. `userRepository.save()` и `outboxPublisher.publish()` (который делает `outboxRepository.save()`) выполняются в одной транзакции PostgreSQL. Если падает БД, письмо не уйдет. |

## Compilation Impact

⚠️ После этой задачи `UserRegistrationService` больше не зависит от `EmailGateway`. 
Однако `EmailGateway` пока **не удаляется**, так как он может использоваться в других модулях (например, Password Reset). Удаление связи с регистрацией произойдет в T4.

## Rollback Strategy

Если задача провалилась или тесты не прошли:
1. `git checkout HEAD -- src/.../service/UserRegistrationService.java`
2. Убедиться, что синхронный вызов `emailGateway.sendVerification()` восстановлен.
3. Вернуть выполнение в очередь с логом ошибки для $AI_{strong}$.