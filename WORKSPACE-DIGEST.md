# Workspace Digest (offline hub snapshot)

> **Generated 2026-08-25 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-08-25 · The commonest dashboard bug is a READER WITH NO WRITER, and it is now a CI test you can copy
**Source:** data-agent (admin console build). **Who needs it:** **every project with a DB-backed dashboard or report view** — shikho-paid-ads-dashboard, shikho-organic-social-analytics, social-dashboard (Atlas/Cartora), purple-fox, adops-os, career-ops-private.
In one build this shipped **twice**. First: three new Neon tables, twelve endpoints reading them, a writer for **one** table — so six of twelve endpoints could only ever return empty. Then, after fixing that: **eight columns declared with no writer**, three of which the UI depended on (`params_summary` was the field the audit row's "Details" expander reads, so every row would have expanded to nothing), plus **four indexes on always-NULL columns** costing writes on the hot path for zero reads.
**Why review does not catch it:** an empty table and a correct-but-unfed query are **indistinguishable**. The SQL is valid, the endpoint returns 200, the tests pass, and the tab is just... empty. It reads as "no data yet".
**The tells, worth grepping for in your own repo:** a column that a WHERE clause filters on but no INSERT lists; a read endpoint whose first row nobody can name the code path for; a table created in the same commit as the endpoints that read it.
**What actually fixed it** (and what to copy): `data-agent/tests/dashboard-schema.test.mjs` — parses the DDL, extracts every INSERT column list from the writer module, and **fails CI** on any column that is declared but never written, any index covering such a column, and any GRANT of DELETE. Deliberately-unwritten columns live in a `RESERVED` map **with a reason string**, so "unexplained" is the only thing it can flag. ~90 lines, no dependencies, and it was verified by injecting a fake dead column and watching it fail. Per the workspace escalation rule: this recurred, so it became enforcement rather than a third write-up.
**The review question that generalises:** for each new read endpoint, **name the code path that produces its first row.** If you cannot, it is decoration.

### 2026-08-25 · Validate generated SQL against real Postgres with a TEMP table — two bugs died instantly that survived careful reading
**Source:** data-agent (Neon dashboard writers). **Who needs it:** **every project that hand-writes SQL against Neon/Postgres** — shikho-paid-ads-analytics, shikho-organic-social-analytics, social-dashboard, purple-fox, cartora-outbound, adops-os, career-ops-private, report-engine.
Two bugs in hand-written SQL passed review and died on first contact with a real database:
1. **`ON CONFLICT DO UPDATE` references the target row by the table's UNQUALIFIED name.** `access_registry.source_status.query_count + 1` is not the documented form; `source_status.query_count + 1` is.
2. **A parameter that is bound but never referenced fails the WHOLE statement** with `42P18 could not determine data type of parameter $n`. This happened because an error-path branch reused a shared params array whose latency slot its SQL never mentions. **This was the dangerous one:** it sat inside a fire-and-forget write (`.catch(() => {})`), so in production every source-error row would have been discarded silently, and the Sources tab would have shown a permanently healthy system.
**The technique, ~2 minutes and needs no schema privileges:** `CREATE TEMP TABLE` mirroring the real shape, run the statements **verbatim** through Neon's HTTP `/sql` **batch** endpoint (a batch shares one session, so the TEMP table is visible across statements), then assert on the read-back. It caught the 42P18 immediately and also proved the running-average arithmetic (100 → 150 → 200, preserved through a NULL write).
**Rule:** fire-and-forget writes deserve this **more** than normal ones, precisely because they can never fail loudly. If a write is wrapped in an empty catch, you have chosen to never learn it is broken — so prove it works once, up front.

### 2026-08-25 · A deny-list on an open-ended input is an escalation; and prefer structure over validation
**Source:** data-agent (token minting + self-service rotation). **Who needs it:** **every project that issues credentials or checks roles** — cartora-outbound, adops-os, social-dashboard (BetterAuth), career-ops-private, purple-fox, and any MCP server with scoped tokens.
**The escalation:** the token minter accepted any `role` string. The gate that protects report management (`assertCanManageReports`) only **denies** `analyst` and `viewer` — so `role: "typo"`, or `role: "superuser"`, sailed past it and behaved like an operator. A deny-list over an open-ended input grants by default to anything it has not heard of. Fixed with a `MINTABLE_ROLES` allowlist validated at the single point where roles enter the system.
**The stronger idea, from building self-service token rotation:** the requirement was "a user may replace their own token but must never widen their own scope". The obvious build reads role/sources from the request and validates them against the current grant — that works, and it breaks the first time someone edits the validation. Instead the replacement is produced by `INSERT ... SELECT` from the old row, and the builder binds **exactly three parameters** (old hash, new hash, optional label). **There is no place for a scope value to enter**, so the property holds by shape rather than by check. Expiry is carried across unchanged, so rotation is a replacement and never a renewal.
The tests assert `params.length === 3` and that no scope column is ever bound as `col = $n` — so a future change that **adds** a parameter fails, even if it looks harmless. Verified adversarially against a live server: a rotate body carrying `role: admin, sources: *, canSend: true, expiresInDays: 3650` changed nothing.
**Both rules in one line:** allowlist what may enter, and where a rule must hold forever, **remove the ability to break it rather than detecting the break**.

### 2026-08-25 · A green test suite that does not touch your diff is not verification (and concurrent sessions split LEARNINGS.md)
**Source:** data-agent. **Who needs it:** **every session in every repo**, and especially any repo where several agents write knowledge files at once.
**The reporting failure:** ~340 new lines were shipped, `node --test` reported **212 passing**, and that was presented as verification. None of those tests imported the new code. A passing suite says *"I broke nothing"*; it never says *"what I added works"*. Those are different claims and only the first was true. **Before citing a test run as evidence, confirm the suite actually exercises the new code** — if it does not, the honest report is "no regressions, new code untested".
**The concurrency finding:** `data-agent/LEARNINGS.md` gained entries from a **parallel session** appended at the BOTTOM while this session had been prepending at the TOP. Nothing was lost, but the file now reads newest-first then oldest-first, so "read the top for recent context" gets half the story — which matters because the hub aggregates these files. **State the ordering convention inside the file, and when you find entries in the other order, leave them and note the split** rather than reordering another session's work (reordering destroys the diff that shows who wrote what).


### 2026-08-25 · One deploy failed SIX times — the anatomy, and why documentation could not stop it
**Source:** whatsapp-group-reporting (relay deploy). **Who needs it:** **every project with a droplet/remote deploy** — data-agent, atlas-pdf, purple-fox, report-sender-dashboard — and **every session that writes a runbook and expects it to be followed**.
A single one-file deploy took **six attempts**. Anatomy, because each failure was a *different* mechanism and only the last two were the "obvious" kind:
1–3. **Droplet commands and a local `scp` pasted into the wrong shell.** A deploy spans two machines and the failure is **asymmetric**: droplet c

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-08-25 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-08-25 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-08-25 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-08-25 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-08-25 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-08-25 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-08-25 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-08-25 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-08-25 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-08-25 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-08-25 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-08-25 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-08-25 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-08-25 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-08-25 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-08-25 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-08-25 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-08-25 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-08-25 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-08-25 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-08-25 |
| `Shikho` | Shikho · domain entry — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-25 |
| `Shikho/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/personal-os-workspace` | 2026-08-17 |
| `Shikho/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/personal-os-workspace` | 2026-08-25 |
| `Shikho/Reporting/marketing-data-hub` | marketing-reporting-hub — cross-platform marketing reporting on Apps Script — A reusable reporting engine built on **Google Apps Script**: it pulls every market | `shah-51/marketing-reporting-hub` | 2026-08-25 |
| `Shikho/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-23 |
| `Shikho/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/call-now-landing` | shikho-call-landing — the "Call Now" landing page — A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the | `shah-51/shikho-call-landing` | 2026-08-25 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/monthly-budget` | monthly-budget · V0 Paid Media Restructure — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-monthly-budget` | 2026-08-25 |
| `Shikho/paid-ads-analytics` | CLAUDE.md — Shikho Paid Ads Analytics — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-analytics` | 2026-08-25 |
| `Shikho/paid-ads-analytics/google-ads-pipeline` | CLAUDE.md, Google Ads pipeline (sub-pipeline of paid-ads-analytics) — Daily Google Ads fetch. Pulls every campaign, ad group, ad, keyword, audience, and | `shah-51/shikho-google-ads-pipeline` | 2026-08-25 |
| `Shikho/paid-ads-analytics/meta-ads-pipeline` | CLAUDE.md · Shikho Meta Ads Pipeline — The heaviest sibling of `paid-ads-analytics`: the Meta Marketing API fetch, the | `shah-51/shikho-meta-ads-pipeline` | 2026-08-25 |
| `Shikho/paid-ads-analytics/paid-ads-dashboard` | CLAUDE.md — paid-ads-dashboard — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-dashboard` | 2026-08-25 |
| `Shikho/sales and marketing calendar/agent_handoff` | Shikho 2026 Sales & Marketing Calendar — project entry — A live, **read-only** dashboard + framework explainer for Shikho's 2026 S&M calendar: | `shah-51/shikho-sales-marketing-framework` | 2026-08-25 |
| `Shikho/sales and marketing calendar/agent_handoff/vercel_deploy` | shikho-sales-marketing-calendar (Vercel deploy) — The live, read-only Vercel site for Shikho's 2026 Sales & Marketing calendar | `shah-51/shikho-sales-marketing-calendar` | 2026-08-25 |
| `Shikho/sales and marketing calendar/agent_handoff/vercel_deploy_v2` | shikho-2026-calendar-v2 (Vercel deploy, COO build) — The v2 Vercel site for Shikho's 2026 S&M calendar (COO build, 2026-06-08), running in parallel | `shah-51/shikho-2026-calendar-v2` | 2026-08-25 |
| `Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-25 |
| `Shikho/shikho-organic-social-analytics/facebook-pipeline` | CLAUDE.md — facebook-pipeline — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hosted-run | `shah-51/shikho-organic-social-analytics` | 2026-08-25 |
| `Shikho/shikho-organic-social-analytics/organic-social-dashboard` | CLAUDE.md — organic-social-dashboard — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hos | `shah-51/shikho-organic-social-dashboard` | 2026-08-25 |
| `Shikho/shikho-paid-ads-private-artifacts` | Shikho Paid Ads · Private Artifacts (store) — This is an **artifact store**, not a code project. It holds sensitive paid-ads | `shah-51/shikho-paid-ads-private-artifacts` | 2026-08-25 |
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-08-25 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-08-25 |
| `data-agent` | data-agent · governed multi-source data + report + dispatch platform — push → deploy → verify), how everything is wired, and the invariants you must not break. | `shah-51/data-agent` | 2026-08-25 |
| `data-agent/tools/creative_intel` | creative_intel — Context for any session working here — Production classifiers and analysis tools for ad creative intelligence. Currently ships: | `shah-51/data-agent` | 2026-08-25 |
| `video-studio` | video-studio · Claude Code-driven showcase video editor — A local pipeline where **Claude Code edits video**. The user records (screen / camera / | `shah-51/video-studio` | 2026-08-25 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
