---
id: USER-REGISTRATION-SPEC
status: Active
affected_by_adrs:
  - path: /docs/adrs/TEMPLATE.adr.md
related_specs:
  - path: /docs/specs/EMAIL-GATEWAY.spec.md
tags: ["auth", "core"]
---

# Specification: User Registration

## 1. Scope

The `UserRegistrationService` module is **responsible for**:

* Receiving and strictly validating registration data.
* Creating an inactive user account.
* Publishing an asynchronous email verification event.
* Confirming email ownership through a one-time verification token.

**Out of Scope** (covered by other specifications):

* Authentication and JWT token issuance (`AuthService`).
* Password reset and password change flows.
* Physical email delivery via SMTP (`NotificationWorker`).

---

## 2. Business Rules and Invariants

*These are the core rules of the module and must be verified through isolated unit tests.*

| ID         | Rule                  | Formal Logic                                                                                                 |
| :--------- | :-------------------- | :----------------------------------------------------------------------------------------------------------- |
| **INV-01** | **Email Uniqueness**  | Email uniqueness is enforced case-insensitively using `email.toLowerCase()`.                                 |
| **INV-02** | **Password Security** | Minimum length of 8 characters and at least one digit. Only a password hash is stored (BCrypt, cost=12).     |
| **INV-03** | **Outbox Atomicity**  | Creation of the `User` record and the `UserRegisteredEvent` must occur within the same database transaction. |
| **INV-04** | **Token Lifecycle**   | Verification tokens have a TTL of $24 \text{ hours}$. A token is destroyed immediately after successful use. |

---

## 3. Storage Requirements

| Entity              | Storage    | Constraints                                                                     |
| :------------------ | :--------- | :------------------------------------------------------------------------------ |
| `User`              | PostgreSQL | PK: `id` (UUID). UNIQUE: `email_lower`. Field `is_verified` (default: `false`). |
| `VerificationToken` | Redis      | Key: `reg_tkn:{token}`. Value: `userId`.                                        |

<details data-audience="implement-agent">
<summary>Implementation Notes</summary>

Redis is protected by a Circuit Breaker. If Redis becomes unavailable, registration requests must be rejected.

</details>

---

## 4. Test Scenarios (Executable Test Sketches)

*A condensed mapping used for test generation during the Plan phase.*

*Format: `[Relevant Invariants] Preconditions ➔ Expected Result and Side Effects`*

### Command: `registerUser()`

* `[INV-01, INV-03]` Valid input ➔ `User` created (`is_verified=false` by default) **+** verification token stored in Redis **+** `UserRegisteredEvent` published (atomically).
* `[INV-01]` Duplicate email ➔ Throw `DuplicateEmailException`. No event is published.
* `[INV-02]` Password shorter than 8 characters ➔ Throw `ValidationException(WEAK_PASSWORD)`. Database remains untouched.

### Command: `verifyEmail()`

* `[INV-04]` Valid token ➔ `user.is_verified = true` **+** token removed from Redis.
* `[INV-04]` Token not found (or TTL expired) ➔ Throw `ExpiredTokenException`.
* `[-]` User already verified ➔ Return idempotent `200 OK` response without modifying the database.

---

<details>
<summary><b>5. Technical Contracts and DTOs (Expand)</b></summary>

### Incoming Commands (Inputs)

```java
public record RegisterUserCommand(
    String email,
    String rawPassword,
    String displayName
) {}

public record VerifyEmailCommand(String token) {}
```

</details>
