# Workspace Digest (offline hub snapshot)

> **Generated 2026-09-02 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-09-02 · A from-address broke every email for six days, and nothing noticed
**Source:** data-agent (console magic-link login). **Who needs it:** **every project that sends email** — report-engine, whatsapp-group-reporting, cartora-outbound, career-ops-private, social-dashboard, purple-fox, and anything using Resend, SES, Postmark or SendGrid.
`dispatch.email.fromAddress` was changed to a **gmail.com** address. Resend refuses to send from a domain you have not verified, and gmail.com can never be verified because you do not own it, so **every outbound email failed with HTTP 403 for six days** — login links and scheduled reports alike. No code changed; a config value did.
**Why it stayed invisible, which is the transferable part:** a failed send only reached `console.error`. There was no audit row, so there was nothing to query and nothing to display. It surfaced only by asking a different question — *had either invite link ever been CLICKED?* Both were six days old and unused, which prompted querying the provider directly. **"The send API did not throw" is not evidence of delivery**, and where the provider validates the sender it is not even evidence of an attempt.
**Three things worth copying:**
1. **Audit send failures as first-class events**, with the provider's own error string. A `console.error` is invisible to everyone who matters.
2. **Record outcomes, not just attempts.** An audit trail that logs "link requested" cannot answer "did anyone get in"; one that also logs `login.succeeded` can, and that question is what found this.
3. **Send from a SUBDOMAIN you control** (`data-agent.shah.works`), so its DKIM/SPF/MX sit off the apex and cannot disturb mail for the main domain. Setup is two API calls plus three DNS records and verifies in under a minute.
**The shape to watch for anywhere:** a config value, outside the repo and outside CI, that silently decides whether a whole feature works. Tests cannot see it, review cannot see it, and it fails open into a cheerful success message unless you make the failure loud.

### 2026-09-02 · Anti-enumeration silence made a product unsupportable
**Source:** data-agent. **Who needs it:** **any internal tool with a "we emailed you a link" flow** — social-dashboard, purple-fox platform, adops-os, career-ops-private, report-sender-dashboard.
Sign-in replied identically for a registered and an unregistered address, so an attacker could not discover who holds an account. Correct for a public SaaS. On a **seven-person internal tool** it meant a colleague who mistyped his own address saw "check your inbox" and waited for mail that was never sent, with no way to tell a typo from an outage. He lost a day; so did the person debugging it.
**Judge the trade by the population.** The value of learning "this colleague has an account" at a company whose staff list is public is close to zero. The cost of silence is that every failure looks identical and no user can self-diagnose. Say plainly that the address has no access, echo back what was typed so the typo is visible, and keep the strict posture behind a config flag for when the tenant list genuinely is sensitive.
**The generalisation past auth:** every branch that ends in *"we did not do the thing"* needs its own message. One reassuring sentence covering several outcomes — unknown address, failed send, rate limit — is how a product becomes unsupportable, because the one thing the user needs is which of those happened.


### 2026-08-30 · Meta custom events (app_session_start, Free Trial Enrolled, ...) are in the `conversions` field, not `actions`
**Source:** `data-agent`. **Who needs it:** `paid-ads-analytics`, `marketing-reporting-hub`, `report-engine`, and any
code pulling Meta insights. Meta's `actions` field COLLAPSES every custom app/pixel event into an opaque
`app_custom_event.other` / `offsite_conversion.fb_pixel_custom` bucket, so a per-name custom event looks "missing".
The **`conversions` field breaks them out by name** (`app_custom_event.<Name>`, `offsite_conversion.fb_pixel_custom.<Name>`)
— the exact columns Ads Manager shows. Rule that travels: **if the Ads Manager UI shows a column, the Marketing API
has it** — use the right field (`conversions`) or breakdown before concluding it's unavailable. To enumerate every
named event an account fires: insights with fields=`conversions`. data-agent meta-ads `results` now merges both.

---

### 2026-09-01 · Gate re-processing on a VERSION column, never on "is this field still empty"
**Source:** `data-agent` (`tools/creative_intel/`). **Who needs it:** anything that re-processes
records when the processor improves — `cartora-outbound` (10-attribute post classification),
`paid-ads-analytics`, `organic-social-analytics`, `career-crm` enrichment, `report-sender`.

Any stage that picks its work with `WHERE some_output_column IS NULL` is asking *"was this ever
processed"*. It cannot ask *"was this processed by the CURRENT version"*. The moment a v2 of the
processor exists, every v1 record looks finished. The run reports `0 rows to process`, exits 0, and
is indistinguishable from success. This fired three times in one day in data-agent, once over 61
records.

**Fix:** store a version string per record and gate on it —
`WHERE (version IS NULL OR version NOT LIKE 'v2%')` for "bring anything not on the current major
up to date", and `!= exact_version` for a strict re-run. Two details that matter:

- **`version != 'v2'` alone is NULL-unsafe.** In SQL `NULL != 'x'` evaluates to NULL, not TRUE, so
  records that never recorded a version are excluded from the very query meant to catch
  stragglers. Always `(version IS NULL OR version != ?)`.
- **If the version lives inside a JSON blob rather than a column, compare against it anyway.**
  Testing "does the key exist" is the same trap wearing a different hat — data-agent's thumbnail
  layer would have reported zero work at every future version, forever.

**A gate that filters work must print what it held back**, or "nothing to do" and "everything was
skipped" produce identical output.

**On enforcement:** the first response here was a status dashboard, which is NOT enforcement — it
only helps someone who remembers to run it. The fix that counted was correcting the defaults. Note
also that a blocking hook was considered and rejected: running without the re-process flag is
legitimate whenever genuinely new records exist, so a trap would fire false and teach people to
route around it. Per the workspace rule, only block what is *certain* to fail.
Full write-up: `data-agent/LEARNINGS.md` (2026-09-01).

---

### 2026-09-01 · In a one-way sync, upstream NULL means "no opinion", never "erase"
**Source:** `data-agent` (`tools/creative_intel/sync_classifications_to_neon.py`). **Who needs it:**
every project that mirrors a local store into a remote one — `report-sender` (desktop to Neon),
`career-crm`, `paid-ads-analytics`, `organic-social-analytics`, `cartora-outbound`, `wa-report`.

A sync built as `UPDATE ... SET col=%s` treats a NULL in the upstream row as an instruction to
blank the mirror. It usually is not. The mirror often holds values from an older vintage the
upstream no longer carries, and the moment a row-selection filter is widened for any reason, those
rows get admitted and their remote values destroyed. In data-agent this nulled one row's
`visual_class`, which dropped the record out of every downstream query (all of which filter
`visual_class IS NOT NULL`) — the record count fell 592 to 591 and nothing errored.

**The fix is one line per column:** `col = COALESCE(%s, col)`. Propagate what the upstream knows;
never propagate its ignorance. Accepted tradeoff: you can no longer clear a remote column through
the sync, so a genuine clear becomes an explicit migration. That is the right side to err on — a
stale value is visible in a report and fixable, a destroyed one is neither, and there is rarely a
snapshot to restore from.

**Two companion rules from the same incident.** (1)

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-08-31 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-09-01 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-08-31 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-08-31 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-08-31 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-08-31 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-08-31 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-08-31 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-08-31 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-08-31 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-08-31 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-08-31 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-09-02 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-08-31 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-08-31 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-08-31 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-08-31 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-08-31 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-08-31 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-08-31 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-08-31 |
| `Shikho` | Shikho · domain entry — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-09-02 |
| `Shikho/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-30 |
| `Shikho/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/personal-os-workspace` | 2026-08-17 |
| `Shikho/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/personal-os-workspace` | 2026-09-02 |
| `Shikho/Reporting/marketing-data-hub` | marketing-reporting-hub — cross-platform marketing reporting on Apps Script — A reusable reporting engine built on **Google Apps Script**: it pulls every market | `shah-51/marketing-reporting-hub` | 2026-09-02 |
| `Shikho/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-23 |
| `Shikho/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/call-now-landing` | shikho-call-landing — the "Call Now" landing page — A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the | `shah-51/shikho-call-landing` | 2026-09-02 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/data-agent` | Shikho/data-agent · redirect stub — This folder is a deprecated pointer. The project moved to its own repo on 2026-08-04. | `shah-51/personal-os-workspace` | 2026-09-02 |
| `Shikho/monthly-budget` | monthly-budget · V0 Paid Media Restructure — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-monthly-budget` | 2026-08-31 |
| `Shikho/paid-ads-analytics` | CLAUDE.md — Shikho Paid Ads Analytics — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-analytics` | 2026-08-31 |
| `Shikho/paid-ads-analytics/google-ads-pipeline` | CLAUDE.md, Google Ads pipeline (sub-pipeline of paid-ads-analytics) — Daily Google Ads fetch. Pulls every campaign, ad group, ad, keyword, audience, and | `shah-51/shikho-google-ads-pipeline` | 2026-08-31 |
| `Shikho/paid-ads-analytics/meta-ads-pipeline` | CLAUDE.md · Shikho Meta Ads Pipeline — The heaviest sibling of `paid-ads-analytics`: the Meta Marketing API fetch, the | `shah-51/shikho-meta-ads-pipeline` | 2026-08-31 |
| `Shikho/paid-ads-analytics/paid-ads-dashboard` | CLAUDE.md — paid-ads-dashboard — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-dashboard` | 2026-08-31 |
| `Shikho/sales and marketing calendar/agent_handoff` | Shikho 2026 Sales & Marketing Calendar — project entry — A live, **read-only** dashboard + framework explainer for Shikho's 2026 S&M calendar: | `shah-51/shikho-sales-marketing-framework` | 2026-08-31 |
| `Shikho/sales and marketing calendar/agent_handoff/vercel_deploy` | shikho-sales-marketing-calendar (Vercel deploy) — The live, read-only Vercel site for Shikho's 2026 Sales & Marketing calendar | `shah-51/shikho-sales-marketing-calendar` | 2026-08-31 |
| `Shikho/sales and marketing calendar/agent_handoff/vercel_deploy_v2` | shikho-2026-calendar-v2 (Vercel deploy, COO build) — The v2 Vercel site for Shikho's 2026 S&M calendar (COO build, 2026-06-08), running in parallel | `shah-51/shikho-2026-calendar-v2` | 2026-08-31 |
| `Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-31 |
| `Shikho/shikho-organic-social-analytics/facebook-pipeline` | CLAUDE.md — facebook-pipeline — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hosted-run | `shah-51/shikho-organic-social-analytics` | 2026-08-31 |
| `Shikho/shikho-organic-social-analytics/organic-social-dashboard` | CLAUDE.md — organic-social-dashboard — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hos | `shah-51/shikho-organic-social-dashboard` | 2026-08-31 |
| `Shikho/shikho-paid-ads-private-artifacts` | Shikho Paid Ads · Private Artifacts (store) — This is an **artifact store**, not a code project. It holds sensitive paid-ads | `shah-51/shikho-paid-ads-private-artifacts` | 2026-08-31 |
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-09-02 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-08-31 |
| `data-agent` | data-agent · governed multi-source data + report + dispatch platform — push → deploy → verify), how everything is wired, and the invariants you must not break. | `shah-51/data-agent` | 2026-09-02 |
| `data-agent/showcase` | data-agent/showcase · sales and demo materials — Presentation and recording assets for pitching the data-agent product. Not deployed; used locally for | `shah-51/data-agent` | 2026-09-02 |
| `data-agent/tools/creative_intel` | creative_intel — Context for any session working here — It covers all filter specs, known traps, use case playbooks, and config values. | `shah-51/data-agent` | 2026-09-02 |
| `video-studio` | video-studio · Claude Code-driven showcase video editor — A local pipeline where **Claude Code edits video**. The user records (screen / camera / | `shah-51/video-studio` | 2026-09-02 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
