# Workspace Digest (offline hub snapshot)

> **Generated 2026-09-07 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-09-06 :: Drill-down shipped, and it immediately invalidated a quarter of the report's own

**Source:** `data-agent`.
**Who needs it:** paid-ads-analytics, organic-social-analytics.

Drill-down shipped, and it immediately invalidated a quarter of the report's own rows: 82 of 342 attribute cells with 3+ creatives have ONE creative holding more than half the cell's impressions, several above 96 percent. Those rows read as attribute findings and are one ad's result. direct_cta showed 5,718 installs across 6 creatives; 4,086 of them, 71 percent, come from a single ad. A number nobody can interrogate one level down will be over-read, and the fix was not more analysis but letting the reader open the row.

### 2026-09-06 :: The brief carried 1,783 creative thumbnails and every one was inside a collapsed

**Source:** `data-agent`.
**Who needs it:** cartora, shah-portfolio.

The brief carried 1,783 creative thumbnails and every one was inside a collapsed details element: zero visible without clicking, and zero charts of any kind. A report about creative work showed no creative work. Feature counts hid it - I verified '1,783 images embedded' and never asked how many a reader SEES. Worse, the CSS for the fix landed as literal text inside a CSS string, so charts and cards rendered completely unstyled while my grep for 'is the markup there' passed. Count what reaches the reader, not what reaches the file.

### 2026-09-04 · CleverTap outage RESOLVED: app 6.0.7 shipped ~21:00 on 3 Sep 2026 and ingestion 
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management, paid-ads-analytics.

### 2026-09-03 - cartora.shah.works is live; and the trailingSlash trap that broke every image on it
**Source:** `cartora-site` (new repo `shah-51/cartora-site`).
**Who needs it:** purple-fox / Cartora, social-dashboard, shah-portfolio, any Vercel static site here.
The Cartora umbrella site now exists at `cartora.shah.works`, seeded from `social.shah.works`,
with a public brand page at `/brand` carrying the logo files. Vercel project `cartora-site`;
Cloudflare `A cartora.shah.works -> 76.76.21.21` DNS-only, matching every other Vercel subdomain
in the zone.
**The trap, which will recur:** with `cleanUrls` + `trailingSlash:false`, the live URL is `/brand`
with NO slash, so a browser resolves a relative `src="x.svg"` against the ROOT and 404s. Locally
`python -m http.server` redirects to `/brand/` WITH a slash, where the same relative path resolves
fine, so local testing passes and production breaks. **Asset refs on any such page must be
root-absolute.** Sharper second lesson: `curl` returning 200 for the asset proved the FILE was
fine and said nothing about whether the PAGE could reach it. The real check is
`[...document.images].filter(i => i.complete && i.naturalWidth === 0)` run in the live page.
**Also flagged:** `social.shah.works/product` and `/pricing` are still branded **Purple Fox
Social**, stale since the Cartora rename locked 2026-07-04. They were deliberately not copied
into the new site. They want fixing or deleting at the source.

### 2026-09-03 · CleverTap 6.0.5 outage bucket analysis: version 6.0.6 DOES report to CleverTap (
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management.

### 2026-09-03 - Finished brand assets (headshot, Cartora logo) now have ONE canonical home: `_shared/brand-assets/`
**Source:** `personal-os-workspace`.
**Who needs it:** Career Ops, Content Ops, outbound, shah-portfolio, purple-fox / Cartora, social-dashboard.
Stop re-cropping the headshot and re-exporting the logo per project. `_shared/brand-assets/`
holds sized, named exports for real destinations (avatar 648 down to 64, portrait 716 down to 256,
each PNG and JPG; Cartora mark and both wordmarks in SVG, PNG, JPG) plus a README saying which
size goes where. The WORKING sources stay put (`shah-portfolio/public/headshot.png`, the 864x1184
master; the purple-fox Cartora design folder) - edit there, re-export to the store.
Two gotchas that cost time and will cost it again: (1) the Cartora wordmark SVG asks for **Roboto**,
which is not installed here, so every baked PNG/JPG of the wordmark is silently Segoe UI Bold -
install Roboto before rendering a wordmark raster that matters. (2) On this machine **cairosvg
cannot run** (no libcairo DLL) and the `convert` on PATH is Windows' filesystem tool, NOT
ImageMagick. Use `sharp` (node, bundles librsvg) for SVG to raster; Pillow for photo crops.
Also: an avatar must be cropped tight (head ~50% of frame) - Gmail renders inbox avatars at ~32px,
where a head-and-shoulders crop is an unrecognisable smudge.

### 2026-09-02 · Data agent NEVER sends WhatsApp. Email and Telegram are built inside it; WhatsAp
**Source:** `data-agent`.
**Who needs it:** report-sender.

### 2026-09-02 · CleverTap backfill: profile data (phone identity, class, passing year) is upload
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management.

### 2026-09-02 · CleverTap 6.0.5 outage: push DELIVERY still works (344k sent 1 Sep, 670k reachab
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management.

### 2026-09-02 · Shikho Android app 6.0.5 (rolled out 25 Aug 2026) sends zero events to CleverTap
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management, paid-ads-analytics.

### 2026-09-02 · GA4 source takes startDate/endDate, NOT from/to (CleverTap takes from/to). Unkno
**Source:** `data-agent`.
**Who needs it:** paid-ads-analytics, shikho-organic-social-analytics, Customer Lifecycle Management.

### 2026-09-02 · Metabase public.Course_Subscriptions has taken zero writes since 2026-07-11 but 
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management, paid-ads-analytics.

### 2026-09-02 · CleverTap app_registration fires on ~0.7% of real app registrations (18 vs Metab
**Source:** `data-agent`.
**Who needs it:** Customer Lifecycle Management, paid-ads-analytics.
### 2026-09-02 · Meta Graph API v25 silently zeroed `reach` on every post (cross-pollinate auto-detect)
Source: `shikho-organic-social-analytics/CHANGELOG.md` (2026-07-02). Who needs it: any project pulling Meta
**page/post insights** (Graph API, not the Marketing API) — `shikho-organic-social-dashboard`,
`purple-fox-social-platform` (its Facebook pipeline was stood up on Shikho data the SAME day this was found,
per the changelog note "found while standing up the Purple Fox framework" — check whether its own fetch code
carries the same removed fields), `social-dashboard`, `data-agent`.
Graph API v25 removed `post_impressions_unique` and `post_impressions_organic_unique`. Meta fails the WHOLE
`/{post}/insights` metric-group call if any one requested metric is invalid (error 100) — so leaving those two
dead metrics in the reach group didn't just drop two numbers, it zeroed `reach` on every post fetched through
that call. Fix: drop the deprecated metrics from the request group; `reach` now comes from
`post_total_media_view_unique`. **Rule that travels:** a Graph API metric-group call is all-or-nothing — one
deprecated field poisons every metric requested alongside it, and the symptom (a legitimate-looking zero) gives
no hint which field caused it.

### 2026-09-02 · A trigger dedup guard closes a hole any multi-editor Apps Script project shares (cross-pollinate auto-detect)
Source: `shikho-sales-marketing-framework/CHANGELOG.md` (2026-08-16). Who needs it: `marketing-reporting-hub`
(also Google Apps Script) and any other GAS project where more than one Google account can run the trigger-install
function.
Root cause of duplicate kickoff emails: two editors had each independently run `installDigestTriggers()`.
Apps Script's `getProjectTriggers()` only returns the CURRENT user's own triggers, so neither editor could see
the other's set — both 3-trigger sets coexisted invisibly and both fired daily. Fix: a cross-user dedup guard
via `PropertiesService.getScriptProperties()` (shared store, unlike triggers) — the first trigger to fir

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — job-search relationship CRM — A hosted CRM for Shahriar's job search: roles, companies, contacts, outreach, and a **warm | `shah-51/career-crm` | 2026-09-02 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-09-03 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-09-06 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-06 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-09-02 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-09-02 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-09-02 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-09-02 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-09-02 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-09-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-09-02 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-09-02 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-09-06 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-09-07 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-09-02 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-09-02 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-09-03 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-09-07 |
| `May 2026/purple-fox/cartora-site` | CLAUDE.md, cartora-site (cartora.shah.works) — this file is stale; fix it in the session you notice. | `shah-51/cartora-site` | 2026-09-07 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-09-02 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-09-02 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-09-06 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-09-02 |
| `Shikho` | Shikho · domain entry — This is its own repo (`shah-51/shikho`), sitting at `D:\Shahriar\Claude\Shikho\` inside the | `shah-51/shikho` | 2026-09-07 |
| `Shikho/brand-design/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/brand-design/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/call-now-landing` | shikho-call-landing — the "Call Now" landing page — A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the | `shah-51/shikho-call-landing` | 2026-09-06 |
| `Shikho/campaigns/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/shikho` | 2026-08-17 |
| `Shikho/campaigns/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/data-agent` | Shikho/data-agent · redirect stub — This folder is a deprecated pointer. The project moved to its own repo on 2026-08-04. | `shah-51/shikho` | 2026-09-02 |
| `Shikho/measurement/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/shikho` | 2026-09-02 |
| `Shikho/measurement/Reporting/marketing-data-hub` | marketing-reporting-hub — cross-platform marketing reporting on Apps Script — A reusable reporting engine built on **Google Apps Script**: it pulls every market | `shah-51/marketing-reporting-hub` | 2026-09-02 |
| `Shikho/measurement/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/monthly-budget` | monthly-budget · V0 Paid Media Restructure — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-monthly-budget` | 2026-09-02 |
| `Shikho/operations/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/operations/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-08-05 |
| `Shikho/paid-ads-analytics` | CLAUDE.md — Shikho Paid Ads Analytics — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-analytics` | 2026-09-02 |
| `Shikho/paid-ads-analytics/google-ads-pipeline` | CLAUDE.md, Google Ads pipeline (sub-pipeline of paid-ads-analytics) — Daily Google Ads fetch. Pulls every campaign, ad group, ad, keyword, audience, and | `shah-51/shikho-google-ads-pipeline` | 2026-09-02 |
| `Shikho/paid-ads-analytics/meta-ads-pipeline` | CLAUDE.md · Shikho Meta Ads Pipeline — The heaviest sibling of `paid-ads-analytics`: the Meta Marketing API fetch, the | `shah-51/shikho-meta-ads-pipeline` | 2026-09-02 |
| `Shikho/paid-ads-analytics/paid-ads-dashboard` | CLAUDE.md — paid-ads-dashboard — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-paid-ads-dashboard` | 2026-09-02 |
| `Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-09-02 |
| `Shikho/shikho-organic-social-analytics/facebook-pipeline` | CLAUDE.md — facebook-pipeline — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hosted-run | `shah-51/shikho-organic-social-analytics` | 2026-09-02 |
| `Shikho/shikho-organic-social-analytics/organic-social-dashboard` | CLAUDE.md — organic-social-dashboard — Workflows run on the shared self-hosted DO runner (`self-hosted` label), not `ubuntu-latest`. See `_shared/infra-self-hos | `shah-51/shikho-organic-social-dashboard` | 2026-09-02 |
| `Shikho/shikho-paid-ads-private-artifacts` | Shikho Paid Ads · Private Artifacts (store) — This is an **artifact store**, not a code project. It holds sensitive paid-ads | `shah-51/shikho-paid-ads-private-artifacts` | 2026-09-02 |
| `Shikho/spreadsheet-to-telegram-report` | report-engine · config-driven reporting product (v0.2) — A shared rendering engine that pulls data from Metabase or Google Sheets, renders a styled | `shah-51/report-engine` | 2026-09-07 |
| `Shikho/strategy/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho` | 2026-06-10 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff` | Shikho 2026 Sales & Marketing Calendar — project entry — A live, **read-only** dashboard + framework explainer for Shikho's 2026 S&M calendar: | `shah-51/shikho-sales-marketing-framework` | 2026-09-02 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy` | shikho-sales-marketing-calendar (Vercel deploy) — The live, read-only Vercel site for Shikho's 2026 Sales & Marketing calendar | `shah-51/shikho-sales-marketing-calendar` | 2026-09-02 |
| `Shikho/strategy/sales and marketing calendar/agent_handoff/vercel_deploy_v2` | shikho-2026-calendar-v2 (Vercel deploy, COO build) — The v2 Vercel site for Shikho's 2026 S&M calendar (COO build, 2026-06-08), running in parallel | `shah-51/shikho-2026-calendar-v2` | 2026-09-02 |
| `Shikho/whatsapp-group-reporting` | Report Sender (WhatsApp group reporting) — Electron desktop app that renders reports locally and posts them to a WhatsApp group — on demand or | `shah-51/whatsapp-group-reporting` | 2026-09-07 |
| `data-agent` | data-agent · governed multi-source data + report + dispatch platform — push → deploy → verify), how everything is wired, and the invariants you must not break. | `shah-51/data-agent` | 2026-09-07 |
| `data-agent/showcase` | data-agent/showcase · sales and demo materials — Presentation and recording assets for pitching the data-agent product. Not deployed; used locally for | `shah-51/data-agent` | 2026-09-02 |
| `data-agent/tools/creative_intel` | creative_intel — Context for any session working here — It covers all filter specs, known traps, use case playbooks, and config values. | `shah-51/data-agent` | 2026-09-06 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
