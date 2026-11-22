# R2: Does Neon's free tier fit a small Postgres DB for a few users?

Date of research attempt: 2026-11-23. Status: **INCOMPLETE - no facts verified.**

## What happened
Every WebFetch to Neon primary sources was refused by the permission layer
("user did not answer an earlier permission request in time"). Per the tool
instruction I did not retry. No page content was read, so no limit below is
confirmed from a source. WebSearch (neon.com only) returned titles and URLs but no page text.

Blocked URLs (read these first when access is granted):
- https://neon.com/docs/introduction/plans (plan limits table)
- https://neon.com/docs/introduction/scale-to-zero
- https://neon.com/docs/guides/scale-to-zero-guide
- https://neon.com/docs/connect/connection-pooling
- https://neon.com/docs/introduction/regions
- https://neon.com/docs/introduction/branch-restore
- https://neon.com/terms-of-service
- https://neon.com/pricing
- https://neon.com/faqs/free-plan-limits-and-quotas

## Per-question status
| Topic | Verified? | Note |
|---|---|---|
| Storage limit | No | Check plans page |
| Compute hours / transfer (egress) | No | Check plans + free-plan FAQ |
| Scale-to-zero, cold start | No | Check scale-to-zero docs; confirm whether idle timeout is configurable on Free |
| Connection limits, pooling | No | Check pooling page (PgBouncer, pooled hostname) |
| Region nearest Israel | No | Check regions page; likely candidates to look for: Frankfurt (eu-central-1), London; confirm whether any Middle East region exists |
| Backup / PITR on Free | No | Check branch-restore page for the free history window |
| Terms that could end free allowance | No | Check ToS and pricing for inactivity, change-of-plan and fair-use clauses |

## Design implication to verify (reasoning, not sourced)
If Free computes suspend after an idle period, a long-lived SSE connection that
is held open to the DB (or a keep-alive query loop) would either keep compute
awake and burn compute-hours, or be dropped on suspend. Safer design to
confirm against the docs: app server holds SSE to browsers, and opens short
DB queries via the pooled endpoint with retry on first-connect cold start;
do not hold a DB connection per SSE client; avoid LISTEN/NOTIFY across
idle periods (not supported through PgBouncer transaction pooling - verify).

## Conclusion
Cannot be given from evidence. Re-run this ticket with web access to the URLs above.
