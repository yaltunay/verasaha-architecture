# ADR-004 Outbox Pattern for Notifications

*This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.*

## Status

Accepted

## Context

The platform must send background notifications (e.g. push notifications via FCM) when domain events occur (e.g. meeting reminder). Sending directly from request-handling code would tie HTTP response time to external service latency and would risk losing notifications if the process crashes or the external call fails after the transaction is committed. We need reliable, at-least-once delivery and clear separation between domain persistence and notification dispatch.

## Decision Drivers

1. **Reliability:** Notifications must not be lost if the process crashes or FCM fails after commit; same-transaction write to Outbox guarantees durability.
2. **Request path decoupling:** HTTP response must not wait on FCM; background worker handles retry, backoff, and dead-letter.
3. **Consistency:** At-least-once delivery with idempotency keys; eventual consistency is acceptable for reminders and similar use cases.

## Decision

We use the **Outbox pattern for notifications**:

- **Background notifications:** Notifications are not sent synchronously from controllers or handlers. When a domain event that triggers a notification occurs, a row is written to a **NotificationOutbox** table in the same database transaction as the domain change. The HTTP response can complete without waiting for FCM or any other external service.

- **Reliable message dispatch:** A background worker (or scheduled job) reads rows from NotificationOutbox and sends them to FCM (or a noop sender in development). The worker can retry failed sends, apply backoff, and move permanently failed items to a dead-letter path. Handlers and controllers do not call FCM directly; delivery is decoupled from the request path.

- **Retry strategy:** The worker implements retries with backoff for transient failures. Idempotency keys (e.g. TenantId + IdempotencyKey) prevent the same logical event from being enqueued twice and allow safe retries. At-least-once delivery is the goal; duplicate suppression is handled at the consumer or by idempotent notification semantics where possible.

- **Eventual consistency:** Notifications are eventually sent after the domain change is committed. There is no strong guarantee of immediate delivery; the system prioritises not losing notifications over strict real-time delivery. This is acceptable for reminders and similar use cases.

## Alternatives Considered

- **Send directly from request handler:** Would block the response on FCM and risk losing the notification if the process dies after commit but before send. Rejected.

- **External message queue only:** Would add infrastructure and operational complexity; the Outbox keeps the “enqueue” step in the same transaction as the domain write, avoiding dual-write and consistency issues. External queue can be added later for scaling. Rejected for initial design.

- **Fire-and-forget from handler:** No durability; crash or failure after commit loses the notification. Rejected.

## Consequences

- **Positive:** Notifications are durable once the transaction commits; request path is fast and independent of FCM; retry and dead-letter handling are centralised in the worker; same transaction ensures no “domain saved but notification lost” gap.

- **Negative:** Notifications are eventually consistent; worker must be run and monitored.
- **Negative:** IdempotencyKey design and format must be documented; noop sender in dev avoids FCM but behaviour differs from production.

## When to Revisit

Revisit if: strong real-time delivery guarantee is required (e.g. critical alerts); or scale demands moving from DB Outbox to a dedicated message broker (e.g. RabbitMQ, SQS).
