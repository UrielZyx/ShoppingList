# R7: Free automated code review tools and what each consumes

Researched 2026-11-24 for UrielZyx/ShoppingList issue #18. Every claim below comes from a WebFetch of the cited page on this date. Pages rarely show a publication date, so "retrieved 2026-11-24" is the as-of date unless stated. Items marked UNVERIFIED could not be read or were not stated by the source. Nothing is from memory.

Note: GitHub's Copilot billing has moved from "premium requests" to "AI credits" (1 credit = $0.01). One older page still describes premium requests (see 1.2), so docs conflict by plan and date.

## 1. GitHub Copilot code review

### 1.1 Plans
- Free: code review NOT included. Sources: https://docs.github.com/en/copilot/concepts/agents/code-review ("Available for all paid Copilot plans"; Free "does not include Copilot code review") and https://github.com/features/copilot/plans (Free excludes code review).
- Caveat: https://docs.github.com/en/copilot/get-started/plans lists Free as "Only 'Review selection' in VS Code" (an IDE feature). The two pages disagree slightly; the IDE-only item is not PR review.
- Pro ($10/mo, 1,500 credits), Pro+ ($39, 7,000), Max ($100, 20,000): code review included (features/copilot/plans). Business $19 and Enterprise $39 per seat: pooled credits (get-started/plans; availability markers for code review on that page UNVERIFIED).
- Free cannot buy extra credits (features/copilot/plans).

### 1.2 What it consumes
- AI credits: "each time Copilot reviews a pull request or reviews code in your IDE, the interaction consumes AI credits." Estimate $0.05-$1 per Lite review, $0.25-$5 per Balanced review (concepts/agents/code-review).
- Premium requests: the legacy page https://docs.github.com/en/copilot/concepts/billing/copilot-requests says code review has a multiplier of 13 from 2026-06-01, but applies only to Pro/Pro+ annual subscribers who stayed on legacy billing. Current docs use AI credits.
- GitHub Actions minutes: YES. Agentic review capabilities run on Actions, on standard GitHub-hosted runners by default. Larger runners bill at a higher rate; self-hosted runners consume no minutes; Actions need not be enabled in the org; if GitHub-hosted runners are disabled, the agentic capabilities are unavailable (concepts/agents/code-review). Balanced/Max efforts "may consume marginally more GitHub Actions minutes" (https://docs.github.com/en/copilot/how-tos/use-copilot-agents/request-a-code-review/configure-automatic-review).
- Whether Actions use on a public repo is free: the Actions billing page says standard runners in public repos are free (see 2), but no page I read states how Copilot review runner time is billed on a public repo. UNVERIFIED.
- Attribution: automatic reviews bill the PR author; manual reviews bill the requester; bot-authored PRs bill the organisation; unlicensed members' usage bills the org as paid additional usage (concepts/agents/code-review).
- Public-repo behaviour of Copilot code review: the page does not mention public repos. UNVERIFIED.

### 1.3 Automatic or on request
- Default: manual. "By default, you manually request a review from Copilot on each pull request" (https://docs.github.com/en/copilot/how-tos/use-copilot-agents/request-a-code-review/use-code-review). Automatic review is opt-in.
- Manual triggers: PR sidebar Reviewer, `gh pr edit N --add-reviewer @copilot`, REST API reviewer `copilot-pull-request-reviewer[bot]`, IDEs (use-code-review).
- Automatic review: user setting (Copilot settings > Code review > "Automatic Copilot code review", plus "Review new pushes" and "Review draft pull requests"), or a branch ruleset at repo/org/enterprise level with "Automatically request Copilot code review" (configure-automatic-review). Configuring it is available on Pro, Pro+, Max, Business, Enterprise; Free is not listed.

### 1.4 Ways to switch off or stop triggering
Confirmed in docs:
1. User toggle: turn off "Automatic Copilot code review" in Copilot settings. Does not override rulesets (configure-automatic-review).
2. Do not enable "Review new pushes"/"Review draft PRs"; without push review it reviews once (configure-automatic-review).
3. Do not add a ruleset with the Copilot review rule. How to disable or delete an existing ruleset rule: not documented on that page. UNVERIFIED (check Settings > Rulesets; set Enforcement to Disabled).
4. "Only allow Copilot code review to be triggered by authorized users" (restricts who can trigger; does not turn review off) (concepts/agents/code-review).
5. Org/enterprise Copilot policy for code review, and the sub-policy "Allow members without a Copilot license to use Copilot code review in GitHub.com" (disabled by default; requires "AI credits paid usage" policy) (concepts/agents/code-review). The org policy page https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/manage-policies did not cover these; the dedicated page https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/manage-copilot-code-review returned a client error and is UNVERIFIED.
6. Personal account: Copilot settings > Visibility > "Show Copilot" = Disabled (https://docs.github.com/en/copilot/how-tos/manage-your-account/disable-copilot-free).
7. A repository-level "turn code review off" setting: not described. UNVERIFIED.

### 1.5 Budgets to zero
- Personal: github.com/settings/billing > Budgets and alerts > New budget; types Product, SKU, Bundled AI credits; scope can be a repository; "Stop usage when budget limit is reached" (user-level budgets always hard-stop); alerts at 75/90/100% (https://docs.github.com/en/billing/tutorials/set-up-budgets).
- Orgs/enterprises can use a SKU budget for Copilot code review specifically, or a Bundled AI credits budget (same page).
- "Reviews are blocked when the applicable budget is exhausted" (concepts/agents/code-review).
- A $0 budget: not addressed by GitHub docs on the budget page or the spending page (https://docs.github.com/en/copilot/how-tos/manage-and-track-spending/manage-company-spending). UNVERIFIED whether the UI accepts $0.
- Without a payment method or purchased extra usage, individual plans have no pay-as-you-go beyond the included allowance (the usage page says additional usage needs a dollar budget; https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals). Included monthly credits on a paid plan still get used; a budget does not cap included credits.

### 1.6 Instruction files it reads
`.github/copilot-instructions.md`; `.github/instructions/**/*.instructions.md` (needs `applyTo`; `excludeAgent: "code-review"` hides one); `AGENTS.md` anywhere (nearest wins); a single `CLAUDE.md` or `GEMINI.md` in root. Read from the PR head branch. No stated character limit. Does not say it follows links to a TESTING.md (https://docs.github.com/en/copilot/how-tos/use-copilot-agents/request-a-code-review/configure-coding-guidelines). Pointing it at TESTING.md therefore needs a line in AGENTS.md or copilot-instructions.md; whether it opens the linked file is UNVERIFIED.

## 2. GitHub free-tier services a tool could draw on
- Actions: standard GitHub-hosted and self-hosted runners are free in public repos; larger runners are billed even in public repos. Free plan private-repo allowance: $12/month compute, 500 MB artifacts, 10 GB cache. Without a payment method, usage blocks at the allowance; budgets and 90%/100% email alerts available (https://docs.github.com/en/billing/concepts/product-billing/github-actions).
- Packages: public packages free; private 500 MB storage and 1 GB transfer on Free; container registry currently free; blocked at quota without payment method; budgets and alerts (https://docs.github.com/en/billing/concepts/product-billing/github-packages).
- Codespaces: Free personal account 15 GB-month storage and 120 hours compute; whether public repos differ is not stated; blocked at quota without payment method; budgets and alerts (https://docs.github.com/en/billing/concepts/product-billing/github-codespaces).
- Copilot AI credits: see 1. Free plan has an allowance of unstated size (usage-based-billing-for-individuals).
- Capping: budgets at github.com/settings/billing (set-up-budgets). Keeping no payment method on file is the documented hard stop for Actions, Packages and Codespaces.

## 3. Other tools

| Tool | Free terms | Runs on | Trigger | Instruction files | Data/training |
|---|---|---|---|---|---|
| CodeRabbit | Free plan: unlimited public/private repos, PR summary only; code review via IDE/CLI (3 per hour). OSS: public repos unlimited, Team features, 1-10 PR reviews per developer/hour by stars. Public repos under 10 stars: manual trigger only (`@coderabbitai review`). Paid from $24/user/mo; 14-day trial. | Its own service (Actions use not stated; UNVERIFIED) | Auto by default; manual under 10 stars | `.coderabbit.yaml` path_instructions; auto-picks up `AGENTS.md` and `.cursorrules`. Code guidelines page unreadable (UNVERIFIED) | Not found. Trust center did not state it. UNVERIFIED |
| Qodo Merge | "We don't offer a permanent free tier"; 14-day trial; OSS programme by application; Pro Team $30/mo for 2,000 credits | Its own service | Not read | Not read | "does not train AI models on customer code" |
| PR-Agent (OSS, MIT) | Free software, but you bring an LLM key (e.g. `OPENAI_KEY`); "community-maintained legacy project of Qodo" | YOUR GitHub Actions minutes (free on public repos for standard runners, per 2) plus your LLM key cost | `issue_comment` only possible; auto flags can be set false | `.pr_agent.toml` (`extra_instructions`); no standards-file import documented | Depends on your LLM provider |
| Sourcery | Free for public GitHub/GitLab repos (AI review, summaries, rules); private from $12/dev/mo annual; 14-day trial | Its own service (UNVERIFIED) | Auto on open/push; `@sourcery-ai review`; manual-only mode not documented. UNVERIFIED | Reads AGENTS.md, CLAUDE.md, GEMINI.md, copilot-instructions, cursor/windsurf/cline files, plus `SOURCERY.md`; follows imports and linked guides | Code retention page not read. UNVERIFIED |
| Greptile | Starter free for one developer, 50 credits/month, unlimited repos; OSS free for qualified non-commercial OSI-licensed projects by application; Pro credits $1 each (base review 1 credit) | Its own service (UNVERIFIED) | Docs inaccessible (login wall). UNVERIFIED | UNVERIFIED | UNVERIFIED |
| Gemini Code Assist for GitHub | Consumer version deprecated 2026-06-18, shutdown 2026-07-17; so not available now | n/a | n/a | n/a | n/a |

Sources: CodeRabbit https://docs.coderabbit.ai/management/plans.md, https://docs.coderabbit.ai/guides/review-instructions, https://www.coderabbit.ai/pricing, https://www.coderabbit.ai/oss. Qodo https://www.qodo.ai/pricing/ (the page contradicts itself on a free Developer plan). PR-Agent https://github.com/qodo-ai/pr-agent, https://docs.pr-agent.ai/usage-guide/automations_and_usage/. Sourcery https://www.sourcery.ai/pricing, https://docs.sourcery.ai/reviews/agent-instruction-files/, https://docs.sourcery.ai/reviews/commands/. Greptile https://www.greptile.com/pricing. Gemini https://developers.google.com/gemini-code-assist/docs/deprecations.

Failed fetches (retried once or declined by the tool, UNVERIFIED): docs.coderabbit.ai guides/code-guidelines.md and configuration/yaml-configuration.md; docs.qodo.ai Qodo Merge overview; Greptile docs (login redirect); coderabbit trust-center (no data terms); GitHub manage-copilot-code-review.

Not found in this research: other tools beyond those listed (no search was run).

## 4. Standards file and request-only review
- Can read a repo-level standards file: Copilot (AGENTS.md, CLAUDE.md, GEMINI.md, copilot-instructions; TESTING.md only via an explicit reference, UNVERIFIED), Sourcery (follows imports and linked guides, strongest fit), CodeRabbit (AGENTS.md auto; yaml path_instructions), PR-Agent (`.pr_agent.toml` only).
- Request-only: Copilot (default), PR-Agent (issue_comment trigger), CodeRabbit (documented only for public repos under 10 stars; a config switch is UNVERIFIED), Sourcery (not documented), Greptile (UNVERIFIED).

## 5. Verdicts on "free and does not consume GitHub credits"
- Copilot: not free (Free plan excludes PR review) and consumes AI credits plus Actions runner time on paid plans. Fails the constraint. Avoid.
- CodeRabbit: free for public repos; runs in its own cloud, though Actions use is UNVERIFIED. Likely passes; confirm.
- Sourcery: free for public repos; likely passes; confirm.
- PR-Agent: uses your Actions minutes (free on public repos for standard runners) and a paid LLM key. Passes the GitHub-credits rule only on public repos; costs your LLM key.
- Greptile: free only for one developer (50 credits) or by OSS application; UNVERIFIED details.
- Qodo Merge: no permanent free tier; OSS application only.
- Gemini Code Assist for GitHub: shut down.

## 6. Safest way to keep Copilot from consuming anything
1. Disable "Show Copilot" in personal Copilot settings, and stay on or return to Copilot Free (which excludes PR review).
2. Do not create a ruleset that requests Copilot review; turn off the "Automatic Copilot code review" toggle.
3. Keep no payment method and no extra-usage purchase; add a budget (stop usage on) as a belt-and-braces measure, noting $0 handling is UNVERIFIED.

## Verified vs unverified
Verified: Copilot plan/credits/Actions/trigger/disable facts, GitHub Actions, Packages and Codespaces allowances, CodeRabbit plans, Sourcery pricing and instruction files, PR-Agent triggers, Gemini shutdown, Qodo and Greptile pricing. Unverified: Copilot public-repo billing, $0 budgets, repo-level off switch, CodeRabbit/Sourcery/Greptile hosting and data terms, Greptile config and triggers.
