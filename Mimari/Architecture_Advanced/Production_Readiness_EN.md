This document describes the architectural approach of the VeraSaha platform and is intended for technical reference only. It does not constitute legal advice or a formal security certification.

## Deployment topology

The platform is deployed as a set of clearly separated components: a reverse proxy at the edge, web frontends (landing, admin, tenant UIs), a single backend API, background workers, a database, storage for files, and a CDN/file-serving layer. All external HTTP(S) traffic terminates at the reverse proxy, which routes by host to the appropriate backend. The API and workers run in an environment where configuration, secrets, and network boundaries are explicit, allowing deployments to be automated and rolled back without hidden coupling.

## Reverse proxy model

The reverse proxy (e.g. Nginx/NPM) is the single public entry point for `verasaha.com`, `app.verasaha.com`, `api.verasaha.com`, and `cdn.verasaha.com`. It terminates TLS, manages certificates (e.g. via Let’s Encrypt), and forwards requests over internal HTTP to the API and other services. Host-based routing keeps concerns separated and allows per-host policies (CORS, rate limiting, caching) while maintaining a simple mental model: nothing behind the proxy is directly exposed to the public internet.

## Secrets management

Secrets such as JWT signing keys, database connection strings, CDN signing keys, and FCM credentials are injected via environment variables or secret stores, not hardcoded in source or logged. The architecture assumes that secret rotation must be possible without code changes and that different environments will use different secrets. Config files and `.env` samples avoid real values and document what is required; production systems are expected to pull from a secure secret provider or CI/CD-managed configuration.

## Observability

Production readiness depends on having enough signal to understand behavior. The API and workers emit structured logs with correlation IDs, tenant/user context (where appropriate), and clear event names. Metrics (e.g. request rates, error rates, outbox depth) are exposed via a protected `/metrics` endpoint and scraped by a metrics backend. Optional distributed tracing ties together flows across controllers, handlers, and background workers. These building blocks make it feasible to answer questions like "what failed?", "for which tenant?", and "how often?" without guesswork.

## Failure handling

The system prefers explicit failure and bounded degradation over silent misbehavior. API endpoints return meaningful status codes (401/403, 409, 429, 5xx) when invariants are broken or capacity is exceeded. Sync contracts are designed so that partial failures can be retried without corrupting state. Outbox and SyncInbox introduce persistence layers that ensure work is not lost when processes crash. Combined with rate limiting and quotas, this reduces the chance that localized failures grow into systemic incidents.

## Background workers

Background workers process non-interactive workloads such as notification delivery from NotificationOutbox or other future asynchronous tasks. Their responsibilities are separated from request/response paths, allowing HTTP latency to remain low even when downstream services are slow. Workers implement retry policies with backoff and dead-letter behaviors so that transient issues do not cause permanent data loss, and persistent problems can be surfaced for manual intervention instead of silently looping.

## Quota enforcement

Quota enforcement prevents a single tenant from exhausting shared resources. Storage quotas bound how much data a tenant can upload; attempts to exceed quotas result in clear 4xx responses. The architecture expects quota checks to be performed in the API before committing expensive work or generating signed URLs. This forms part of the protection strategy both for cost control and for limiting damage in abuse scenarios.

## Rate limiting

Rate limiting is applied at least at two levels: authentication endpoints (`discover`, `login`), which are sensitive to brute-force attacks and credential stuffing, and write/mutation endpoints, which can impact shared resources and database load. Limits are defined per IP and/or per tenant+user to keep legitimate usage viable while throttling abuse. The reverse proxy and/or API layer can enforce these limits; the important property is consistency and observability of rate-limit decisions.

## Scaling considerations

The architecture scales primarily by horizontal replication of stateless components (reverse proxy, API instances, workers) behind load balancing, and by careful indexing and query design in the single multi-tenant database. CDN-based file delivery offloads bandwidth-heavy traffic from the API. Because tenant isolation is logical, adding tenants does not require provisioning new databases, but it does require attention to hotspots: heavy tenants may drive index tuning, partitioning strategies, or sharding in future evolutions.

## Disaster recovery thinking

Disaster recovery focuses on restoring core data and making the system available again in a controlled sequence. The database is treated as the system of record and should be backed up and tested for restore regularly. Storage for files must support durability and region-aware replication as needed. Configuration and infrastructure are expressed as code or documented scripts so that environments can be recreated. Observability data (logs, metrics) is preserved long enough to reconstruct incident timelines. The design assumes that in a serious incident, operators should be able to bring the system back with known steps rather than ad-hoc fixes.

