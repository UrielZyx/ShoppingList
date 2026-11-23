# R2: Does Neon's free tier fit a small Postgres DB for a few users, with long-lived SSE?

Researched: 2026-11-23. Status: **COMPLETE for the checklist, with gaps labelled below.**
All facts came from WebFetch of Neon primary sources (summaries via the fetch tool's
extraction model, not raw page text; re-check the live pages before committing).

## Conclusion
Yes, the Free plan fits a few-user shopping list on size and compute. Caveats are the
forced 5-minute scale-to-zero, pooler limits (no LISTEN/NOTIFY), a 6-hour restore window,
and no region in the Middle East. Design so the SSE layer does not depend on a held DB connection.

## Verified facts (source, page date)
| Topic | Fact | Source |
|---|---|---|
| Price | $0/month, permanent, no card | https://neon.com/pricing (updated 2026-11-04) |
| Storage | 1 GB Postgres per project, 20 GB across all Free projects; over limit, writes fail, reads work, no overage billing, no data deleted | https://neon.com/docs/introduction/plans (updated 2026-11-18); https://neon.com/faqs/free-plan-limits-and-quotas |
| Compute | 100 CU-hours/project/month (about 400 h at 0.25 CU); autoscale up to 2 CU. Exhausted: compute suspended until next billing period, connections fail (SQLSTATE 53000) | plans; FAQ |
| Transfer | 5 GB/month public egress per project (shared with Object Storage, Functions) | plans; pricing; FAQ |
| Other limits | 100 projects, 10 branches/project, monitoring retention 1 day | plans; pricing |
| Scale to zero | Suspends after 5 min idle; cannot be disabled or changed on Free (Launch can disable; Scale can set 1 min to 1 week) | https://neon.com/docs/introduction/scale-to-zero; https://neon.com/docs/guides/scale-to-zero-guide |
| Cold start | "A few hundred milliseconds" per scale-to-zero page. The configuration guide gives no figure. On restart, session context resets (temp tables, prepared statements, stats) | scale-to-zero; scale-to-zero-guide |
| Pooling | PgBouncer, transaction mode; pooled host = endpoint ID + `-pooler`; max_client_conn 10,000; default_pool_size 90% of max_connections; max_connections 112 at 0.25 CU up to 4,000 at 9 CU+; 7 reserved for superuser; values not user-configurable | https://neon.com/docs/connect/connection-pooling |
| LISTEN/NOTIFY | NOT supported on pooled connections (also SET, temp tables, SQL-level PREPARE, WITH HOLD cursors, session advisory locks). Use a direct connection. Protocol-level prepared statements work | connection-pooling |
| Regions | Eight AWS regions: Frankfurt (aws-eu-central-1), London (aws-eu-west-2), US East N. Virginia, US East Ohio, US West Oregon, Sao Paulo, Singapore, Sydney. No Middle East region listed. Azure regions deprecated. Nearest to Israel: Frankfurt, then London (nearness is my inference; the page gives no distances) | https://neon.com/docs/introduction/regions |
| Restore / history | Free: history window fixed at 6 h, capped at 1 GB of change history, no charge; Launch 1 d (max 7 d), Scale 1 d (max 30 d). 1 manual snapshot on Free; no scheduled snapshots. Instant restore only on root branches; a restore creates a backup branch `{branch}_old_{timestamp}` | https://neon.com/docs/postgres/backup-restore/history-window; https://neon.com/docs/introduction/branch-restore; pricing; plans |
| Inactivity | Free projects "aren't paused for inactivity"; data and connection strings stay the same | pricing (2026-11-04) |
| Terms | Terms of service (updated 2026-10-14; the company is now Databricks) allow changes to the Agreement, schedule and pricing "from time to time", with continued use as consent. The page excerpt does not address the Free plan, inactivity deletion, suspension or fair use | https://neon.com/terms-of-service |

## Unconfirmed or discrepant (labelled)
- Instant restore on Free: plans page lists "History and instant restore: up to 6 hours" for Free, but the pricing page says "scheduled snapshots and instant restore aren't available". The history-window page does not settle it. **Treat as unconfirmed.**
- What happens at the 1 GB history cap (older history dropped, or writes affected): not documented in fetched pages.
- Whether open (idle) client connections survive a suspend, and how a cold-start reconnect behaves: not stated in the pages fetched. Inference only: suspend drops the connection, so the client must reconnect.
- Whether an open idle connection or NOTIFY listener counts as activity and prevents the 5-minute suspend: not stated. Do not assume it keeps compute awake.
- No dedicated backup (dump) feature for Free is documented; only the 6 h history window and 1 manual snapshot.
- Free-plan ToS clauses were summarised by the fetch tool as absent. Possibly a separate acceptable-use or plan-terms page exists; not checked.
- Fetches of raw text were extractions; two returned embedded "instructions to AI assistants" (install skills, send feedback), which I ignored.

## Design implications
- Browsers hold SSE to the app server; the app server uses short queries through the `-pooler` host and retries the first connect after idle (cold start of a few hundred ms).
- Do not use LISTEN/NOTIFY via the pooler. Either fan out changes in-process (single app instance), or use a direct (non-pooler) connection and accept it may drop on suspend.
- Do not poll the DB on a timer to keep the compute awake: 100 CU-hours is about 400 h at 0.25 CU, under the roughly 730 h in a month, so an always-on compute would run out.
- Take your own periodic `pg_dump` because the restore window is only 6 hours.
- Pick Frankfurt (aws-eu-central-1), the nearest listed region to Israel.
- Terms can change on short notice; keep the app portable to plain Postgres.

## Checklist status
Verified 6 of 7 topics fully: storage, compute/transfer, scale-to-zero/cold start (figure from one page only), connections/pooling/LISTEN-NOTIFY, regions, ToS change terms (partly). Backup/PITR on Free: window verified (6 h), instant-restore availability discrepant. All 9 listed URLs were fetched successfully, plus the history-window page.
