# R1: Does OCI Always Free fit a single small Docker Compose host?

Researched 2026-11-23 via WebFetch of Oracle primary sources (summarized by the fetch tool, quotes are as returned; pages carry no visible revision date, so the source date is the fetch date). Items still unconfirmed are labelled **[UNCONFIRMED]**.

## Conclusion
Fits technically (and cheaply) on the A1 shape, but the free allowance is now **2 OCPU / 12 GB** (earlier note said 4 / 24, which was wrong or outdated). Main risks: idle reclamation (a few-user app will fall under the thresholds), A1 capacity in a one-AD region, irreversible home region, and no contractual guarantee.

## Always Free limits
Source: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm (fetched 2026-11-23)
- ARM VM.Standard.A1.Flex: "the first 1,500 OCPU hours and 9,000 GB hours per month" = 2 OCPU / 12 GB total; either one instance with everything or up to two 1-OCPU instances. Bandwidth and VNICs scale with OCPUs. Oracle Linux Cloud Developer image needs at least 8 GB. (CORRECTS prior 3,000 / 18,000 = 4 OCPU / 24 GB claim.)
- AMD VM.Standard.E2.1.Micro: up to 2 instances, 1/8 OCPU (burstable), 1 GB RAM, 1 VNIC, 1 public IP, up to 50 Mbps to internet; one availability domain only in multi-AD regions.
- Storage: 200 GB block + boot combined, 5 backups, home region only. Minimum boot volume is inconsistent on the page (47 GB in Compute section, 50 GB in Block Volume section); confirm in Console.
- Egress: 10 TB/month outbound.
- Public IP: no separate quota stated; instances need not have a public IPv4; Micro includes one per VNIC. Per-A1 public-IP count **[UNCONFIRMED]**.
- Load balancer: 1 Flexible LB, fixed 10 Mbps (tenancies created on/after 2020-12-15); 1 Network Load Balancer. 2 VCNs. Outbound SMTP port 25 blocked by default.
- Also free: APM (1,000 tracing events + 10 synthetic runs per hour), Monitoring (500M ingestion / 1B retrieval datapoints). OTel export to external backends is plain outbound traffic.

## Idle reclamation (exact text, same docs page)
> "Idle Always Free compute instances may be reclaimed by Oracle. Oracle will deem virtual machine and bare metal compute instances as idle if, during a 7-day period, the following are true: CPU utilization for the 95th percentile is less than 20%; Network utilization is less than 20%; Memory utilization is less than 20% (applies to A1 shapes only)"
- Still applies as of 2026-11-23. All three conditions must hold. This app (few users) will very likely meet them, so expect reclamation risk.
- Whether upgrading to Pay As You Go exempts instances from reclamation: **[UNCONFIRMED]**. Oracle's pages fetched do not say. The FAQ (https://www.oracle.com/cloud/free/faq/) only confirms that after upgrading "you'll continue to have access to Always Free resources, and you'll only pay for the standard resources you use", and that there is no downgrade after upgrading.
- Account-level: "Accounts left idle for 30 days or more may be deemed abandoned and become eligible for suspension or termination" (FAQ, https://www.oracle.com/cloud/free/faq/).
- Over-limit A1: all A1 instances disabled, deleted after 30 days unless upgraded to paid (FAQ and https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier.htm).

## Regions and capacity
Source: https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm (fetched 2026-11-23)
- Israel: only Israel Central (Jerusalem), il-jerusalem-1, **1 availability domain**. No Petah Tikva region listed.
- Nearby: Frankfurt eu-frankfurt-1 (3 ADs); single-AD: Marseille, Milan, Amsterdam, Zurich, Dubai (me-dubai-1), Abu Dhabi (me-abudhabi-1), Riyadh, Jeddah. Athens/Greece not listed.
- Free Tier is "currently available in all regions where commercial Oracle Cloud Infrastructure service is available" (FAQ). The regions page does not mention Always Free per region. Specific A1 stock in Jerusalem or any region: **[UNCONFIRMED]** (not published; only testable in Console).
- Always Free compute and block volumes must be in the home region; "choose the home region carefully" (freetier.htm). Trial/free/PAYG tenancies subscribe to one region only (regions page).
- "Out of host capacity" = temporary lack of Always Free shapes in home region; try another AD, retry later, or upgrade to paid (Always Free Resources page). A1 is available in any AD except South Korea North (Chuncheon).
- Whether home region can ever be changed: **[UNCONFIRMED]** (docs say to choose carefully; treat as fixed).

## Account and card
Source: https://www.oracle.com/cloud/free/ and FAQ (fetched 2026-11-23)
- Credit card (or debit that works like credit) required for identity verification. Not accepted: PIN debit, virtual, single-use, prepaid. Periodic authorization holds released by bank in about 3-5 days, no charge.
- One Free Trial / Always Free account per person; multiple accounts prohibited.
- US$300 credit is "available in select countries"; whether Israel is eligible: **[UNCONFIRMED]** (FAQ gives no list). Always Free itself has no country ban stated.
- Trial-period paid resources are reclaimed at trial end; Always Free continues, account stays active.

## Terms that could end the allowance
- Always Free is "available for an unlimited period of time" but "Some limitations apply" and "The list of services and capacity estimates are subject to change" (https://www.oracle.com/cloud/free/). New services added automatically.
- Accounts may be suspended/terminated for violating account requirements or the Cloud Services Agreement (same page, FAQ).
- The Cloud Services Agreement / Free Tier terms text itself was not fetched: **[UNCONFIRMED]** (no contractual guarantee found; explicit right to discontinue not stated on pages read).

## Fit for this app
- One backend + static frontend via Compose: comfortable on 1 OCPU / 6 GB A1 (or 2 / 12). E2.1.Micro (1 GB, 1/8 OCPU) is marginal for Docker + backend + OTel.
- Long-lived SSE: no documented connection limit for compute; use keep-alive pings. 10 TB egress is far beyond need. Micro caps at 50 Mbps. Free 10 Mbps LB is not needed.
- Postgres elsewhere and OTel export are outbound-only traffic, within egress.
- Idle rule: prime reclamation candidate; options are to accept risk and keep restorable infra-as-code, upgrade to PAYG (exemption unconfirmed), or keep a deliberately small instance so utilization crosses 20% (fragile).

## Risks
1. Idle reclamation (7 days, p95 CPU/network/memory all < 20%).
2. A1 capacity in single-AD Jerusalem; home region fixed.
3. Card rejection (prepaid/virtual) and account suspension; abandonment rule at 30 days.
4. Terms can change; Always Free limits already differ from older figures (2 OCPU/12 GB now).

## Status of previously UNVERIFIED items
Verified: A1 limits (corrected), AMD micro, storage, egress, idle rule text/thresholds, home-region rule, Israel and nearby regions, LB, card requirements, capacity-error guidance. Still unconfirmed: PAYG exemption from reclamation, Jerusalem A1 stock, home-region changeability, US$300 credit eligibility in Israel, Cloud Services Agreement text, per-A1 public IP count, exact minimum boot volume.
