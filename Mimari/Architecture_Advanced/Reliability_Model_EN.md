This document describes the architectural approach of the VeraSaha platform and is intended for technical reference only. It does not constitute legal advice or a formal security certification.

## API failure scenarios

The API is designed so that common failure modes are explicit and diagnosable. Authentication and authorization errors (401/403) are distinguished from business conflicts (409) and overload or infrastructure issues (429/5xx). When dependencies such as the database or storage are unavailable, the API should fail fast rather than hang indefinitely, surfacing clear errors to clients and logs. Tenant resolution failures (invalid or missing `X-Tenant-Key`) are treated as client errors, preserving the principle that every protected call must declare a tenant.

## Sync conflict handling

Sync is built around an offline-first model with `changes` and `apply` endpoints. Conflicts are minimized by using timestamps and tombstones, but not every race can be prevented. The base model is "last successful apply wins" for a given record, with the expectation that domain rules will be added where stricter semantics are needed. Because changes are recorded and correlated via SyncInbox and audit logging, operators can reconstruct sequences of updates and, in exceptional cases, correct data manually.

## Idempotent operations

Idempotency is a core reliability tool for sync and background processes. The SyncInbox table, keyed by `(TenantId, UserId, OpId)`, ensures that re-sending the same operation does not double-apply mutations. This makes retries safe across network glitches, client restarts, and transient API failures. Similar patterns apply to NotificationOutbox via idempotency keys for notification batches. Idempotency not only protects data integrity but also simplifies client error handling: clients can repeat requests without needing to know exactly where a previous attempt failed.

## Retry strategies

Retries are applied where they add value: background workers and clients are expected to retry transient failures (timeouts, temporary 5xx responses) with exponential backoff. The architecture discourages blind, tight loops; instead, retries are rate-limited and bounded to avoid storm effects. For synchronous API calls, clients should treat repeated 4xx errors as terminal and only retry on clearly transient conditions. This separation of transient vs permanent errors is critical to avoiding self-inflicted denial-of-service during partial outages.

## Notification retries

NotificationOutbox exists specifically to decouple delivery from domain writes and to support reliable retries. Workers read from the outbox, attempt delivery (e.g. to FCM), and on failure schedule subsequent attempts with backoff. After a configured number of attempts or specific error conditions (e.g. invalid token), items can be moved to a dead-letter queue or marked as permanently failed. This ensures that temporary outages of notification infrastructure do not cause silent data loss, and that problematic messages can be inspected and handled manually.

## Rate limit protections

Rate limiting protects both infrastructure and tenants from overload and abuse. When limits are exceeded, the system responds with 429 or other clear status codes, signaling clients to back off. Because rate limiting can itself trigger retries, it must be combined with guidance (e.g. Retry-After headers or client conventions) to prevent thrashing. Properly configured limits ensure that in failure scenarios—such as partial outages or hot tenants—core services remain responsive for most traffic instead of degrading uniformly for everyone.

## Circuit breaker thinking

While not tied to a particular library, the architecture encourages "circuit breaker" style thinking: if a downstream system is failing persistently, clients and workers should fail fast or degrade behavior rather than continuing to send full traffic. Examples include temporarily skipping non-critical integrations, reducing sync frequency, or delaying heavy background jobs. The goal is to localize impact and give operators time to repair issues without the system continuously amplifying failures.

## Observability detection

Reliability depends on early detection of anomalies. Structured logs, metrics, and traces provide inputs for alerting on error rates, latency spikes, queue depths, and unusual access patterns. Correlation IDs and tenant-aware context allow operators to identify whether issues are global, per-tenant, or tied to specific features. The design assumes that alerts should be actionable: dashboards and queries exist that link "something looks wrong" to "here is the failing component and likely cause."

## Manual recovery procedures

Not all failure modes can be fully automated away. The architecture therefore emphasizes traceability (AuditLog, SyncInbox, NotificationOutbox) and clear invariants so that manual interventions can be safe and repeatable. Example procedures include replaying or cancelling specific sync operations, re-queuing stuck notifications, temporarily disabling problematic tenants, or restoring data from backups for a bounded scope. Documented runbooks and ADRs provide the context needed for staff-level engineers to design and execute these interventions without guesswork.

