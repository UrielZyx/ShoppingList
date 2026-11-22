# Personal Web App --- Architecture Decisions

*Last updated: 2026-11-22*

## Context and goals

-   Build a small web application for personal use, potentially shared
    with family and a few friends.
-   Keep hosting costs at or near zero initially, using free tiers where
    practical.
-   Avoid unnecessary vendor lock-in and keep the option to move
    providers.
-   A multi-cloud setup is acceptable.
-   The application is expected to start small: one backend application,
    one database, and one server instance. There is no current need for
    multiple independently deployed services or instances.

## Current architecture direction

### Compute and deployment

-   **Provider:** Oracle Cloud Infrastructure (OCI), using an Always
    Free VM if capacity is available.
-   **Runtime/deployment:** Linux VM with Docker Compose.
-   **Application shape:** a single deployable backend, with background
    work running in the same process initially.
-   OCI's free-tier capacity and availability should be verified when
    provisioning; do not treat Always Free capacity as guaranteed.

### Backend language

-   **Not decided.**
-   Candidates: **.NET / ASP.NET Core** and **Rust**.
-   .NET is a strong practical option given existing experience and its
    mature PostgreSQL, Linux/Docker, background-worker, and
    OpenTelemetry support.
-   Rust is also a reasonable choice, especially as a learning
    experience. Resource efficiency is a bonus, not the main
    architectural driver.
-   Keep the architecture language-neutral. A small vertical-slice
    prototype in both languages can inform the eventual choice.
-   Do not add elaborate abstractions solely to make changing languages
    easier.

### Database

-   **PostgreSQL hosted by an external managed provider** (rather than
    self-hosting it on the OCI VM).
-   **Neon** was the leading candidate discussed, subject to checking
    its current free-tier limits and terms before committing.
-   Use PostgreSQL and standard SQL where practical to preserve
    portability.
-   Keep an eye on managed-database storage, compute-hour, and
    network-transfer quotas.

### Cache, messaging, and background work

-   **No Redis for now.**
-   There is currently one application instance and no plan for multiple
    services/instances, so Redis/Kafka would add operational and quota
    complexity without a clear need.
-   For simple in-process work, use an in-memory queue (for example,
    `.NET`'s `System.Threading.Channels`, or an equivalent in the chosen
    language). Such queued work is lost if the process stops.
-   If jobs need to survive restarts, use PostgreSQL-backed job
    storage/queueing.
-   PostgreSQL `LISTEN/NOTIFY` can be useful as a wake-up signal, but
    notifications are not a durable queue; disconnected listeners can
    miss them.
-   Revisit Redis/Valkey or a dedicated event-streaming service only
    when a concrete requirement appears.

### Observability

-   Use **OpenTelemetry** for application instrumentation, so telemetry
    is not tightly coupled to a particular vendor.
-   **New Relic** was the preferred hosted-observability candidate
    discussed, partly to avoid defaulting to Grafana, which is already
    familiar from work.
-   Verify the current free-tier limits and included features before
    relying on them.
-   Keep logs, traces, and metrics exportable through standard
    OpenTelemetry-compatible paths where possible.

### DNS, TLS, and edge services

-   **Cloudflare** is the current candidate for DNS and, as appropriate,
    proxy/CDN and TLS-related edge functionality.
-   Keep the application deployable without Cloudflare-specific runtime
    dependencies where practical.

## Portability principles

-   Package the backend as a standard Linux container.
-   Use PostgreSQL rather than a provider-specific database where
    possible.
-   Use HTTP/JSON APIs and OpenAPI for the frontend/backend boundary.
-   Use environment variables or mounted configuration for deployment
    settings and secrets.
-   Use OpenTelemetry for instrumentation.
-   Avoid provider-specific queues, databases, or application runtimes
    unless a concrete benefit justifies them.

## Explicitly deferred decisions

These are intentionally not settled yet:

-   Backend language: .NET or Rust.
-   Frontend framework and rendering model (the next topic to design).
-   Exact OCI VM shape/architecture and deployment setup.
-   Exact managed PostgreSQL provider, after checking current quotas,
    regions, reliability, and pricing.
-   Authentication and user/account model.
-   Backup and restore strategy.
-   CI/CD and container registry.
-   Email, file/object storage, and other supporting services, if the
    app needs them.
-   Whether any durable background-job framework is needed.

## Important caveats

-   Free-tier offerings, quotas, regions, and eligibility rules change.
    Recheck provider documentation before provisioning or depending on a
    quota.
-   A free-tier design is not automatically highly available. The
    initial setup is intentionally simple and should have a
    backup/restore plan for any data that matters.
-   An in-memory queue is suitable only for work that can be retried or
    safely lost on process restart.
-   Avoid introducing Redis, Kafka, or multiple services preemptively.

## Next discussion

Design the frontend independently of the backend-language decision. In
particular, consider the app's interaction model, whether it should be a
PWA, hosting/deployment options, and how the frontend will consume the
backend API.
