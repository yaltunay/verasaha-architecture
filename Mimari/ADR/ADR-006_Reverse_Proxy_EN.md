# ADR-006 Reverse Proxy Deployment

*This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.*

## Status

Accepted

## Context

The platform exposes multiple hosts: api.verasaha.com (API), cdn.verasaha.com (file serving), app.verasaha.com (admin/panel), and tenant subdomains for UI. We need TLS termination, a single point to manage certificates, and routing of external requests to the appropriate internal service. Internal services should not be directly exposed to the internet; the reverse proxy should be the only public entry point for HTTP/HTTPS traffic.

## Decision Drivers

1. **TLS and certificates:** Centralised termination at the proxy; single place for Let's Encrypt or cert management; backends stay off public internet.
2. **Routing and security:** Host-based routing to landing, API, CDN, app, tenant UIs; X-Forwarded-* for correct client IP and scheme (rate limiting, audit).
3. **Operational simplicity:** One edge to harden and monitor; internal services listen on localhost or private network only.

## Decision

We use a **reverse proxy layer** in front of all HTTP-serving components:

- **Reverse proxy layer:** A reverse proxy (e.g. Nginx) is the single public entry point for HTTPS. It accepts client connections, terminates TLS, and forwards requests to the appropriate backend (API, CDN, or app host) based on configuration. Backends can listen on localhost or a private network; they do not need to handle TLS or be directly reachable from the internet.

- **TLS termination:** TLS (HTTPS) is terminated at the reverse proxy. Certificates (e.g. Let’s Encrypt) are managed at the proxy. Backends receive plain HTTP (typically on a trusted network or loopback). This centralises certificate renewal and reduces configuration and attack surface on each backend.

- **Internal service exposure:** Internal services (API, CDN validation, app server) are not exposed on public ports. They are bound to localhost or a private IP and are reached only by the reverse proxy. This limits exposure and allows firewall rules to restrict direct access to internal hosts.

- **Host-based routing:** The proxy routes by host (and optionally path). For example, api.verasaha.com → API backend, cdn.verasaha.com → CDN or file-validation service, app.verasaha.com → admin app, and {tenant}.verasaha.com → tenant UI. This keeps a clear mapping between public hostnames and backend services and supports CORS and security policy defined per host.

## Alternatives Considered

- **TLS on each backend:** Each service would need its own certificate and renewal; configuration and operational overhead increase. Rejected in favour of centralised termination at the proxy.

- **No reverse proxy (direct exposure of backends):** Would require each backend to be publicly reachable and to handle TLS; harder to add WAF, rate limiting, or centralised logging at the edge. Rejected.

- **Cloud load balancer only:** A cloud LB can fulfil a similar role; the decision is to use a reverse proxy (e.g. Nginx) as the conceptual and often actual implementation. Same principles apply if the proxy is replaced by a managed LB with equivalent features.

## Consequences

- **Positive:** Single place for TLS and certificates; internal services stay non-public; host-based routing is clear and maintainable.
- **Positive:** X-Forwarded-* headers allow backends to see correct client IP and protocol (rate limiting, audit, security).
- **Negative:** The proxy is a single point of failure and must be highly available; proxy configuration must be kept in sync with backend deployment.
- **Negative:** Trusted-proxy configuration on backends is required; misconfiguration can affect security (e.g. IP-based limits).

## When to Revisit

Revisit if: scale or compliance requires a managed load balancer or WAF in front of the proxy; or multi-region deployment needs region-specific edge routing.
