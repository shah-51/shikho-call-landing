# Workspace Digest (offline hub snapshot)

> **Generated 2026-09-28 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

## 2026-09-27 · A per-item error handler can turn total, systematic failure into something that never surfaces at all (cross-pollinate auto-detect)
Source: data-agent/LEARNINGS.md, 2026-09-25 entry ("A worker can report success on every job while doing zero real work")
`data-agent-worker.service` ran for over an hour, healthy by every systemd measure, completing every `classify_image`
job with no crash or restart — because a `_load_key()` helper never found the brand's Gemini key, and the per-asset
`except Exception` handler that exists to keep one bad asset from aborting a whole batch silently swallowed every
single failure instead. "The job completed without error" was not the same claim as "the job did the thing it was
enqueued to do." Applies to any workspace pipeline that wraps per-item work in a broad try/except for throughput
(paid-ads/organic-social ingestion, Career Ops' daily evals, Outbound's per-lead sends): check what each completed
run actually *produced*, not just that the queue/service stayed healthy — a per-tenant credential cache is the same
risk one layer deeper (a worker serving multiple brands per process can keep using the first brand's key after
switching tenants, with no error, just billed against the wrong account).

## 2026-09-27 · A sub-package's package.json can correctly declare a dependency and CI still breaks, because install order — not declaration — decides reachability (cross-pollinate auto-detect)
Source: whatsapp-group-reporting/CHANGELOG.md + LEARNINGS.md, 2026-09-24 entry ("`pg` declared at root; CI was red on every run since Actions came back")
`relay/db.mjs` imports `pg`, and `relay/package.json` declared it correctly the whole time — CI still failed with
`ERR_MODULE_NOT_FOUND` because root-level test files import `relay/server.mjs` (which imports `db.mjs`) during
root's `npm test` step, which runs *before* CI's later step installs `relay/node_modules`. Checking "is this bare
import declared in the nearest package.json above it" finds nothing wrong and is still false: what matters is
which scope's `npm install` is guaranteed to have already run by the time the failing file is reached, not which
directory declares the dependency. A second, unrelated test failure (a stale `MAX_SOURCES` assert) was hiding
behind this one the whole time, because a step blocked by an earlier red CI step is never actually run, so it can
never be seen failing. Applies directly to `adops-os` (a pnpm monorepo with `apps/app` + `packages/core`, the same
shape) and any other multi-package repo here where root-level tests reach into a sub-package: trace the real
import graph reached from root's test entrypoints against root's own installed deps, don't reason from
per-directory `package.json` correctness alone — `whatsapp-group-reporting/scripts/check-runtime-deps.mjs` is a
ready-made pattern to port.

## 2026-09-27 · GitHub's "Require approval for fork pull request workflows" can also gate same-repo pushes from an automated/bot identity (cross-pollinate auto-detect)
Source: cartora-coordination/DECISIONS.md, 2026-09-26 entry
Two PRs (`data-agent#239`/`#241` and `whatsapp-group-reporting#12`) sat stuck at `conclusion: action_required` —
CI runs created but never executed, needing a manual "Approve and run workflows" click — on branches that were
**not** fork PRs, just pushes from an automated identity (a builder, or a session pushing under the owner's own
credential via the GitHub API). The setting's name and GitHub's own UI description both say "fork pull requests,"
giving no reason to suspect it reaches same-repo branches — but empirically, on this account, it did; unchecking
it under Settings → Actions → General on both repos fixed both. Applies to any repo in this workspace where an
agent/bot pushes branches or opens PRs under automation (any repo with a builder/reviewer agent loop, or any
GH Actions workflow that pushes on another's behalf): if automated CI runs sit at `action_required` with no runs
starting, test this setting directly rather than ruling it out from the docs alone.

## 2026-09-24 · A CI/deploy gate must check the command's exit code, never grep its output text — a grep gate merges over real failures (cross-pollinate auto-detect)
Source: data-agent/LEARNINGS.md, 2026-09-21 entry ("A pipeline merged a pull request over 88 failing tests, because its test step was a grep")
The ship chain was `npm test | grep summary && git commit && ... && merge`: `grep summary` succeeds whenever a
summary line exists in the output at all, regardless of whether it says "0 failed" or "88 failed" — so a real
88-test failure passed the gate and merged. The failures happened to be environmental (a worktree missing
`node_modules`) and main was green once checked directly, but nothing in the chain would have caught a genuine
regression. Applies to any repo in this workspace with a shell-scripted CI/ship/deploy gate that pipes a test
runner through `grep`/`tail`/`head` to decide pass/fail instead of checking `$?` — the fix data-agent shipped:
gate on the command's own exit code, and verify a worktree has its dependencies installed before its test run
means anything.

## 2026-09-24 · On Postgres/Neon, a schema's default privileges make a narrower `GRANT` a no-op — verify live grants, don't trust the migration's own claim (cross-pollinate auto-detect)
Source: data-agent/CHANGELOG.md + LEARNINGS.md, 2026-09-24 entry ("Migration 042 applied, and the verification found a false claim in two migrations")
Two migrations stated a role was granted "SELECT, INSERT, UPDATE and no DELETE" on a new table; it actually held
DELETE, because the schema carries a default privilege (`ALTER DEFAULT PRIVILEGES`) that grants all four
automatically to every new table, and a later, narrower `GRANT` is additive — it can never revoke what a default
privilege already gave. The false claim was believed because the migration's own printed output sat right next
to the GRANT that appeared to cause it. Applies to any project in this workspace provisioning roles/permissions
in a Postgres or Neon schema (data-agent, and any sibling pipeline using Neon per `_shared/CREDENTIAL-MAP.md`):
after any migration that claims to restrict a role, read the live grants back from
`information_schema.table_privileges` rather than trusting the migration's own narration — an explicit `REVOKE`
is the only thing that actually removes a privilege a default grant already conferred.

## 2026-09-24 · A secrets file that fails to parse can echo plaintext into the terminal — encrypt with `--input-type binary`, never a parsed mode (cross-pollinate auto-detect)
Source: data-agent/LEARNINGS.md, 2026-09-21 entry ("sops echoes the offending line when it cannot parse a dotenv file")
Encrypting a malformed `.env` file with `sops --input-type dotenv` printed part of an unencrypted value to the
terminal when the parser choked on it, because a parsed input mode has to show the reader where parsing failed.
This is a direct hit on the workspace's own canonical secrets policy (SOPS + age, see this repo's root
CLAUDE.md and `_shared/SECRETS.md`): any project encrypting a secrets file should use `sops --input-type
binary`, which never parses the contents and therefore never echoes them, and should treat any parse error
from a secrets file as a possible disclosure needing a terminal-history check, not just a formatting bug to fix
and re-run.

## 2026-09-23 · A GitHub billing failure makes CI runs finish in ~2 seconds with no steps and no test output — it reads as green-adjacent, not as an outage (cross-pollinate auto-detect)
Source: data-agent/LEARNINGS.md, 2026-09-22 entry ("'CI failed' was a billing message, and two merges sat undeployed behind it")
Three CI runs failed in two seconds with no steps: GitHub refused to start the job because of the account's
payment/spending-limit state, and the reason lived in the check run's *annotation*, not in any log a normal
"why did CI fail" read would open. Because deploys in th

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-09-21 |
| `August 2026/cartora-canon` | Cartora Canon, project context — The single source of truth for Cartora's business and strategy. It is the "brain": strategy, market, | `shah-51/cartora-canon` | 2026-09-27 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-09-27 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-09-21 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-21 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-09-21 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-09-21 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-09-21 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-09-21 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-09-21 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-09-21 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-09-21 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-04 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-09-21 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-09-28 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-09-21 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-09-21 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-09-21 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-09-21 |
| `May 2026/purple-fox/cartora-playbook-inbox/cartora-playbook` | Cartora Playbook, project context — The original Cartora business playbook (foundation, market intelligence, commercial structure, sales and | `shah-51/personal-os-workspace` | 2026-09-21 |
| `May 2026/purple-fox/cartora-site` | CLAUDE.md, cartora-site (cartora.shah.works) — line, the route count and the gate list were each checked against `package.json`, `app/sitemap.js` and | `shah-51/cartora-site` | 2026-09-21 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-09-21 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-09-21 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-09-21 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-09-21 |
| `Shikho` | Shikho · domain entry — This is its own repo (`shah-51/shikho`), sitting at `D:\Shahriar\Claude\Shikho\` inside the | `shah-51/shikho` | 2026-09-28 |
| `Shikho/brand-design/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/call-now-landing` | shikho-call-landing — the "Call Now" landing page — A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the | `shah-51/shikho-call-landing` | 2026-09-21 |
| `Shikho/campaigns/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/shikho` | 2026-08-17 |
| `Shikho/campaigns/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/data-agent` | Shikho/data-agent · redirect stub — This folder is a deprecated pointer. The project moved to its own repo on 2026-08-04. | `shah-51/shikho` | 2026-09-02 |
| `Shikho/measurement/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/shikho` | 2026-09-21 |
| `Shikho/measurement/Reporting/marketing-data-hub` | marketing-reporting-hub — cross-platform marketing reporting on Apps Script — A reusable reporting engine built on **Google Apps Script**: it pulls every market | `shah-51/marketing-reporting-hub` | 2026-09-21 |
| `Shikho/measurement/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/monthly-budget` | monthly-budget · V0 Paid Media Restructure — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-monthly-budget` | 2026-09-21 |
| `Shikho/operations/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/operations/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-08-05 |
| `Shikho/paid-ads-analytics` | CLAUDE.md — Shikho Paid Ads Analytics — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-analytics` | 2026-09-21 |
| `Shikho/paid-ads-analytics/google-ads-pipeline` | CLAUDE.md, Google Ads pipeline (sub-pipeline of paid-ads-analytics) — Daily Google Ads fetch. Pulls every campaign, ad group, ad, keyword, audience, and | `shah-51/shikho-google-ads-pipeline` | 2026-09-21 |
| `Shikho/paid-ads-analytics/meta-ads-pipeline` | CLAUDE.md · Shikho Meta Ads Pipeline — The heaviest sibling of `paid-ads-analytics`: the Meta Marketing API fetch, the | `shah-51/shikho-meta-ads-pipeline` | 2026-09-21 |
| `Shikho/paid-ads-analytics/paid-ads-dashboard` | CLAUDE.md — paid-ads-dashboard — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-dashboard` | 2026-09-21 |
| `Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-21 |
| `Shikho/shikho-organic-social-analytics/facebook-pipeline` | CLAUDE.md — facebook-pipeline — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hosted-run | `shah-51/shikho-organic-social-analytics` | 2026-09-21 |
| `Shikho/shikho-organic-social-analytics/organic-social-dashboard` | CLAUDE.md — organic-social-dashboard — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hos | `shah-51/shikho-organic-social-dashboard` | 2026-09-21 |
| `Shikho/shikho-paid-ads-private-artifacts` | Shikho Paid Ads · Private Artifacts (store) — This is an **artifact store**, not a code project. It holds sensitive paid-ads | `shah-51/shikho-paid-ads-private-artifacts` | 2026-09-21 |
| `Shikho/shongbordhona-venue-landing` | shikho-gpa5-reception — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/shikho-gpa5-reception` | 2026-09-22 |
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-09-28 |
| `Shikho/strategy/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff` | Shikho 2026 Sales & Marketing Calendar — project entry — A live, **read-only** dashboard + framework explainer for Shikho's 2026 S&M calendar: | `shah-51/shikho-sales-marketing-framework` | 2026-09-21 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy` | shikho-sales-marketing-calendar (Vercel deploy) — The live, read-only Vercel site for Shikho's 2026 Sales & Marketing calendar | `shah-51/shikho-sales-marketing-calendar` | 2026-09-21 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy_v2` | shikho-2026-calendar-v2 (Vercel deploy, COO build) — The v2 Vercel site for Shikho's 2026 S&M calendar (COO build, 2026-06-08), running in parallel | `shah-51/shikho-2026-calendar-v2` | 2026-09-21 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-09-27 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
