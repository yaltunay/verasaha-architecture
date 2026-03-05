# ADR-007 Observability Stack

*This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.*

## Status

Accepted

## Context

The platform must support production operations: diagnosing failures, understanding performance, and detecting abuse or anomalies. We need a consistent approach to logging, metrics, and tracing that works across the API, workers, and other components, without exposing sensitive data or creating unsustainable cardinality (e.g. per-tenant metrics that explode in size).

## Decision Drivers

1. **Incident response and audit:** CorrelationId and structured logs enable request-level tracing; AuditLog supports compliance and KVKK-relevant audit; redaction protects privacy.
2. **Operational visibility:** Metrics and optional tracing support capacity planning and anomaly detection; /metrics protected to avoid information leakage.
3. **Cardinality control:** No high-cardinality tenant labels on metrics; optional tenant_id in spans to keep observability stack manageable.

## Decision

We adopt a **centralised observability stack** with the following elements:

- **Structured logging:** Application logs are structured (e.g. Serilog with JSON or key-value output). Each log entry can include correlation_id, tenant_id, user_id (when available), and other context. Logs are suitable for aggregation and querying in a central log store. **Sensitive data** (passwords, tokens, full PII) is not written to logs; redaction or masking is applied so that operational visibility does not compromise security or privacy.

- **Metrics:** Key operational metrics are exposed in a standard format (e.g. Prometheus). Examples include request counts, error rates, and outbox queue depth (e.g. notifications.outbox.pending). The metrics endpoint (e.g. /metrics) is protected (e.g. Admin JWT or internal network only) so it is not publicly scraped. **Cardinality** is controlled: high-cardinality labels such as tenant_id are avoided on metrics that would create too many time series, to keep the metrics stack manageable.

- **Tracing:** Distributed tracing (e.g. OpenTelemetry) is used to follow a request across services or components. Trace IDs and correlation_id are aligned where possible. Spans can carry correlation_id, user_id, and optionally tenant_id (or omit tenant_id to limit cardinality). In development, a console exporter may be used; in production, traces can be exported to an OTLP endpoint or similar for centralised analysis.

- **Centralised observability:** Logs, metrics, and traces are designed to be shipped to or scraped by centralised systems (e.g. log aggregation, Prometheus, trace backend). This supports incident response, capacity planning, and auditing. The exact choice of backends (e.g. Grafana, Jaeger, Elasticsearch) is an operational concern; the architecture commits to producing structured, non-sensitive, and cardinality-conscious signals.

## Alternatives Considered

- **Unstructured logs only:** Harder to query and correlate; rejected in favour of structured logging.

- **Tenant_id on every metric:** Would cause high cardinality and cost in metric storage and querying. Rejected; tenant is included only where necessary and where cardinality is bounded.

- **No tracing:** Would make cross-service or cross-component diagnosis harder. Rejected in favour of optional but supported tracing.

- **Logging sensitive data for debugging:** Security and compliance risk. Rejected; redaction is mandatory.

## Consequences

- **Positive:** Single approach to logs, metrics, and traces; correlation_id ties logs and traces for incident investigation; redaction and cardinality controls protect security and KVKK-aligned logging.
- **Positive:** Metrics support alerting and capacity planning; AuditLog and CorrelationId support audit and DSR-related traceability.
- **Negative:** Operational dependency on log/metric/trace backends; instrumentation must be applied consistently.
- **Negative:** Tenant-level debugging may require log queries rather than high-cardinality metrics; dev and production exporters may differ (console vs OTLP).

## When to Revisit

Revisit if: compliance or audit requires bounded tenant-level metrics (e.g. per-tenant dashboards with cardinality limits); or centralised logging/tracing backend choice changes (e.g. vendor lock-in or cost).
