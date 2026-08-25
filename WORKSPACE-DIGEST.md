# Workspace Digest (offline hub snapshot)

> **Generated 2026-08-25 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-08-25 · Machine clients cannot log in: split the credential from the identity before arguing about passwords
**Source:** data-agent (console auth rebuild). **Who needs it:** **every project that serves BOTH a machine client and a human** — report-sender / whatsapp-group-reporting (relay tokens), cartora-outbound, social-dashboard (Atlas), purple-fox, adops-os, career-ops-private, and any future MCP server.
The ask was "replace token login with user id and password". Taken literally it is **impossible**: Claude, ChatGPT and Gemini have no login UI and can only present a bearer credential. But the goal behind it was sound, and the useful reframe is that **ONE token was doing TWO jobs** — a machine credential for an AI client, and the human login for the web console. Those have opposite requirements (long-lived and portable vs short-lived and revocable), which is exactly why people were copy-pasting tokens between devices: pasting one *was* the way to sign in.
**The split:** the console gets human identity (session cookie), the AI client keeps the bearer token, and the console becomes the only place a token is ever produced. Nothing about the MCP surface changed, so no client broke.
**Magic link beat password on the merits, not on taste:** email was **already** the identity (`users.user_email` is the unique natural key and every active grant carried one), delivery already existed, and **a password needs a reset flow that requires email anyway** — so password is strictly *more* machinery on top of everything magic link needs, plus a new class of stored secret that people reuse across sites. If your project already has email + an email-keyed user table, passwordless is the smaller build, not the fancier one.
**Generalises past auth:** when a request cannot be implemented as stated, the goal behind it usually still can. Name the jobs the current design conflates before debating the mechanism.

### 2026-08-25 · Moving from an API key header to a session cookie ADDS a vulnerability class (and three costs)
**Source:** data-agent. **Who needs it:** **any project migrating from API keys/bearer tokens to browser sessions** — social-dashboard, adops-os, purple-fox platform, career-ops-private, report-sender-dashboard.
`Authorization: Bearer` is **structurally immune to CSRF**: a hostile page cannot set that header cross-origin, and cannot read the token out of another origin's storage. Cookies are attached by the browser **automatically**, so the moment sessions land, every mutating route becomes CSRF-reachable. The move is still correct — `HttpOnly` means an XSS can no longer exfiltrate a credential that works forever from any machine — but it is a **trade, not an upgrade**, and it is easy to ship the upside while silently inheriting the downside.
**The three costs, all mandatory, none optional:**
1. `SameSite=Strict` on the session cookie (barrier one).
2. A **per-session CSRF token** required on every cookie-authenticated mutation, compared in constant time (barrier two). Bearer requests must stay exempt, or you have broken your API for no gain.
3. **Retire `Access-Control-Allow-Origin: *` on the cookie routes.** Wildcard origin plus credentials is invalid and unsafe; split CORS by surface — open for the machine API, own-origin-with-credentials for the console.
**Two more that are easy to miss:** an unauthenticated "email me a link" endpoint is an **email bomb** without per-address and per-IP rate limits; and it must reply **identically for known and unknown addresses** or it becomes a user-enumeration oracle. Verify that by diffing the two responses byte for byte, not by reading the code.
**Scope the session deliberately:** here a session authorises the management plane ONLY and carries no data-source scope at all, so a stolen session cannot query a BI tool or an ad account. That cap is what made it safe to let a session take the highest role across a person's tokens.


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

### 2026-08-25 · A deny-list on an open-ended input is an escala

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
