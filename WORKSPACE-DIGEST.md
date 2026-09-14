# Workspace Digest (offline hub snapshot)

> **Generated 2026-09-14 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

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

The 'one open gap blocked by the platform' was a wrong parameter, and it took three wrong diagnoses to reach that. conversion_destination returned rows on Meta's SYNCHRONOUS insights edge and the async report job refused to read its own completed result: '(#100) Tried accessing nonexisting summary field (conversion_destination)'. Diagnosis 1, a platform limitation, was recorded in the store. Diagnosis 2, that time_increment was the problem, was disproved by running the job without it and getting the identical error. The API settled it in one call: asked to enumerate, it lists breakdowns (70 values) and action_breakdowns (16) as SEPARATE parameters, and conversion_destination appears only in the second. It was being passed as a row breakdown; the sync edge tolerated the wrong parameter and returned rows WITHOUT the field, while the async job stored it and choked reading it back. Then the CORRECTED call also returned nothing, and only a control settled that: with action_breakdowns=conversion_destination this account returns 2,713 actions carrying action_type and value only, byte-identical to a request with no breakdown, while action_device on the same window returns android_smartphone, iphone, desktop populated. So the mechanism works and that one breakdown is empty here. Three takeaways. A parameter accepted is not a parameter honoured, so a probe must check that the requested FIELD is in the payload rather than that rows came back. Two parameters with similar names and separate value lists will be confused, and the API will enumerate both for free. And when a corrected call still returns nothing, run a control on the same account and window before concluding anything: without it this would have been recorded as unavailable twice over. The investigation's real yield was three action breakdowns that DO carry data and were never ingested - action_device 212,382 rows, action_video_type 17,528 - found only by enumerating the correct list.

### 2026-09-09 :: Two patches that were nearly shipped as fixes, and what each actually needed. FI

**Source:** `data-agent`.
**Who needs it:** paid-ads-analytics, organic-social-analytics.

Two patches that were nearly shipped as fixes, and what each actually needed. FIRST: a Meta async report job that ends as 'Job Failed' says WHY, in plain fields on the job object, and this code read only async_status and discarded the rest. Both failures measured on 2026-09-09 were error_code 4 / error_subcode 1504022, 'Application request limit reached' - a RATE LIMIT, transient, nothing to do with the parameters. It had been read for a day as 'this breakdown is not obtainable' and two working breakdowns were nearly recorded as gaps on that basis. This is the third time this project has treated a stated limit as a mystery: six blind backoffs against a header carrying the exact recovery time, a documented workaround that had silently stopped working, and now this. The patch was a retry at ONE call site; async_insights has TEN, so the other nine would have gone on treating a rate limit as a permanent verdict - a fix applied at a call site instead of the shared path is not a fix. The real fix raises AsyncJobFailed carrying code, subcode and message, subclassing SystemExit so no existing caller needed editing, retries only codes in the measured transient set, and waits the recovery time Meta publishes rather than a guess. tests_async_retry.py proves all of it against a fake job runner, with no API calls. SECOND: capability_probe.ingested_at was a free-text assertion, and within an hour of its introduction one was written for a surface whose 180-day job was still running. The job failed, and the row said in the system's own voice that a surface with zero rows was ingested. It was noticed and withdrawn, but noticing is not a control. safe_write.mark_ingested now counts before it writes, REFUSES to mark a zero, and stores the query that produced the count so it can be recomputed rather than believed - the third member of the family with verified_upsert ('announced success, wrote nothing') and assert_replaced ('a patch that matched nothing'). verify_data_layer fails on any claim without evidence, and --plant-fault proves that gate goes red. All 38 existing claims were re-derived from counts; none was refused.

### 2026-09-09 :

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-09-13 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-09-13 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-09-07 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-07 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-09-07 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-09-07 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-09-07 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-09-07 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-09-07 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-09-07 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-09-07 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-09-07 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-09-14 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-09-07 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-09-07 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-09-07 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-09-13 |
| `May 2026/purple-fox/cartora-site` | CLAUDE.md, cartora-site (cartora.shah.works) — this file is stale; fix it in the session you notice. | `shah-51/cartora-site` | 2026-09-13 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-09-07 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-09-07 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-09-07 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-09-07 |
| `Shikho` | Shikho · domain entry — This is its own repo (`shah-51/shikho`), sitting at `D:\Shahriar\Claude\Shikho\` inside the | `shah-51/shikho` | 2026-09-14 |
| `Shikho/brand-design/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/call-now-landing` | shikho-call-landing — the "Call Now" landing page — A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the | `shah-51/shikho-call-landing` | 2026-09-07 |
| `Shikho/campaigns/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/shikho` | 2026-08-17 |
| `Shikho/campaigns/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/data-agent` | Shikho/data-agent · redirect stub — This folder is a deprecated pointer. The project moved to its own repo on 2026-08-04. | `shah-51/shikho` | 2026-09-02 |
| `Shikho/measurement/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/shikho` | 2026-09-07 |
| `Shikho/measurement/Reporting/marketing-data-hub` | marketing-reporting-hub — cross-platform marketing reporting on Apps Script — A reusable reporting engine built on **Google Apps Script**: it pulls every market | `shah-51/marketing-reporting-hub` | 2026-09-07 |
| `Shikho/measurement/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/monthly-budget` | monthly-budget · V0 Paid Media Restructure — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-monthly-budget` | 2026-09-07 |
| `Shikho/operations/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/operations/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-08-05 |
| `Shikho/paid-ads-analytics` | CLAUDE.md — Shikho Paid Ads Analytics — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-analytics` | 2026-09-07 |
| `Shikho/paid-ads-analytics/google-ads-pipeline` | CLAUDE.md, Google Ads pipeline (sub-pipeline of paid-ads-analytics) — Daily Google Ads fetch. Pulls every campaign, ad group, ad, keyword, audience, and | `shah-51/shikho-google-ads-pipeline` | 2026-09-07 |
| `Shikho/paid-ads-analytics/meta-ads-pipeline` | CLAUDE.md · Shikho Meta Ads Pipeline — The heaviest sibling of `paid-ads-analytics`: the Meta Marketing API fetch, the | `shah-51/shikho-meta-ads-pipeline` | 2026-09-07 |
| `Shikho/paid-ads-analytics/paid-ads-dashboard` | CLAUDE.md — paid-ads-dashboard — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-dashboard` | 2026-09-07 |
| `Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-07 |
| `Shikho/shikho-organic-social-analytics/facebook-pipeline` | CLAUDE.md — facebook-pipeline — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hosted-run | `shah-51/shikho-organic-social-analytics` | 2026-09-07 |
| `Shikho/shikho-organic-social-analytics/organic-social-dashboard` | CLAUDE.md — organic-social-dashboard — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hos | `shah-51/shikho-organic-social-dashboard` | 2026-09-07 |
| `Shikho/shikho-paid-ads-private-artifacts` | Shikho Paid Ads · Private Artifacts (store) — This is an **artifact store**, not a code project. It holds sensitive paid-ads | `shah-51/shikho-paid-ads-private-artifacts` | 2026-09-07 |
| `Shikho/shongbordhona-venue-landing` | shikho-gpa5-reception — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/shikho-gpa5-reception` | 2026-09-08 |
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-09-14 |
| `Shikho/strategy/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff` | Shikho 2026 Sales & Marketing Calendar — project entry — A live, **read-only** dashboard + framework explainer for Shikho's 2026 S&M calendar: | `shah-51/shikho-sales-marketing-framework` | 2026-09-07 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy` | shikho-sales-marketing-calendar (Vercel deploy) — The live, read-only Vercel site for Shikho's 2026 Sales & Marketing calendar | `shah-51/shikho-sales-marketing-calendar` | 2026-09-07 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy_v2` | shikho-2026-calendar-v2 (Vercel deploy, COO build) — The v2 Vercel site for Shikho's 2026 S&M calendar (COO build, 2026-06-08), running in parallel | `shah-51/shikho-2026-calendar-v2` | 2026-09-07 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-09-07 |
| `data-agent` | data-agent · governed multi-source data + report + dispatch platform — push → deploy → verify), how everything is wired, and the invariants you must not break. | `shah-51/data-agent` | 2026-09-13 |
| `data-agent/showcase` | data-agent/showcase · sales and demo materials — Presentation and recording assets for pitching the data-agent product. Not deployed; used locally for | `shah-51/data-agent` | 2026-09-09 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
