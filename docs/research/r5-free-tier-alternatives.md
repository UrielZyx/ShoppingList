# R5: Free-tier alternatives to Oracle, Neon, New Relic, Cloudflare

Researched 2026-11-23 (GitHub issue #6). All figures below were read from the vendor's own page on that date via WebFetch. Where a page showed a date it is given; otherwise "no date shown". Items marked UNVERIFIED were not confirmed from a primary source.

Requirements: a few users, long-lived SSE, standard Linux containers, no vendor lock-in, web app plus CLI.

## Verified facts by vendor

### Oracle Always Free (compute)
Source: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm (last modified 2026-06-12)
- Arm A1 Flex: 1,500 OCPU-hours and 9,000 GB-hours a month, equal to 2 OCPU / 12 GB total, in one or two instances.
- AMD micro: up to two instances, each 1/8 OCPU, 1 GB RAM, 50 Mbps.
- Storage: 200 GB boot plus block volumes, 5 volume backups. Egress: 10 TB a month.
- Instances must be in the home region. "Out of host capacity" errors are possible (temporary shortage).
- Idle reclamation: Oracle may reclaim an instance if over 7 days the 95th-percentile CPU is under 20%, network is under 20%, and (A1 only) memory is under 20%. A lightly used personal app can meet this.
- Outbound TCP port 25 is blocked by default.
- No explicit account-termination policy on this page. Free Trial expiry moves the account to Always Free-only, and Object Storage over 20 GB is deleted.
- The page does not list managed PostgreSQL in Always Free (it lists Autonomous Database, NoSQL, MySQL HeatWave).

### Neon (managed Postgres)
Source: https://neon.com/pricing (page updated 2026-11-04)
- Free: $0, no card, permanent. 100 projects, 100 CU-hours per project a month, up to 2 CU autoscale.
- Scale to zero after 5 minutes idle, cannot be disabled.
- Storage 1 GB per project and 20 GB total (writes blocked when exceeded). Egress 5 GB per project a month; compute suspends if exhausted.
- 6-hour history window, 1 manual snapshot, 1 day monitoring retention. Pooling up to 10,000 pooled connections per compute (all plans). No free-specific connection cap listed.
- Paid: Launch $0.106/CU-hour and $0.35/GB-month; no monthly minimum.

### New Relic (observability)
Sources: https://newrelic.com/pricing ; https://docs.newrelic.com/docs/opentelemetry/best-practices/opentelemetry-otlp/ (no date shown on either)
- Free: 100 GB ingest a month, perpetual. Exceeding it stops ingest and removes platform access until next month or upgrade. Retention at least 8 days. One free full-platform user, unlimited basic users.
- Paid ingest: $0.40/GB (Original), $0.60/GB (Data Plus).
- Native OTLP ingest, gRPC and HTTP. Endpoint https://otlp.nr-data.net (US), auth via `api-key` header. Docs recommend OTLP/HTTP protobuf over the public internet. Limits: 1 MB payload, attribute count and length caps, delta temporality preferred.
- The pricing page itself does not mention OTel in its free-tier details, but the OTLP docs page confirms native support. Not stated whether the OTLP docs apply to the free tier specifically (UNVERIFIED, likely yes).

### Grafana Cloud
Sources: https://grafana.com/pricing/ ; https://grafana.com/docs/grafana-cloud/send-data/otlp/send-data-otlp/ (no dates shown)
- Free: 10k active metric series, 50 GB logs, 50 GB traces, 50 GB profiles, all with 14-day retention; 3 active users.
- OTLP over HTTP to a regional gateway (`https://otlp-gateway-<region>.grafana.net/otlp`), Basic-auth token. gRPC support not confirmed on the page (only the first 100k of 366k characters were read).
- Pricing page does not say what happens on overage (UNVERIFIED).

### Honeycomb
Source: https://www.honeycomb.io/pricing (no date shown)
- Free: up to 20M events a month, 100M metric data points, 2 triggers, OpenTelemetry supported. No SLOs, Service Map or SSO.
- Retention and user limits for Free are not stated on the page (UNVERIFIED). Described as "best for testing and individual projects".

### Fly.io
Source: https://fly.io/docs/about/pricing/ (no date shown)
- No general free compute allowance on the page. Credit card required for orgs (except Linked Organizations). Details of a free trial live on a separate page that was not read (UNVERIFIED).
- shared-cpu-1x 256 MB: $0.0027/hour, about $1.94 a month. Volumes $0.15/GB-month. Egress $0.02/GB in NA/EU. First 10 GB snapshots and first 10 certificates free.
- So Fly is not free; it is low-cost.

### Supabase
Sources: https://supabase.com/pricing ; https://supabase.com/docs/guides/platform/backups (no dates shown)
- Free: 500 MB database per project, 2 active projects, 5 GB egress (+5 GB cached). Free projects are paused after 1 week of inactivity. No automatic backups on Free; docs recommend regular exports and off-site backups.
- Pro from $25 a month.
- Standard Postgres, so pg_dump or `supabase db dump` works for portability.

### Cloudflare (DNS and edge)
Sources: https://developers.cloudflare.com/dns/ ; https://developers.cloudflare.com/fundamentals/reference/connection-limits/ (no dates shown)
- DNS is "available on all plans". Free-plan record limits were not found; the FAQ page failed to load twice (UNVERIFIED).
- Proxy idle timeout 900 s (HTTP 520, not configurable). Proxy Read Timeout 125 s (524), configurable only on Enterprise. The page says nothing about SSE or streaming.
- Implication (inference, not stated by Cloudflare): an SSE stream that sends nothing for over 125 s risks a 524 through the proxy, so the app should send heartbeat comments well under that, or use DNS-only (grey cloud). Needs a real test.

## Comparison and verdicts

### Compute: Oracle Always Free vs Fly.io vs small VPS
- Oracle gives by far the most free resources (2 OCPU / 12 GB / 200 GB, 10 TB egress) and runs plain Linux containers, so portability is excellent. Fly's Machines are also container-based but it is not free.
- Oracle risks: idle reclamation for a low-traffic app, capacity errors at creation, and unclear termination terms. These are real for a few-user app. Mitigations: keep the box above the thresholds (not recommended as a design goal), or keep the Compose stack portable so it can move.
- Verdict: Oracle still wins on free capacity and lock-in (it is just a VM). Fly beats it on reliability predictability only if you accept about $2 a month and up. A paid VPS (price not researched, UNVERIFIED) is the fallback. Keep Oracle, with documented migration steps.

### Postgres: Neon vs Supabase vs self-hosted in Compose
- Neon Free has 20 GB total across projects but 1 GB per project, scale-to-zero with 5 minute suspend (cold start latency, and it may interact with a long-lived SSE app holding connections; cold-start time not checked, UNVERIFIED), 6-hour history only.
- Supabase Free is worse for this use: 500 MB, pauses after 1 week inactive, no backups.
- Self-hosted Postgres in the Compose stack: no caps beyond the 200 GB disk, no cold starts, plain pg_dump. Cost is operations: you must run off-site backups (for example pg_dump to an object store) and handle upgrades. Given Oracle reclaim and termination risk, off-site backups are essential.
- Verdict: Neon beats Supabase on every verified free metric. Self-hosted beats Neon on portability, cold starts and size, and loses on ops burden. For minimal lock-in, self-hosted plus off-site backups is the strongest. Neon is a fine pick if you want zero ops and accept cold starts, since it is standard Postgres and pg_dump works (pg_dump compatibility is general knowledge, not read from a Neon page, UNVERIFIED).

### Observability: New Relic vs Grafana Cloud vs Honeycomb
- All three accept OTLP, so with an OTel Collector in the Compose stack the backend is swappable and lock-in is low.
- New Relic: 100 GB a month is the largest verified allowance, with hard stop at the cap, 8-day retention, one full user.
- Grafana Cloud: 50 GB traces and logs, 14-day retention, 3 users, and a 10k-series metrics cap that is the likely constraint. OTLP over HTTP works.
- Honeycomb: 20M events, OTel supported, but retention and users unverified, and framed as for testing.
- Verdict: New Relic beats on raw volume. Grafana Cloud beats on retention (14 vs 8 days), user count and being built on open-source components, which lowers lock-in in query and dashboards (open-source claim is general knowledge, UNVERIFIED here). Neither is clearly dominant; for this load both fit, and either is acceptable. Slight preference to Grafana Cloud for lock-in, New Relic for volume. Self-hosted Grafana/Tempo/Loki was not researched (UNVERIFIED) and would use the Oracle VM's capacity.

### DNS/edge: Cloudflare
- No alternative was researched (UNVERIFIED). DNS is on all plans. Proxy read timeout of 125 s is the thing to design around for SSE (heartbeats) or avoid by DNS-only mode.
- Verdict: keep Cloudflare for DNS. Treat the proxy as optional for SSE routes until tested.

## Unverified summary
- Cloudflare free DNS record limits (FAQ page failed twice).
- Cloudflare SSE behaviour (the docs page is silent).
- Fly.io free trial terms (separate page not read).
- Grafana gRPC OTLP and overage behaviour; Honeycomb Free retention and users.
- Neon cold-start time, Neon pg_dump support statement.
- Any alternatives not fetched: other DNS providers, other VPS/free hosts, self-hosted observability stacks.
