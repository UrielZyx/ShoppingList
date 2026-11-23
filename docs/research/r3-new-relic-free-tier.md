# R3: Does New Relic's free tier fit OTel traces, metrics and logs from one small app?

Researched: 2026-11-23. All pages below were read in full via WebFetch this session (first-party newrelic.com / docs.newrelic.com). None of the doc pages show a publication date; newrelic.com/pricing shows "(c)2008-26" and 2026-02 image paths, so it appears current (inference).

## Conclusion
Yes, it plausibly fits. A small app's traces, metrics and logs should sit well under 100 GB/month, OTLP is supported natively, and usage is queryable by NRQL for a FinOps dashboard. Caveats below: hard stop at the cap, short trace retention, and a few items still unconfirmed.

## Verified facts

### Pricing and plan (https://newrelic.com/pricing)
- 100 GB free data ingest per month.
- One free (full platform) user; unlimited free basic users.
- Default retention "at least 8 days".
- Basic users get dashboards, queries and alerts; no numeric limit stated for either.
- 85% usage: email plus in-platform alert. Over 100 GB: "data ingest stops and platform access ends until you upgrade or the first day of the next month" (hard stop, not silent overage billing).
- Log obfuscation (credit-card/SSN patterns) on by default. 500 synthetic checks (FAQ).
- "No time limit" (from earlier search titles) is not stated on the page text I read; keep as search-title-only.

### Billing docs (https://docs.newrelic.com/docs/accounts/accounts-billing/new-relic-one-pricing-users/pricing-billing/ and https://docs.newrelic.com/docs/accounts/accounts-billing/new-relic-one-pricing-billing/new-relic-one-pricing-billing/)
- "You have access to 100 GB Ingested per month for free"; GB rounded up to whole GB.
- User types: Basic (not charged; no curated UIs such as APM UI), Core, Full platform (Core and Full are billable). No free-plan user cap stated on these pages (the pricing page says one free full user).
- Basic users: "quickstart dashboards for up to 7 days" and "some basic alerting features" (the second page's wording). Custom dashboards/queries are listed as possible for Basic on the first page.
- EU data region is listed as a billable add-on ("EU Data Center") on paid plans. Free-tier region choice is not described there.
- Behaviour at the limit is not described on these pages (only on newrelic.com/pricing, above).

### OTLP (https://docs.newrelic.com/docs/opentelemetry/best-practices/opentelemetry-otlp/)
- US: `https://otlp.nr-data.net`; EU: `https://otlp.eu01.nr-data.net`. Ports 443 (gRPC and HTTP), 4317 (gRPC), 4318 (HTTP).
- Auth: `api-key` header carrying the license key (`OTEL_EXPORTER_OTLP_HEADERS=api-key=<LICENSE_KEY>`); missing it gives authentication errors.
- Recommends HTTP/protobuf over the public internet (`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`). Supports gzip and zstd (zstd recommended).
- Limits: payload < 1 MB; string values 4095 chars; attribute keys 255; 64 resource / 8 scope / 128 record attributes; arrays max 64 entries. Over-limit attributes are pruned/truncated and an NrIntegrationError is created. Over rate limit: requests rejected.
- The page does not list supported signals in one place; it links to separate traces, metrics and logs mapping guides. Signal support is therefore still not directly confirmed (see below).

### Retention (https://docs.newrelic.com/docs/data-apis/manage-data/manage-data-retention/)
Defaults, "Original Data" option (days): dimensional metric raw 30; aggregated metric rollups 395; spans/distributed traces 8; logs 30; custom events 30; APM events 8. "Data Plus" extends (spans 98, logs 120). Metric retention is not editable. The page does not mention the free plan, so whether free accounts get these exact values is unconfirmed (pricing page says "at least 8 days").

### Platform limits (https://docs.newrelic.com/docs/data-apis/manage-data/view-system-limits/)
- 55 million records/account/minute; 1 MB payload; attribute value 4096 chars (Log API 4094); 254 attributes per data type. At the per-minute limit New Relic returns HTTP 429 for the rest of the minute. No free-plan-specific numbers.

### Programmatic usage for a FinOps dashboard (https://docs.newrelic.com/docs/accounts/accounts-billing/new-relic-one-pricing-billing/usage-queries-alerts/)
- NRQL events `NrConsumption` (hourly) and `NrMTDConsumption` (month-to-date approximation).
- Daily ingest: `FROM NrConsumption SELECT sum(GigabytesIngested) WHERE productLine = 'DataPlatform' SINCE 3 months ago TIMESERIES 1 day` (add `FACET usageMetric` to split by source).
- Month to date: `FROM NrMTDConsumption SELECT latest(GigabytesIngested) WHERE productLine = 'DataPlatform'`.
- Alert on those same queries (event timer aggregation recommended). Results are approximate vs invoices; MTD data lags ~3 hours (use `SINCE 4 hours ago`).
- The page does not document NerdGraph access or plan requirements.

### NerdGraph (https://docs.newrelic.com/docs/apis/nerdgraph/get-started/introduction-new-relic-nerdgraph/)
- Auth with a user API key in an `API-Key` header. Endpoints: US `https://api.newrelic.com/graphql`, EU `https://api.eu.newrelic.com/graphql`. NRQL can be run through NerdGraph. Page says to sign up for a free account to start; "user type and roles can affect permissions". Rate limits are on separate pages, not read.

## Key risks
1. Hard stop at 100 GB: ingest and platform access stop until month end (85% warning only by email/in-app). Set an alert on `NrMTDConsumption` and sample traces/logs.
2. Trace retention is short (8 days by default; a "Data Plus" 98-day option is paid).
3. Only one full-platform user; Basic users lack curated UIs (APM UI).
4. Region must match endpoint (US vs EU); free-tier region choice process unconfirmed.
5. Payload (1 MB) and attribute limits can truncate OTel data; use batching and zstd/gzip.

## Still UNVERIFIED / partial
- Explicit statement that OTLP traces, metrics and logs are all supported (linked mapping pages not read; very likely, but unconfirmed).
- Free-plan-specific alert and dashboard limits (no numbers on any page read; alerts overview page does not state free availability).
- Whether free accounts get full NerdGraph/user API keys and NRQL access to NrConsumption (not stated; NerdGraph and NRQL rate-limit pages not read).
- Free-plan retention being the same as the "Original Data" defaults.
- How region (US/EU) is chosen at signup on the free tier.
- "No time limit" wording and any 2026 plan changes (no dated changelog read).
- Not fetched: https://docs.newrelic.com/docs/data-apis/manage-data/manage-data-coming-new-relic/ and the NerdGraph/NRQL limits pages.
