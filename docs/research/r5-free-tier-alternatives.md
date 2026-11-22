# R5: Free-tier alternatives (Oracle, Neon, New Relic, Cloudflare) - INCOMPLETE

Date: 2026-11-23. Status: BLOCKED. No primary source could be read.

## What happened
- WebFetch on all 8 pages (Oracle Always Free docs, Neon pricing, New Relic pricing, Grafana pricing,
  Honeycomb pricing, Fly.io pricing, Supabase pricing, Cloudflare DNS docs) returned PROVENANCE_REQUIRED
  (permission request not answered in time; told not to retry).
- curl via the agent proxy to neon.com returned HTTP 403 (host not in the session's allowed domains).
- WebSearch returned only result titles/URLs, no page content.
So no limit, term or price below has been verified. I did not fill the gaps from memory, because
the task asks for current (2026-11-23) primary-source figures and memory figures for free tiers go stale quickly.

## URLs to read (primary sources) once access is granted
Oracle Always Free: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
  (check: A1 OCPU/RAM, idle-reclaim policy, egress, account termination terms)
Neon: https://neon.com/pricing , https://neon.com/docs/introduction/plans (search also surfaced
  https://neon.com/faqs/free-plan-limits-and-quotas) (check: storage, compute hours, scale-to-zero, connection limits)
New Relic: https://newrelic.com/pricing , OTLP docs on docs.newrelic.com (check: free ingest GB, retention, users)
Grafana Cloud: https://grafana.com/pricing/ (check: traces/logs/metrics free limits, retention, OTLP endpoint)
Honeycomb: https://www.honeycomb.io/pricing (check: free events/month, retention, OTel support)
Fly.io: https://fly.io/docs/about/pricing/ (check: whether any free allowance still exists for new orgs; long-lived connections)
Supabase: https://supabase.com/pricing (check: pause-after-inactivity, DB size, portability)
Cloudflare: https://developers.cloudflare.com/dns/ , https://developers.cloudflare.com/fundamentals/reference/connection-limits/
  (check: free-plan proxy idle/timeouts for SSE, record limits)

## Candidate comparison to complete (unverified hypotheses, not findings)
- Compute: Oracle Always Free vs Fly.io vs other small VPS. Key question: idle reclamation and account-termination risk.
- Postgres: Neon vs Supabase vs self-hosted Postgres in the same Compose stack (best portability, plain pg_dump;
  costs ops/backup work). Key question: scale-to-zero cold starts and storage cap.
- Observability: New Relic vs Grafana Cloud vs Honeycomb vs self-hosted OTel Collector + Grafana/Tempo/Loki.
  All accept OTLP; lock-in is minimal if the app emits OTLP to a Collector and the backend is swappable.
- Edge/DNS: Cloudflare free. Key question: proxied-connection behaviour for long-lived SSE (needs verification).
