# ADR-005 Offline Sync Architecture

*This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.*

## Status

Accepted

## Context

Mobile clients (e.g. Flutter) operate in environments where connectivity is intermittent. Users must be able to view and modify data offline and have changes synchronised when the connection is restored. We need a sync model that supports offline-first clients, avoids duplicate application of mutations, and keeps client and server eventually consistent without conflicting or lost updates.

## Decision Drivers

1. **Offline-first UX:** Field users need to work without connectivity; sync must support delta and idempotent apply so retries are safe.
2. **Data integrity:** Duplicate OpId must not double-apply mutations; SyncInbox (TenantId, UserId, OpId) enforces idempotency and tenant isolation.
3. **Operational clarity:** Single sync contract (changes + apply); UTC and tombstone model keep client and server aligned.

## Decision

We adopt an **offline sync architecture** with the following elements:

- **Offline-first mobile clients:** Clients maintain a local store and can read and write data without being online. All mutations are recorded locally and sent to the server when possible. The UI can work from local data; sync runs in the background. The server exposes a **changes** endpoint (delta of changes since a given timestamp) and an **apply** endpoint (client sends a batch of mutations).

- **Delta synchronization:** The server’s **changes** endpoint returns only entities (or relevant fields) that were created, updated, or deleted since the client’s `since` parameter (e.g. serverTimeUtc). Tombstones (deleted markers) are included so the client can remove local copies. This minimises payload size and allows incremental sync. All timestamps are UTC (ISO 8601).

- **Idempotent apply endpoint:** The client sends mutations with a unique **OpId** per operation. The server stores applied operations in a **SyncInbox** table keyed by (TenantId, UserId, OpId). If the client retries the same OpId (e.g. after network failure), the server does not re-apply the mutation; it returns the previously stored result. This prevents double application and data corruption and makes retries safe.

- **Sync inbox pattern:** SyncInbox provides both idempotency and tenant/user isolation. Only the owning tenant and user can see or replay their operations. The server applies mutations in order where required and returns success or structured errors so the client can reconcile.

## Alternatives Considered

- **Always-online only:** Would not meet field-user needs where connectivity is poor. Rejected.

- **Last-write-wins without idempotency:** Would risk duplicate application on retry and inconsistent state. Rejected.

- **Server-authoritative conflict resolution (e.g. CRDTs):** Could be added for specific entities; for the initial design, client applies mutations with OpId idempotency and ordered apply where needed. More complex conflict handling can be introduced later.

## Consequences

- **Benefits:** Offline-first UX; safe retries via OpId; delta sync reduces bandwidth; SyncInbox enforces tenant and user isolation; UTC and tombstone model keep client and server aligned.

- **Trade-offs:** Client must generate and persist OpIds; server must maintain SyncInbox (retention policy may be applied). Conflict resolution beyond “last apply wins” or ordering is not in scope for this ADR; product requirements may drive future refinements.

## When to Revisit

Revisit if: conflict rate or product requirements demand CRDTs or server-authoritative conflict resolution; or SyncInbox retention or cardinality becomes an operational issue.
