---
id: USER-REGISTRATION-ADR-001
status: Accepted
related_specs:
  - path: /docs/specs/TEMPLATE.spec.md
tags: ["messaging", "kafka", "resilience"]
---

# ADR-001: Asynchronous Email Verification During User Registration

> **Summary:** User registration requests must not be blocked by email delivery. User creation and verification token generation occur synchronously, while email delivery is delegated to an asynchronous consumer via Kafka using the Transactional Outbox pattern.

---

## 1. Context and Problem

In the synchronous registration flow, a user clicks "Register", the server creates a database record, and then initiates an SMTP session to send a verification email.

**Problem:**

API response time degrades because of external dependencies. Mathematically, response latency can be represented as:

$$
T_{response} = T_{db_commit} + T_{smtp_handshake} + T_{email_send}
$$

Under load or during SMTP provider outages, $T_{response}$ may reach 10–15 seconds, causing client-side timeouts.

**Business Requirement:**

P99 latency for the `/api/v1/users/register` endpoint must not exceed $250 \text{ ms}$ regardless of email provider availability.

---

## 2. Decision

We are adopting an Event-Driven Architecture.

**Selected Approach: Transactional Outbox + Kafka + Dedicated Notification Service**

### Architectural Concept

The registration service no longer depends directly on `EmailGateway`.

Instead, it produces a domain event containing `userId` and the generated verification token. This event is guaranteed to be delivered to Kafka.

*Exact DTO definitions are specified in `TEMPLATE.spec.md`.*

### System Topology

1. `UserRegistrationService` writes both the `User` entity and an `outbox_events` record within the same PostgreSQL transaction.
2. A change-data-capture process (Debezium) streams changes from `outbox_events` into a Kafka topic.
3. The `NotificationWorker` microservice consumes the topic, renders an HTML template, and asynchronously calls the email provider's HTTP API.

---

## 3. Consequences

### Positive

* **Improved Responsiveness:** `/register` latency is now limited to PostgreSQL persistence time only (< 50 ms).
* **Resilience:** Registration succeeds even if the email provider is unavailable. Events accumulate safely in Kafka.
* **Scalability:** Email delivery can be scaled independently from the registration API.

### Negative

* **Infrastructure Overhead:** Requires maintaining Kafka and Debezium infrastructure.
* **Eventual Consistency:** Users may occasionally receive verification emails with a short delay (1–3 seconds). The UI must display a "Verification email sent" screen instead of immediately activating the account.

---

<details>
<summary><b>4. Considered Alternatives (Rejected Options)</b></summary>

### ❌ Option A: Synchronous Delivery with Thread Pool (`@Async`)

Use an internal thread pool and send emails in a background thread.

**Why Rejected**

Unsent emails stored only in memory are lost during pod restarts, crashes, or deployments. Reliable retry mechanisms become significantly more complex.

### ❌ Option B: In-Process Events (Spring `@EventListener`)

Publish events inside the JVM and process them using another bean.

**Why Rejected**

Does not solve message durability problems during process failures. Also increases coupling between the monolith and external heavyweight integrations.

</details>

<details>
<summary><b>5. Pre-Mortem Analysis Results (Risk Mitigation)</b></summary>

| Failure Mode                                    | Probability / Impact | Mitigation Strategy                                                                                                              |
| ----------------------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Kafka event loss due to broker outage           | Low / Critical       | Guaranteed by the Transactional Outbox pattern and producer configuration `acks=all`.                                            |
| Email delivery failure (bounce)                 | Medium / High        | Consumer must not fail. Hard bounces transition the user to `UNDELIVERABLE`. Soft bounces use exponential backoff retries.       |
| User repeatedly requests resending verification | High / Medium        | Introduce `/resend-verification` endpoint with a strict rate limit of one request every 60 seconds per user, enforced via Redis. |

</details>
