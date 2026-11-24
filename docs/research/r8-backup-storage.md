# R8: Free off-VM storage for scheduled pg_dump backups of Neon Free (issue #20)

Researched 2026-11-24 via WebFetch on vendor pages. "Page date" is what the page itself showed. Anything not stated by a fetched page is marked UNVERIFIED.

## Summary verdicts

| Candidate | Free | Separate failure domain | Scriptable | Verdict |
|---|---|---|---|---|
| OCI Object Storage (Always Free) | Yes, 20 GB, 50k req/mo (only when Always-Free-only) | NO, same Oracle account | Yes, S3-compat | Not acceptable as sole copy |
| Cloudflare R2 | Yes, 10 GB-mo, 1M Class A, 10M Class B, free egress | Yes | Yes, S3 API, SSE-C | Best fit |
| Backblaze B2 | Yes, first 10 GB, no card per signup page | Yes | Yes, S3 API + b2 CLI, SSE-B2/SSE-C | Strong second copy |
| GitHub private repo / release | Free account needs no payment info | Yes | Yes (git, gh) | Workable for tiny client-encrypted dumps, but ToS/AUP risk; not a backup service |
| Google Drive | 15 GB shared | Yes | Awkward (OAuth, rclone) | Not recommended; inactivity rule UNVERIFIED |

## Neon side

- Free plan (plans page, last updated 2026-11-18, https://neon.com/docs/introduction/plans): 1 GB storage per project; instant restore window up to 6 h (1 GB change history), not extendable on Free; 1 manual snapshot; scheduled backups not available on Free; no built-in export feature listed, `pg_dump` is the export route.
- `pg_dump` counts against the Free network transfer cap of 5 GB per project per month. Compute cap 100 CU-hours/month. If either is exhausted the compute is suspended and connections (including pg_dump) fail with SQLSTATE 53000. A 1 GB database dumped daily is well inside 5 GB only if the compressed dump is small; budget the transfer (a custom-format dump is usually smaller than 1 GB, but size is not verified for this DB).
- Computes scale to zero after 5 minutes idle; pg_dump will wake it (implied; cold start not measured).
- Backup page (updated 2026-11-09, https://neon.com/docs/guides/backup-restore): scheduled snapshots exist (retention 1 h to 35 days) but are a paid feature; snapshot storage is $0.09/GB-month. The page does not mention pg_dump.
- Connection: use the direct (unpooled) host, not `-pooler`. Pooler is PgBouncer in transaction mode and breaks session `SET` state (pooling page, updated 2026-11-14, https://neon.com/docs/connect/connection-pooling; migrate page, updated 2026-11-20, https://neon.com/docs/import/migrate-from-postgres).
- Versions: Neon supports Postgres 14-17. Use a pg_dump client equal to or newer than the server major version (a newer client works; an older one refuses). Official `postgres:<ver>` Docker image carries the tools. Recommended: `pg_dump -Fc -v -d <direct_url> -f file` (custom format; `-Z 1` suggested; `--lock-wait-timeout=20s` available). `pg_dumpall` and `pg_dump --create` are not supported. Restore with `pg_restore`, `-O` if owner roles differ.

## OCI Object Storage (Always Free)

- Limits (OCI docs "Always Free Resources", last modified 2026-06-12, https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm): 20 GB combined Standard/IA/Archive and 50,000 API requests/month for Always-Free-only tenancies; 10 GB per tier for paid or trial-with-credits. If a Free Trial ends without upgrade and you are over 20 GB, all objects are deleted.
- Egress: not stated on fetched pages (UNVERIFIED).
- S3-compatible API: yes, path-style `https://<namespace>.compat.objectstorage.<region>.oci.customer-oci.com`, Customer Secret Keys, SigV4 only (page 2026-10-19, https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm). Encrypted at rest by default; SSE-C not confirmed on this page (UNVERIFIED).
- Inactivity: Oracle free-tier page (updated 2024-09-10, https://www.oracle.com/cloud/free/): "Accounts left idle for 30 days or more may be deemed abandoned and become eligible for suspension or termination." Card is required for signup/verification; prepaid, virtual and PIN-debit cards are rejected. One free account per person; noncompliance can lead to suspension or termination.
- Failure domain: same tenancy as the VM. Account suspension or termination would take out both. Fails the stated requirement. Useful only as an extra convenience copy.

## Cloudflare R2

- Free tier (pricing page, last updated Oct 30, 2026, https://developers.cloudflare.com/r2/pricing/): 10 GB-month Standard storage, 1M Class A, 10M Class B per month; Standard only (no IA free tier); no egress fees; DeleteObject free; no minimum duration on Standard; usage beyond free billed ($0.015/GB-month Standard); allowances reset monthly.
- Card: product page says "no credit card required" (https://www.cloudflare.com/developer-platform/products/r2/, undated). Whether enabling R2 actually demands a payment method is UNVERIFIED (pricing and S3 get-started pages do not say).
- Upload: S3 API, endpoint `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`, region `auto`, API token with Object Read & Write, optionally bucket-scoped (https://developers.cloudflare.com/r2/get-started/s3/, Nov 12, 2026). Works with aws cli, rclone, boto3. Max 5 GiB single PUT (limits page, Oct 30, 2026, https://developers.cloudflare.com/r2/platform/limits/).
- Encryption: all objects AES-256 at rest with Cloudflare-managed keys (fetched from https://developers.cloudflare.com/r2/reference/data-security/, page date not shown). SSE-C implemented via the S3 headers; SSE-S3/KMS headers unimplemented (S3 API compatibility page, Nov 10, 2026, https://developers.cloudflare.com/r2/api/s3/api/). Client-side encryption (age/gpg before upload) is script-level, not a vendor feature.
- Retention: lifecycle expiration rules in days via dashboard, wrangler or PutBucketLifecycleConfiguration; deletion typically within 24 h of expiry; no versioning API (GetBucketVersioning unimplemented), so overwritten or deleted objects are not recoverable (lifecycle page, Oct 26, 2026, https://developers.cloudflare.com/r2/buckets/object-lifecycles/). Scope the token to avoid a compromised VM deleting backups (no object lock was verified).
- Inactivity and termination terms: UNVERIFIED. The R2 pricing page defers to the Cloudflare Billing Policy, and the Service-Specific Terms page fetched (Sept 28, 2026) had no R2 section. Fetch of https://developers.cloudflare.com/r2/data-security/ and .../encryption/ failed (client_error; retried once with a different path).
- Failure domain: separate from Oracle. Note Cloudflare pricing can change; the free tier is vendor-defined.

## Backblaze B2

- Free tier (pricing page, no date, footer 2024, https://www.backblaze.com/cloud-storage/pricing): first 10 GB storage always free; egress free up to 3x monthly average stored, then $0.01/GB; Class A/B/C calls free; Class D first 2,500/day free; no minimum storage duration or minimum file-size fees (transaction page also undated, https://www.backblaze.com/cloud-storage/transaction-pricing). Pricing page is stale-dated (2024 footer); re-check before committing.
- Card: cloud-storage page says "No credit card required." (https://www.backblaze.com/cloud-storage, undated). The pricing page lists card or invoice as payment methods. Whether card is needed for B2 beyond signup (for example to lift caps) is UNVERIFIED.
- Upload: S3-compatible API (page updated 10 Dec 2025, https://www.backblaze.com/docs/cloud-storage-s3-compatible-api) and native `b2` CLI. Endpoint format not stated on the fetched page (UNVERIFIED here; it is region-specific per bucket).
- Encryption: SSE-B2 (default bucket setting possible) and SSE-C, AES-256, no extra cost; file names and metadata not encrypted (https://www.backblaze.com/docs/cloud-storage-server-side-encryption, undated). Client-side: do it in the script.
- Retention: lifecycle rules (page updated 05 Feb 2026, https://www.backblaze.com/docs/cloud-storage-lifecycle-rules): hide after N days, delete N days after hiding, runs daily, minimum 1 day, versions kept by default, "keep only latest" option; Object Lock files are not deleted by lifecycle. Versioning plus Object Lock gives some protection against a compromised VM deleting backups (lock details not fetched, UNVERIFIED).
- Terms (effective April 16, 2026, https://www.backblaze.com/company/policy/terms-of-service): no inactivity clause found; usage accounts stay active until data removed and account closed; Backblaze may suspend or terminate on violations and "may delete your data"; usage billed at then-current rates; trials may be discontinued at any time. The free-10GB allowance is not described in the terms (only on the pricing page), so it could change.
- Failure domain: separate from Oracle and Cloudflare.

## GitHub private repo / release assets

- Free accounts need no payment information (ToS, effective April 27, 2026, https://docs.github.com/en/site-policy/github-terms/github-terms-of-service). One free account per person; GitHub may suspend or terminate access at any time; private repo contents are confidential to you with listed exceptions (automated scanning for malware etc.).
- Limits (https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github, undated): 100 MiB hard block per file in git, 50 MiB warning; browser uploads 25 MiB; repos ideally under 1 GB; the page says Git is not designed for large SQL files or as a backup tool. Release assets have no stated total size or bandwidth limit but each file must be under the plan's LFS max (value not stated, UNVERIFIED).
- AUP (undated, https://docs.github.com/en/site-policy/acceptable-use-policies/github-acceptable-use-policies): excessive bandwidth or storage may lead to suspension, throttling, or deletion with notice.
- Scripting: git push or `gh release upload`. Encryption is client-side only (age/gpg); never push plaintext dumps.
- Retention: you delete manually or script it; committing dumps to git bloats history (use release assets or force-orphan branch instead). Account deletion removes content within 90 days.
- Verdict: only for a small encrypted dump (well under 100 MiB), as a secondary copy. Daily commits of binary dumps are discouraged by the vendor.

## Google Drive

- 15 GB shared with Gmail and Photos (https://support.google.com/drive/answer/2375123, undated). Inactivity deletion policy: the fetched page did not mention it, so UNVERIFIED. Scripting needs OAuth and a refresh token (rclone); not S3. Not recommended: refresh-token expiry and personal-account coupling add fragility. Not researched further.

## Other S3-compatible free tiers

Not researched in this pass (no verified data). Candidates to check later if needed: other vendors' free object storage. Nothing here is a claim.

## Recommendation

1. Primary: Cloudflare R2 bucket, daily `pg_dump -Fc -Z 1` from the direct Neon URL, client-side encrypt (age) before upload, upload with aws cli or rclone, prune with a lifecycle rule (for example 30 days). A 1 GB DB at one dump per day with 30 retained copies is within 10 GB only if compressed dumps stay below roughly 300 MB (size not measured).
2. Secondary copy (optional, different vendor): Backblaze B2 with default SSE-B2 plus client-side encryption and a lifecycle rule, to avoid a single-vendor dependency.
3. Do not rely on OCI Object Storage as the sole backup; same account as the VM.
4. Always take the restore test: `pg_restore` into a scratch Neon project or local `postgres:<ver>` container.
5. Keep the Neon single manual snapshot as a quick-restore point, refreshed manually.

## Verified vs unverified

Verified (from fetched pages): Neon Free limits, pooler guidance, pg_dump version and flags; R2 free tier, lifecycle, S3 API and SSE-C support, limits; B2 free 10 GB, egress ratio, lifecycle, SSE, S3 API existence, terms text; OCI limits, idle rules, S3 endpoint; GitHub file limits and ToS points.

UNVERIFIED: R2 card requirement when enabling, R2 inactivity/termination terms (pages failed or silent); R2 and B2 free-tier permanence; B2 endpoint format, Object Lock details, B2 pricing page currency (2024 footer); OCI egress and SSE-C; GitHub release-asset per-file cap; Google Drive inactivity rule; other S3-compatible free tiers; compressed dump size of this database; whether a cold-start Neon wake affects scheduled jobs.
