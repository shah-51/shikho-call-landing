# Workspace Digest (offline hub snapshot)

> **Generated 2026-09-21 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-09-17 :: Git calls a squash-merged branch unmerged, and resolving its conflicts rolls main back

**Source:** `data-agent`.
**Who needs it:** every repository that merges pull requests by squash: data-agent, paid-ads-analytics, organic-social-analytics, shah-portfolio, outbound, career-ops, and any future repo.

Asked to merge all outstanding work, `git branch -r --no-merged origin/main` listed 30 branches. Twenty four had pull requests already squash-merged. A squash merge writes a NEW commit, so the branch's own commits never become ancestors of main and git reports the branch as unmerged forever. By the time anyone looks, main has moved on, so merging the branch conflicts, and each conflict is the old version of the code arguing with the new one. Resolving them toward the branch would have regressed main across dozens of files. Takeaway: "unmerged" by ancestry is not "unmerged work". Before merging a branch, check its pull request state (`gh pr list --state all --head <branch>`), and compute what merging would change without touching the working tree: `git merge-tree --write-tree origin/main <branch>`. If the resulting tree equals main's, the branch adds nothing. Then test the MERGED result, because branch CI proves the branch against the base it was cut from, not today's main. A related trap from the same session: `gh search code` returns nothing at all for an unindexed private repository, so run a control query for a term that must match before believing any "not found".

### 2026-09-17 :: Anything run OUTSIDE a Claude session cannot see packages installed inside one

**Source:** `data-agent`.
**Who needs it:** every project that runs a Scheduled Task, a service, CI, or a script on another machine: paid-ads-analytics, organic-social-analytics, outbound, content-ops, career-ops, shah-portfolio, and the workspace's own scheduled scripts.

Claude desktop is a packaged Windows app, and Windows redirects every AppData\Roaming write made by a packaged app AND BY EVERY PROCESS IT STARTS into `%LOCALAPPDATA%\Packages\Claude_<id>\LocalCache\Roaming`. So every `pip install --user` run from a Claude session landed there, and so did any npm or tool config written under Roaming. Inside a session the two locations appear merged under the ordinary path; anything started another way sees only the real one. Measured on this machine: 344 Python packages visible inside a session, 120 visible to a Scheduled Task, with fontTools, PIL and google-genai among the missing. A detached job died in one second on its first import. It is hard to diagnose because sys.executable, sys.path, site.getusersitepackages(), APPDATA and USERPROFILE print IDENTICALLY in both processes; only listing the directory's contents shows the two views differ. Takeaway: a script that works in a session is not evidence it works from Task Scheduler, CI or a colleague's machine. Either install dependencies from outside the session, or add the virtualized site-packages to PYTHONPATH as `data-agent/tools/creative_intel/run_detached.py` does. When two processes with identical configuration disagree about whether a file exists, compare the filesystem view, not the configuration.

### 2026-09-17 :: A gate that passes on the point estimate certifies whatever a small sample got right

**Source:** `data-agent`.
**Who needs it:** any project that sets a threshold on a measured rate: paid-ads-analytics, organic-social-analytics, outbound (reply and conversion rates), content-ops.

A validation table enforced `passed = (score >= threshold)`. Applying the first threshold showed that nine perfect trials would have been recorded as TRUSTED at a 0.95 bar, although the 95% Wilson interval on 9 of 9 reaches 0.70. The fix moves the verdict to the LOWER BOUND of the interval, so a small sample fails for want of evidence rather than passing for want of contradiction. The error had already been corrected three times in how numbers were displayed and survived in the one place that decided anything. Takeaway: find where a number becomes a decision and check that place specifically, because a correction applied everywhere a number is shown does not reach where it is acted on. The same applies to any A/B test, campaign kill rule or quality gate that compares a rate with a bar.

### 2026-09-09 :: A connection opened before a long fetch is IDLE for the whole fetch, and Neon su

**Source:** `.`.
**Who needs it:** paid-ads-analytics, organic-social-analytics.

A connection opened before a long fetch is IDLE for the whole fetch, and Neon suspends it. pull_meta_performance fetched 165,433 hourly rows over several minutes and then lost every one: psycopg AdminShutdown, 'terminating connection due to administrator command', raised by the FIRST count query of the write. The expensive work succeeded and the cheap work it depended on had gone away while it ran. Takeaway: acquire the database connection at WRITE time, not at start, and reopen it if it has died. Both performance pullers now hold a lazy {n: None} handle, reconnect on AdminShutdown / OperationalError / InterfaceError, and retry the write exactly once; anything that is not a dead connection re-raises immediately, because retrying a constraint violation on a fresh connection just hides the real error. This is a different failure from the flush-at-the-end bug and the guarded accumulator does not fix it: the rows were held correctly, the write itself could not run.

### 2026-09-09 :: An API's own error-enumeration can be STALE, so it yields a candidate list rathe

**Source:** `data-agent`.
**Who needs it:** paid-ads-analytics, organic-social-analytics.

An API's own error-enumeration can be STALE, so it yields a candidate list rather than a true one. Instagram's account insights edge, handed a nonsense timeframe, answers 'must be one of the following values: last_90_days, this_week, prev_month, this_month, last_30_days, last_14_days'. Four of those six are then rejected with 'the timeframe parameter specified last_30_days is no longer supported for engaged_audience_demographics and reach'. Only this_month and this_week work. This matters because this project adopted error-enumeration as the fix for 404ing docs and had started treating it as authoritative: it is better than the docs and still not the account. Takeaway: enumerate to get CANDIDATES, then try every one and record which actually returned data. That is exactly what probe_capability and probe_organic do, and it is why they store a verdict per surface instead of a metric list.

### 2026-09-09 :: A resume-by-already-collected set silently blocks a newly ADDED metric from ever

**Source:** `data-agent`.
**Who needs it:** organic-social-analytics, paid-ads-analytics.

A resume-by-already-collected set silently blocks a newly ADDED metric from ever reaching existing rows. pull_organic skipped any post holding ANY lifetime metric, which is correct for resuming an interrupted run. Two metrics were added to its list on 2026-09-09, the pull re-ran, 9,361 posts were skipped as done while carrying neither of them, and it printed a successful backfill of 34 rows. 'Already collected' has to mean 'already collected WHAT THIS RUN ASKS FOR'. The fix that did NOT work was a count threshold (--min-metrics): Facebook posts legitimately hold 5 metrics and Instagram 12, because Meta refuses several per media type, so any single number either re-fetches the whole account forever or skips exactly the posts needing the backfill. The fix that works is naming the metric: --require total_views skips only posts that already have it, which cut a 5,852-post re-fetch to 770. Takeaway: when a resume set is keyed on the presence of work rather than on the SHAPE of the work requested, adding to the request is invisible to it.

### 2026-09-09 :: The 'one open gap blocked by the platform' was a wrong parameter, and it took th

**Source:** `data-agent`.
**Who needs it:** paid-ads-analytics, organic-social-analytics.

The 'one open gap blocke

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-09-14 |
| `August 2026/cartora-canon` | Cartora Canon, project context — The single source of truth for Cartora's business and strategy. It is the "brain": strategy, market, | `shah-51/cartora-canon` | 2026-09-15 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-09-15 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-09-14 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-14 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-09-14 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-09-14 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-09-14 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-09-14 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-09-14 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-09-14 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-09-14 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-09-14 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-09-21 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-09-14 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-09-14 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-09-14 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-09-16 |
| `May 2026/purple-fox/cartora-site` | CLAUDE.md, cartora-site (cartora.shah.works) — line, the route count and the gate list were each checked against `package.json`, `app/sitemap.js` and | `shah-51/cartora-site` | 2026-09-16 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-09-14 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-09-14 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-09-14 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-09-14 |
| `Shikho` | Shikho · domain entry — This is its own repo (`shah-51/shikho`), sitting at `D:\Shahriar\Claude\Shikho\` inside the | `shah-51/shikho` | 2026-09-21 |
| `Shikho/brand-design/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/call-now-landing` | shikho-call-landing — the "Call Now" landing page — A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the | `shah-51/shikho-call-landing` | 2026-09-14 |
| `Shikho/campaigns/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/shikho` | 2026-08-17 |
| `Shikho/campaigns/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/data-agent` | Shikho/data-agent · redirect stub — This folder is a deprecated pointer. The project moved to its own repo on 2026-08-04. | `shah-51/shikho` | 2026-09-02 |
| `Shikho/measurement/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/shikho` | 2026-09-14 |
| `Shikho/measurement/Reporting/marketing-data-hub` | marketing-reporting-hub — cross-platform marketing reporting on Apps Script — A reusable reporting engine built on **Google Apps Script**: it pulls every market | `shah-51/marketing-reporting-hub` | 2026-09-14 |
| `Shikho/measurement/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/monthly-budget` | monthly-budget · V0 Paid Media Restructure — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-monthly-budget` | 2026-09-14 |
| `Shikho/operations/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/operations/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-08-05 |
| `Shikho/paid-ads-analytics` | CLAUDE.md — Shikho Paid Ads Analytics — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-analytics` | 2026-09-14 |
| `Shikho/paid-ads-analytics/google-ads-pipeline` | CLAUDE.md, Google Ads pipeline (sub-pipeline of paid-ads-analytics) — Daily Google Ads fetch. Pulls every campaign, ad group, ad, keyword, audience, and | `shah-51/shikho-google-ads-pipeline` | 2026-09-14 |
| `Shikho/paid-ads-analytics/meta-ads-pipeline` | CLAUDE.md · Shikho Meta Ads Pipeline — The heaviest sibling of `paid-ads-analytics`: the Meta Marketing API fetch, the | `shah-51/shikho-meta-ads-pipeline` | 2026-09-14 |
| `Shikho/paid-ads-analytics/paid-ads-dashboard` | CLAUDE.md — paid-ads-dashboard — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-dashboard` | 2026-09-14 |
| `Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-14 |
| `Shikho/shikho-organic-social-analytics/facebook-pipeline` | CLAUDE.md — facebook-pipeline — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hosted-run | `shah-51/shikho-organic-social-analytics` | 2026-09-14 |
| `Shikho/shikho-organic-social-analytics/organic-social-dashboard` | CLAUDE.md — organic-social-dashboard — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hos | `shah-51/shikho-organic-social-dashboard` | 2026-09-14 |
| `Shikho/shikho-paid-ads-private-artifacts` | Shikho Paid Ads · Private Artifacts (store) — This is an **artifact store**, not a code project. It holds sensitive paid-ads | `shah-51/shikho-paid-ads-private-artifacts` | 2026-09-14 |
| `Shikho/shongbordhona-venue-landing` | shikho-gpa5-reception — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/shikho-gpa5-reception` | 2026-09-14 |
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-09-21 |
| `Shikho/strategy/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff` | Shikho 2026 Sales & Marketing Calendar — project entry — A live, **read-only** dashboard + framework explainer for Shikho's 2026 S&M calendar: | `shah-51/shikho-sales-marketing-framework` | 2026-09-14 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy` | shikho-sales-marketing-calendar (Vercel deploy) — The live, read-only Vercel site for Shikho's 2026 Sales & Marketing calendar | `shah-51/shikho-sales-marketing-calendar` | 2026-09-14 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy_v2` | shikho-2026-calendar-v2 (Vercel deploy, COO build) — The v2 Vercel site for Shikho's 2026 S&M calendar (COO build, 2026-06-08), running in parallel | `shah-51/shikho-2026-calendar-v2` | 2026-09-14 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-09-14 |
| `data-agent` | data-agent · governed multi-source data + report + dispatch platform — push → deploy → verify), how everything is wired, and the invariants you must not break. | `shah-51/data-agent` | 2026-09-21 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
