# R3: Does New Relic's free tier fit OTel traces, metrics and logs from one small app?

Date: 2026-11-23. STATUS: INCOMPLETE, mostly UNVERIFIED. Page fetching was blocked.

## What went wrong
WebFetch was refused for all three New Relic URLs below (permission request to the user timed out, "PROVENANCE_REQUIRED"). I did not retry or route around it (no curl). Only WebSearch result titles were available, so no page body was read.

Blocked URLs (read these to finish the ticket):
- https://newrelic.com/pricing
- https://docs.newrelic.com/docs/opentelemetry/best-practices/opentelemetry-otlp/
- https://docs.newrelic.com/docs/accounts/accounts-billing/new-relic-one-pricing-users/pricing-billing/

## Findings by confidence

Only from search-result titles (first-party newrelic.com pages, not read, dates unknown):
- Free tier is advertised as 100 GB monthly data ingest, one full user, no time limit.
  - https://newrelic.com/info/pricing ("Free Tier, 100GB Monthly Data Ingest")
  - https://newrelic.com/info/blog/solution-recommendation/inside-new-relic-s-free-tier-100-gb-of-data-one-full-user-and-no-time-limit
- A first-party OTLP endpoint doc page exists: https://docs.newrelic.com/docs/opentelemetry/best-practices/opentelemetry-otlp/
- A network-endpoints doc page exists (region/endpoint list): https://docs.newrelic.com/docs/new-relic-solutions/get-started/networks/

Unverified recall from memory (do NOT rely on without checking):
- OTLP endpoints: US `https://otlp.nr-data.net` (4318 HTTP, 4317 gRPC), EU `https://otlp.eu01.nr-data.net`; auth via `api-key` header with an ingest license key. Region is chosen at account creation (US or EU data center) and the endpoint must match.
- Retention on free is roughly 8 days for most data; basic users unlimited, free full user count 1; alerting and dashboards included on free (may have limits).
- Programmatic usage: likely NRQL on `NrConsumption` / `NrMTDConsumption` via NerdGraph or the Query API, which would allow a FinOps dashboard. Unverified; also check whether free accounts can use NerdGraph/API keys.

## Not established
Monthly limit as of today, user limits, retention, OTLP signal support, free alerting/dashboard limits, API access to usage, and any recent (2026) plan changes. Each needs a read of the pages above.

## Preliminary conclusion
Plausibly fits (small app well under 100 GB/month), but this is not confirmed from page content. Main risk: unverified limits and region mismatch.
