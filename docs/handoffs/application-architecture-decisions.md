# Personal Shopping List App --- Architecture Decisions

*Last updated: 2026-11-22*

This document captures the current architecture direction for a
personal, shared shopping-list application. It is intended to give a
fresh implementation agent enough context to begin implementation
planning without reopening settled decisions unnecessarily.

It records decisions and current recommendations, not a fully specified
design. Items explicitly marked **Open** still require discussion or
prototyping.

## 1. Product context and goals

-   Build a shopping-list application initially for personal household
    use, shared by two partners, with the possibility of sharing it with
    family or a few friends later.
-   The primary experience is mobile.
-   Avoid native app development and app-store certification for the
    initial release.
-   A desktop browser experience is also useful: users may update the
    list from a computer.
-   A CLI is an intentional first-class client, not a later add-on.
-   Keep hosting costs at or near zero initially, using free tiers where
    practical.
-   Avoid unnecessary vendor lock-in and preserve the option to move
    providers.
-   The application starts small: one backend application, one database,
    and one server instance. There is no current need for multiple
    independently deployed services or instances.
-   Coding agents are expected to implement much of the code. Favor
    established, well-documented technologies and conventions, and keep
    the codebase easy to review.

## 2. Product behavior and MVP scope

### 2.1 Shopping list behavior

-   The initial product has one persistent shopping list.
-   Unpurchased items appear in the main section.
-   Purchased items move to a separate section at the bottom, similar to
    Google Keep.
-   Marking an item as purchased does **not** delete it.
-   Purchased items remain available and can be unchecked later,
    returning them to the unpurchased section.
-   This persistent-list model is the initial behavior. Shopping-trip
    lifecycle/history is deferred.

### 2.2 MVP capabilities

The first useful release should provide enough functionality to make
switching from Google Keep worthwhile.

MVP scope:

-   Shared list for multiple users.
-   Add, edit, and remove items.
-   Mark items purchased and unmark them.
-   Keep purchased items in a separate section at the bottom.
-   Live synchronization between connected clients.
-   Push notifications for relevant changes.
-   Comments/questions and replies associated with list items or the
    list, as appropriate to the eventual domain design.
-   Photo attachments, such as pictures of products or alternatives on
    supermarket shelves.
-   Activity history showing relevant changes.

The exact UX, notification preferences, and detailed domain model are
not yet specified.

### 2.3 Deferred product functionality

-   Explicit "Start shopping trip" / "End shopping trip" operations.
-   Recording and browsing purchases grouped by shopping trip.
-   More elaborate offline functionality.
-   Native Android/iOS applications.

Shopping trips may eventually provide a way to record what was bought on
each trip, but they are not part of the first iteration.

## 3. Infrastructure and backend decisions

The following decisions carry forward from the infrastructure
architecture discussion.

### 3.1 Compute and deployment

-   **Provider:** Oracle Cloud Infrastructure (OCI), using an Always
    Free VM if capacity is available.
-   **Runtime/deployment:** Linux VM with Docker Compose.
-   **Application shape:** one deployable backend; background work
    initially runs in the same process.
-   OCI free-tier capacity and availability must be verified when
    provisioning; Always Free capacity is not guaranteed.

### 3.2 Backend language

-   **Not decided.**
-   Candidates: **.NET / ASP.NET Core** and **Rust**.
-   .NET is a strong practical option given existing experience and its
    mature PostgreSQL, Linux/Docker, background-worker, and
    OpenTelemetry support.
-   Rust is also reasonable, especially as a learning experience.
    Resource efficiency is a bonus, not the main architectural driver.
-   Keep the architecture language-neutral. A small vertical-slice
    prototype in both languages can inform the eventual choice.
-   Do not add elaborate abstractions solely to make changing languages
    easier.

### 3.3 Database

-   **PostgreSQL hosted by an external managed provider**, rather than
    self-hosted on the OCI VM.
-   **Neon** was the leading candidate, subject to checking current
    free-tier limits and terms before committing.
-   Use PostgreSQL and standard SQL where practical to preserve
    portability.
-   Monitor managed-database storage, compute-hour, and network-transfer
    quotas.

### 3.4 Cache, messaging, and background work

-   **No Redis for now.**
-   There is one application instance and no plan for multiple
    services/instances, so Redis/Kafka would add operational and quota
    complexity without a clear need.
-   Simple in-process work can use an in-memory queue. Such queued work
    is lost if the process stops.
-   If jobs need to survive restarts, use PostgreSQL-backed job
    storage/queueing.
-   PostgreSQL `LISTEN/NOTIFY` can be a wake-up signal, but is not a
    durable queue.
-   Revisit Redis/Valkey or a dedicated event-streaming service only
    when a concrete requirement appears.

### 3.5 Observability

-   Use **OpenTelemetry** for application instrumentation.
-   **New Relic** was the preferred hosted-observability candidate,
    subject to checking current free-tier limits and features.
-   Keep logs, traces, and metrics exportable through standard
    OpenTelemetry-compatible paths where possible.

### 3.6 DNS, TLS, and edge services

-   **Cloudflare** is the current candidate for DNS and, as appropriate,
    proxy/CDN and TLS-related edge functionality.
-   Keep the application deployable without Cloudflare-specific runtime
    dependencies where practical.

## 4. Frontend architecture direction

### 4.1 Application form

-   Build a **mobile-first, responsive web application**.
-   Make it an installable **Progressive Web App (PWA)**.
-   Do not build separate native applications for the initial release.
-   The same web application should work in desktop browsers.
-   Prefer a **client-rendered SPA** rather than SSR.
-   Build static frontend assets and host them independently of the
    backend where practical.
-   Avoid requiring a Node.js server or a framework-specific frontend
    runtime in production.
-   The frontend should not depend on Cloudflare-specific runtime
    features.

Rationale: - SEO is not important for this authenticated personal
application. - SSR and a frontend server would add complexity without a
clear benefit. - Static assets are easy to host on different providers
or serve from the backend if needed. - A PWA provides an installable
mobile experience without app-store distribution.

PWA platform caveats: - Push notification support differs between
platforms. iOS Web Push is available for web apps added to the Home
Screen on supported iOS versions; do not assume identical behavior
across platforms. - Do not assume the browser can continuously execute
background work. - Offline editing is not required.

### 4.2 Frontend framework

**Current recommendation: React + TypeScript + Vite.**

This is the current direction, not a requirement to finalize every
frontend library before implementation.

Reasons: - React has a large, mature ecosystem and extensive
documentation and examples. - It is a safe choice for compatibility with
coding agents. - TypeScript provides useful types and tooling. - React
provides a relatively direct future path to React Native / Expo if
native applications become worthwhile. - Vite can build a static SPA
without introducing SSR or a production Node.js server.

Native portability caveat: - React Native is not simply React DOM
compiled to native. - Web components use HTML/DOM elements; React Native
uses native components and different styling/platform APIs. - Business
logic, API clients, types, validation, and some shared packages can be
reused, but expect the UI layer to require separate implementations. -
Do not compromise the initial web experience to maximize hypothetical
code reuse.

Vue and Svelte were considered as alternatives. Vue may feel more
familiar to developers used to HTML templates and traditional component
structures; Svelte is concise but has its own compiler/reactivity
conventions. React was preferred overall due to ecosystem maturity,
agent familiarity, and the possible React Native path.

### 4.3 Frontend decisions intentionally deferred

Do not treat the following as settled. Decide them during initial
implementation and prototyping:

-   Component library / design system (e.g. Material UI, Mantine,
    shadcn/ui, or another option).
-   Server-state management (e.g. TanStack Query or a simpler approach).
-   Routing.
-   Form handling and validation libraries.
-   Exact PWA tooling and service-worker setup.
-   Detailed component architecture and styling conventions.

Avoid introducing elaborate state-management infrastructure prematurely.
Keep local UI state local; choose a server-state solution when
implementation needs become clear.

## 5. Client/backend boundary and API

### 5.1 API style

-   Use **HTTP/JSON REST APIs**.
-   Use **OpenAPI** as the API contract.
-   The frontend and CLI are independent clients of the same application
    API.
-   Keep the API independent of the backend language.
-   Generate TypeScript API types/client from OpenAPI where practical.
    Other clients can use generated or separately maintained clients
    appropriate to their implementation language.
-   Do not implement business rules exclusively in the frontend or CLI.
    The backend owns validation, authorization, business rules,
    persistence, and activity recording.

GraphQL was considered and rejected for the initial version: - The
application has a relatively straightforward data and interaction
model. - All clients are maintained by us. - REST endpoints can provide
the projections the clients need without introducing GraphQL
schema/resolver infrastructure. - GraphQL does not eliminate the need
for a separate real-time update or push-notification mechanism. -
Reconsider only if substantial query-shaping needs or independently
evolving clients emerge.

Example endpoint shapes discussed illustratively (not a finalized API
specification):

-   `GET /lists/{id}/items`
-   `POST /lists/{id}/items`
-   `PATCH /items/{id}`
-   `POST /items/{id}/purchase`
-   `POST /items/{id}/comments`
-   `POST /items/{id}/attachments`

These examples should not constrain the eventual API design without
considering resource semantics and consistency.

### 5.2 Real-time updates

Use **Server-Sent Events (SSE)** initially for server-to-client updates
while clients are connected.

Expected flow: 1. A client sends a normal HTTP request to make a change.
2. The backend validates and persists the change. 3. The backend records
the relevant activity. 4. Connected clients receive an SSE event and
update their views.

SSE is preferred over WebSockets because the initial use case is
predominantly server-to-client updates. Client actions remain ordinary
HTTP requests.

WebSockets can be reconsidered if requirements such as typing indicators
or richer bidirectional interaction appear.

SSE is not a substitute for push notifications when the app is closed.

### 5.3 Push notifications

Use **Web Push** for notifications when the PWA is not actively
connected/open, subject to platform support and permissions.

Notifications should be sent only after the underlying database
transaction commits. Do not notify clients about changes that were not
successfully persisted.

Initially, best-effort notification delivery may be acceptable. If
reliable processing becomes necessary, consider a PostgreSQL-backed
outbox and durable job processing.

Notification types, preferences, deduplication, and delivery guarantees
remain open.

### 5.4 Attachments

-   Photo attachments are part of the MVP.
-   Start with ordinary HTTP upload, likely `multipart/form-data`.
-   If object storage is introduced later, pre-signed upload URLs are an
    option.
-   Object-storage provider and storage architecture are not yet
    decided.
-   Do not make photo upload depend on GraphQL.

## 6. CLI as a first-class client

The CLI is **in scope from day one**, not a later convenience project.

Rationale: - Designing for CLI parity early helps ensure that the API
exposes application capabilities rather than merely mirroring the web
UI. - Adding CLI support later can be significantly harder if business
logic or operations have leaked into frontend-specific code.

Commitments: - The CLI uses the same public application API as the
PWA. - All domain operations should be available through the API and
usable by the CLI. - The CLI is an independent client with its own
project, tests, build, and release artifacts. - The backend must not
rely on frontend-only business logic. - Client-specific features such as
camera integration and browser push notifications need not have direct
CLI equivalents.

Expected capability parity:

  Capability              PWA   CLI
  ----------------------- ----- -----------------
  View shopping list      Yes   Yes
  Add/edit/remove items   Yes   Yes
  Mark/unmark purchased   Yes   Yes
  View activity           Yes   Yes
  Comments/replies        Yes   Yes
  Upload photos           Yes   Yes, from files
  Live updates            SSE   SSE
  Push notifications      Yes   Not initially
  Camera integration      Yes   Not applicable

Illustrative CLI commands (not a finalized command syntax):

``` bash
shopping list
shopping add "Milk"
shopping add "Bread"
shopping purchased "Milk"
shopping comment "Bread" "Whole wheat, please"
shopping history
shopping watch
```

The `watch` command could subscribe to the SSE stream and display
changes.

The CLI implementation language is not decided.

## 7. Portability principles

-   Package the backend as a standard Linux container.
-   Use PostgreSQL rather than a provider-specific database where
    possible.
-   Use HTTP/JSON APIs and OpenAPI for the frontend/backend boundary.
-   Use environment variables or mounted configuration for deployment
    settings and secrets.
-   Use OpenTelemetry for instrumentation.
-   Keep frontend output as static assets where practical.
-   Avoid provider-specific queues, databases, or application runtimes
    unless a concrete benefit justifies them.
-   Keep the PWA independent of provider-specific runtime features.
-   Keep the CLI and PWA as independent API clients.

## 8. Explicitly open decisions

These are not settled and should not be guessed by an implementation
agent:

-   Backend language: .NET or Rust.
-   CLI implementation language.
-   Authentication and user/account model.
-   Exact authorization/sharing model.
-   Exact API resource design and endpoint contract.
-   Component library and frontend supporting libraries.
-   PWA tooling and service-worker details.
-   Notification types, preferences, and delivery guarantees.
-   Attachment storage provider and implementation.
-   Backup and restore strategy.
-   CI/CD and container registry.
-   Exact OCI VM shape/architecture and deployment setup.
-   Exact managed PostgreSQL provider, after checking current quotas,
    regions, reliability, and pricing.
-   Whether durable background-job infrastructure is needed.
-   Shopping-trip lifecycle and purchase history.

## 9. Implementation guidance for a fresh agent

When starting implementation:

1.  Treat the decisions in this document as the current baseline. Do not
    silently replace them with preferred alternatives.
2.  Keep backend-language-specific implementation decisions separate
    from the language-neutral API and domain design.
3.  Design the API so both the PWA and CLI can perform the same domain
    operations.
4.  Keep business logic in the backend.
5.  Prototype deferred frontend libraries when there is a concrete
    implementation need; do not prematurely settle them.
6.  Implement the persistent-list behavior first: purchased items remain
    in the list and move to a separate bottom section.
7.  Treat real-time updates and push notifications as distinct
    capabilities.
8.  Do not implement offline editing in the initial version.
9.  Do not introduce Redis, Kafka, GraphQL, SSR, or multiple backend
    services without a concrete requirement.
10. Check current provider pricing, free-tier limits, platform support,
    and documentation before relying on them.

## 10. Important caveats

-   Free-tier offerings, quotas, regions, and eligibility rules change.
    Recheck provider documentation before provisioning or depending on a
    quota.
-   A free-tier design is not automatically highly available. The
    initial setup is intentionally simple and needs a backup/restore
    plan for data that matters.
-   An in-memory queue is suitable only for work that can be retried or
    safely lost on process restart.
-   PWA capabilities, especially push notifications, differ across
    browsers and operating systems. Verify behavior on the actual target
    devices.
-   The architecture is intentionally modest. Prefer the simplest
    implementation that preserves the decisions above.
