# R1: Does OCI Always Free fit a single small Docker Compose host?

Researched 2026-11-23. **Evidence is partial**: the Oracle docs page could not be fetched (permission prompt unanswered), so only the FAQ was read directly (and via a summarizing fetch, not raw text). Items marked [UNVERIFIED] come from background knowledge and must be re-checked on the docs page before relying on them.

## Verified today (source: https://www.oracle.com/cloud/free/faq/, fetched 2026-11-23)
- Card required at signup for identity verification; periodic authorization holds (released in ~3-5 days, no charge). Not accepted: PIN debit, virtual, single-use, prepaid cards.
- Free Tier is available in every commercial OCI region; if first choice unavailable, pick an adjacent one. Region list at signup may differ from the general availability list. NoSQL Always Free is Phoenix-only (irrelevant here).
- Always Free resources are not reclaimed when the 30-day, US$300 trial ends and remain "as long as you use them within tenancy limits". Exceeding Always Free Ampere A1 limits: extras disabled, then deleted after 30 days unless upgraded.
- Abandonment: "Accounts left idle for 30 days or more may be deemed abandoned" and become eligible for suspension or termination (account-level; this app's regular use should avoid it).
- Trial paid resources are reclaimed at trial end; Always Free kept.

## [UNVERIFIED] from docs page https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
Re-confirm each:
- ARM: VM.Standard.A1.Flex, total 3,000 OCPU-hours and 18,000 GB-hours per month (= 4 OCPU / 24 GB), usable as 1-4 VMs.
- AMD: up to 2x VM.Standard.E2.1.Micro (1/8 OCPU, 1 GB RAM each).
- Storage: 200 GB total block+boot volume combined, 5 volume backups; min boot volume ~47-50 GB.
- Egress: 10 TB/month outbound data transfer; 1 reserved/ephemeral public IPv4 per instance.
- Idle reclamation: Oracle documents that Always Free compute instances may be reclaimed if idle for 7 consecutive days: 95th-percentile CPU < 20%, network < 20%, and (A1 only) memory < 20%. Stopped/reclaimed instances can be restarted by user only if not yet deleted. Re-verify exact wording and whether it still applies.
- Home region is permanent and fixed at signup; Always Free compute only in home region.

## Israel and nearby regions [UNVERIFIED]
- Israel regions: Jerusalem (il-jerusalem-1) and a Dallas-style naming not applicable; Jerusalem is a commercial region. Nearby: Frankfurt, Marseille, Milan, Amsterdam, Zurich, Dubai, Abu Dhabi, Athens(?). Capacity: A1 "Out of host capacity" errors are widely reported in popular regions (Frankfurt, etc.); smaller regions vary. Check the region list at https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm and test A1 availability during signup. Chosen home region cannot be changed later.

## Fit for this app
- One backend + static frontend + Compose: fits comfortably on 1-2 OCPU / 6-12 GB A1, or even tightly on E2.1.Micro (1 GB RAM is marginal for Docker + JVM/Node + OTel).
- Long-lived SSE: no documented connection limits for compute; egress 10 TB is far beyond need. Keep-alive pings avoid NAT/LB idle timeouts (an OCI load balancer is not needed; Always Free also includes 1 flexible LB 10 Mbps [UNVERIFIED]).
- OTel export: outbound only, within egress allowance.
- Idle risk: a few-user app likely sits under 20% CPU/net/memory, so it is the prime reclamation candidate if the rule applies. Mitigation: convert to Pay As You Go (stays free within Always Free limits; reclamation does not apply to PAYG accounts [UNVERIFIED but widely reported]).

## Risks
1. Idle reclamation of A1/Micro instances (verify rule, consider PAYG upgrade).
2. A1 capacity shortage in chosen region; home region is irreversible.
3. Card verification rejections (prepaid/virtual) and sporadic account terminations without notice reported on Oracle forums.
4. Oracle may change terms; Always Free is not contractually guaranteed.

## To do before deciding
Open the docs page above and the regions page, confirm every [UNVERIFIED] line, and check Jerusalem A1 capacity by attempting to create an instance.
