# Workspace Digest (offline hub snapshot)

> **Generated 2026-08-31 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-08-30 · Embed the "ship a report" playbook in the connector's own instructions, not a manual
**Source:** `data-agent` (+ Report Sender relay). **Who needs it:** `whatsapp-group-reporting` / `report-sender` (the
relay's AGENT-CONTRACT is the mirror of this). The Data Agent's always-on MCP instructions now carry the delivery
decision so any connected agent follows it with zero human docs: email/Telegram via send_now on the data agent;
WhatsApp CANNOT be sent from a server (local session) so author HTML and publish_report to the relay under a stable
id, and the desktop app delivers. Rule that travels: when two connectors compose (data agent produces, relay ships),
put the hand-off playbook in BOTH sides' initialize instructions so an agent needs neither a runbook nor prior context.

---

### 2026-08-27 · WhatsApp delivery is local-only — pick the channel before the scheduler
**Source:** `data-agent` (scheduling the new operator prompts). **Who needs it:** **Report Sender** (owns hosted
WhatsApp delivery), and any project scheduling reports into WhatsApp.
WhatsApp sending in the current stack is whatsapp-web.js with a locally-scanned session — it runs ONLY on the
machine holding that session (the desktop relay). An Anthropic-cloud routine cannot reach a local session; the
droplet has none either. Email (Resend) and Telegram (Bot API) ARE cloud-reachable. So a lean "cloud cron" and a
"WhatsApp destination" are mutually exclusive without a hosted relay. **Rule that travels:** choose the delivery
channel first — it dictates whether the scheduler can be cloud or must be local. Scheduled WhatsApp delivery is
exactly the gap the Report Sender relay (`send.shah.works`) should close: if it accepts an authenticated send and
fans out to WhatsApp, a cloud cron can POST to it. Full write-up: `data-agent/LEARNINGS.md` 2026-08-27.

---

### 2026-08-27 · Meta "add_20_s_calls" customs are the telesales 20s call-connect, NOT registration/purchase
**Source:** `data-agent` (resolving Shikho's Meta event mapping). **Who needs it:** `paid-ads-analytics` (owns the
Meta event dictionary + campaign reporting), `Customer Lifecycle Management`, and any deck/report quoting Meta
"registrations" or "purchases" for Shikho.
Pulling Shikho's account-level `actions` for last 30d exposed that one event is triple-labeled across custom
conversions: `offsite_complete_registration_add_20_s_calls` (903), `offsite_purchase_add_20_s_calls` (902) and
`click_to_call_native_20s_call_connect` (902) are the SAME 20-second telesales call-connect. The real registration
custom is `offsite_complete_registration_add_meta_leads` (239 ≈ standard `lead` 238); standard
`offsite_conversion.fb_pixel_purchase` = 0 (Meta tracks no paid purchase — the sale closes via telesales; purchase
truth is Metabase Course_Subscriptions). **Rules that travel:** (1) a Meta custom-conversion NAME lies — match the
achieved COUNT across candidates to identify what an event really is. (2) Count a shared event once (as telesales),
never also as reg/purchase, or reports triple-count. (3) If a campaign is "optimizing for registrations" but the
custom is really the 20s-call, its CPR is actually a cost-per-call — recheck any Meta CPR figure against this.
Full write-up: `data-agent/LEARNINGS.md` 2026-08-27 (mappings PROVISIONAL pending direct account check).

---

### 2026-08-27 · Multi-tenant proof needs a THROWAWAY second tenant — "works for tenant #1" hides cross-tenant leaks
**Source:** `data-agent`. **Who needs it:** any multi-tenant / multi-brand product here — **Report Sender**
(`app.wa-report.shah.works`, multi-brand console), and any future data-agent brand onboarding.
Everything in data-agent was verified against Shikho (brand #1, hand-built config). The master-product promise —
a NEW brand connects and gets the capacities cleanly — stayed unverified until a throwaway brand was provisioned
and its output diffed. That immediately exposed a cross-tenant leak: the always-on MCP instruction guide had
Shikho's concrete internals (saved report/card IDs, source-of-truth table name, property slugs, cohort vocab)
hardcoded in SHARED code, so tenant #2's instructions carried tenant #1's IDs — wrong routing + a confidentiality
leak. **Rules that travel:** (1) never accept "works for the first tenant" as multi-tenancy proof — spin up a
throwaway tenant and diff the render. (2) Shared always-on prose (system prompts, instruction guides, templates)
is the sneakiest hiding place for a tenant-specific value — code paths were clean, the prose wasn't. (3) Concrete
per-tenant values belong in per-tenant config, never shared code; lock it with a test that fails if a known
tenant internal appears in a tenant-less render. Full write-up: `data-agent/DECISIONS.md` + `LEARNINGS.md` 2026-08-27.

---

### 2026-08-27 · Google Sheets fails by returning WRONG DATA, not by erroring
**Source:** `Reporting / whatsapp-group-reporting` (recovered from a full-transcript audit).
**Who needs it:** every repo that reads a Sheet — paid-ads-analytics, organic-social-analytics, data-agent,
Customer Lifecycle Management, marketing-reporting-hub, and any Apps Script pulling into a report.
Three traps, all silent, all cost real time. (1) **An unquoted tab name containing a space returns the wrong
data with no error.** `Registration Report!A18:E31` does not fail — it returns something. Correct form:
`'Registration Report'!A18:E31`. Quote every tab name in A1 notation, always. (2) **A service-account key is
useless until the sheet is shared with that key's `client_email`** — and the failure is indistinguishable from
a permissions bug at every other layer, so it gets debugged in the wrong place. (3) **Table width must come from
the WIDEST row, not the first** — a merged title row collapses to a single cell and truncates every data row
beneath it. Related: a tab `gid` resolved by a DOM `.click()` is unreliable (Sheets ignores synthetic clicks),
and one such wrong gid was caught only because a downloaded file's *name* did not match. **Rule that travels:
anything reading a Sheet needs an assertion about the SHAPE of what came back** — row count, column count, an
expected header — because nothing upstream will complain.

---

### 2026-08-27 · "The log says it succeeded" is not evidence — and neither is a test run in the wrong environment
**Source:** `Reporting / whatsapp-group-reporting`. **Who needs it:** anything that sends, uploads, deploys or
publishes through an external system — outbound campaigns, data-agent connectors, paid-ads/organic-social
pipelines, Cartora, any scheduled job whose only witness is its own log.
Two failure shapes, both from the same project's history. (1) **A client library's "sent" usually means "queued
locally".** `sendMessage()` resolved on the local queue, the process exited, and every media upload died at
`ack=0` — the log said Sent, nothing ever left the machine. Separately, a **cached destination id** that was no
longer valid caused messages to be accepted and delivered nowhere, **throwing no error**. Same symptom, three
occurrences, three unrelated causes. **Rule: assert on the ACKNOWLEDGEMENT, and log the resolved destination by
name, not the id.** (2) **Four consecutive "final root causes" were declared and all four were wrong**, because
the instances being validated ran in a sandboxed container rather than on the user's machine — *"repeatedly
validating the wrong reality"*. The real cause was an earlier *fix* that saved an empty config over the user's
real one. **Rules: name the machine a claim is true of; when a fix does not hold, suspect the PREVIOUS fix
before inventing a new theory (a self-inflicted cause mimics every environmental one); and never let a
load-then-save cycle write an empty result over a user's only copy.**

---

### 2026-08-30 · A checker that MODELS a system instead of asking it will drift — four times in one day
**Source:** `Reporting / whatsapp-group-reporting`. **Who needs it:** every project with a vali

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-08-25 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-08-28 |
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
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-08-31 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-08-25 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-08-25 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-08-25 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-08-25 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-08-25 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-08-25 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-08-25 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-08-25 |
| `Shikho` | Shikho · domain entry — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-31 |
| `Shikho/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-30 |
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
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-08-31 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-08-30 |
| `data-agent` | data-agent · governed multi-source data + report + dispatch platform — push → deploy → verify), how everything is wired, and the invariants you must not break. | `shah-51/data-agent` | 2026-08-31 |
| `data-agent/tools/creative_intel` | creative_intel — Context for any session working here — It covers all filter specs, known traps, use case playbooks, and config values. | `shah-51/data-agent` | 2026-08-31 |
| `video-studio` | video-studio · Claude Code-driven showcase video editor — A local pipeline where **Claude Code edits video**. The user records (screen / camera / | `shah-51/video-studio` | 2026-08-25 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
