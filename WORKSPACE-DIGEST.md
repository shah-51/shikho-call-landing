# Workspace Digest (offline hub snapshot)

> **Generated 2026-08-25 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-08-24 · An MCP endpoint without OAuth cannot be added from Claude.ai / ChatGPT / Gemini — a bearer alone is not enough
**Source:** whatsapp-group-reporting (relay OAuth adapter, `send.shah.works`). **Who needs it:** **every MCP server we ship or will ship** — data-agent (already has it), cartora, purple-fox, any future `/mcp` endpoint intended for non-technical users.
Claude.ai's Custom Connector does not offer a "paste a bearer token" field. Given an MCP URL it immediately attempts **OAuth 2.0 Dynamic Client Registration**, and a bearer-only server fails with *"Couldn't register with <server>'s sign-in service."* Same for ChatGPT connectors and Gemini Enterprise's Add-MCP-Server. **Claude Code (CLI) is the exception** — it takes a bearer directly, which is why an endpoint can appear "working" to the builder while being unusable by teammates.
**The fix is small and is now a proven, twice-executed port (~40 min):** you do NOT need a real identity provider. Wrap the identity token you already have. Six endpoints: `/.well-known/oauth-authorization-server` (RFC 8414), `/.well-known/oauth-protected-resource`, `POST /register` (RFC 7591, return a public PKCE client), `GET|POST /authorize` (a consent page that asks the user to paste their existing token), `POST /token` (authorization_code + PKCE, plus refresh with rotation). Add `WWW-Authenticate: Bearer resource_metadata="<base>/.well-known/oauth-protected-resource"` to every 401 — that header is the breadcrumb clients follow. Keep `initialize` and `tools/list` open so clients can introspect before authenticating.
**Design rules worth copying verbatim:** (1) **OAuth wraps the existing token identity** — issue access+refresh *bound to that token's hash*, and re-resolve the underlying token on every request, so there is exactly ONE kill switch and revocation stays instant. (2) **Persist access+refresh, hashed** — in-memory state means every deploy/restart forces every client to reconnect (data-agent learned this the hard way 2026-08-03); 30d access / 180d refresh with rotation keeps connections alive indefinitely. (3) Auth codes can stay in-memory (5-min window). (4) **The public base URL env var must be THIS instance's own origin** — a second instance on its own domain that inherits production's base URL redirects users to the wrong consent page, where their token doesn't exist (data-agent's demo brand hit exactly this).
Reference implementation + port guide: `whatsapp-group-reporting/docs/OAUTH.md`; source `relay/oauth.mjs`.

### 2026-08-24 · Windows-authored `.env` files silently break Linux daemons (CRLF) — normalize at the boundary
**Source:** whatsapp-group-reporting (admin dashboard deploy). **Who needs it:** **anything scp'd from this Windows workstation to the droplet or any Linux host** — data-agent, atlas-pdf, purple-fox, relay, dashboard, future workers. Also any CI that authors config on Windows.
An admin password set in a Windows-authored `dashboard.env` never matched, with **no error logged** — the login just kept saying "Wrong password". Cause: CRLF line endings. systemd's `EnvironmentFile=` reads the trailing `\r` as **part of the value**, so `PASSWORD=hunter2` becomes `"hunter2\r"`. Constant-time comparisons short-circuit on the length difference, so it fails before any character comparison — nothing to see in logs.
**Rule: `sed -i 's/\r$//' <file>` after ANY scp of a config/env/secret file from Windows.** Verify with `cat -A <file>` — clean lines end `$`, dirty lines end `^M$`. This applies to every value a Linux daemon reads, not just passwords: connection strings, API keys, feature flags, base URLs. A URL with a trailing `\r` produces especially baffling failures (DNS/TLS errors on a hostname that looks correct).

### 2026-08-24 · `systemctl restart` returns before the app is listening — smoke-test localhost BEFORE the public origin
**Source:** whatsapp-group-reporting (relay OAuth deploy). **Who needs it:** every droplet service deploy — refines the 2026-08-23 "active reads as running" entry with a *different* mechanism.
After a deploy, `systemctl status` said `active (running)` yet the public HTTPS origin returned **502**. Not a crash-loop this time and not a port collision: systemd reports `active` when the process is **spawned**, but a Neon/DB-backed Node service needs ~2 more seconds to open its pool and bind. A `curl` at t+1s reached Caddy, Caddy proxied to a not-yet-listening upstream, and returned a 502 that reads exactly like a code bug.
**Rule: poll `curl -sf localhost:<port>/health` in a retry loop until it returns 200, and only then test the public origin.** Local-first also isolates the failure domain — local 200 + public 502 means Caddy/DNS, not your code. Never conclude "the deploy broke it" from a single early public curl.
**Updated droplet port map:** 8787=atlas-pdf · 8788=data-agent · 8789=demo-data-agent · 8790=report-sender-relay · **8791=report-sender-dashboard** · 8792+ free. Full standard: `whatsapp-group-reporting/docs/DEPLOY-RUNBOOK.md` (portable to any service here).

### 2026-08-24 · Report Sender and Data Agent are TWO standalone products that never merge
**Source:** whatsapp-group-reporting (product architecture, confirmed by owner). **Who needs it:** data-agent, and any session tempted to fold one into the other because they share a droplet, a Neon account, and an OAuth pattern.
**Report Sender** = delivery (renders locally, sends to WhatsApp, desktop app + relay catalog + access model). **Data Agent** = data and governance (MCP/REST query layer over 21 connectors). They share infrastructure patterns deliberately — the OAuth adapter was ported one to the other — but they are separately packaged, separately sold, and separately versioned. Report Sender *optionally consumes* Data Agent as one connector among several (Metabase, Sheets, custom REST, custom MCP). Do not propose unifying their repos, their Neon projects, or their access models.

### 2026-08-24 · Electron desktop-app traps: `files:["**/*"]` leaks `.claude/`, silent whenReady death, safeStorage lives in userData
**Source:** whatsapp-group-reporting (Report Sender). **Who needs it:** any Electron/electron-builder desktop app now or later (report-sender is the only one today).
Three hard-won gotchas from a "won't open after update" incident: (1) **electron-builder `files:["**/*"]` packages `.claude/`** — a `.claude/worktrees/<id>/` git worktree (a full duplicate app tree) got baked into the asar (+43MB, contamination). Exclude `.claude/**`, `.git/**`, `**/*.map`, docs. (2) **A throw in any `app.whenReady` step silently kills the app** (no window/tray/log → clean exit(0), undiagnosable). Wrap EVERY startup step in try/catch + add `uncaughtException`/`unhandledRejection`/`render-process-gone` breadcrumbs to a log file. (3) **`safeStorage` on Windows keeps its key in `Local State` inside userData (`%APPDATA%\<app>\`), NOT the install dir** — so app updates never wipe stored secrets or `config.json`; verified by the key file surviving ~6 reinstalls. Bare `npx electron` uses a different app identity/key — never run it against a real app's userData while debugging (it orphans secrets and gives false "decrypt failed"). Also: leftover dev `electron.exe` holds the single-instance lock, making the packaged app quit on launch — kill all electron/app PIDs between launches.

### 2026-08-23 · Rendering AI/user-authored HTML for others = logic-less template + JS-disabled render (both, not either)
**Source:** whatsapp-group-reporting (live team reports / Option B). **Who needs it:** anything that renders HTML authored by an AI or another user and shows it to a third party — data-agent report HTML, Cartora/purple-fox generated pages, any "publish a report/page" feature.
When one person's HTML template runs on another person's machine (or server), treat it as untrusted. Two independent layers: (1) fill data with a **logic-less** template engine (a Mustache subset — substitution only, no eval/Function/express

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/.claude/worktrees/agitated-stonebraker-c51dbf` | Career-Ops -- AI Job Search Pipeline — This system was built and used by [santifer](https://santifer.io) to evaluate 740+ job offers, generate 100+ tailored CVs | `shah-51/career-ops-private` | 2026-06-23 |
| `April 2026/Career Ops/.claude/worktrees/quizzical-bhabha-7b335c` | Career-Ops -- AI Job Search Pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/career-ops-private` | 2026-08-02 |
| `April 2026/Career Ops/career-crm` | career-crm — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/career-crm` | 2026-08-24 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-08-24 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-08-24 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-08-24 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-08-24 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-08-24 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-08-24 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-08-24 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-08-24 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-08-24 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-08-24 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-08-24 |
| `March 2026/shah-portfolio/.claude/worktrees/published-case-studies-posts-cbc500` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-08-23 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-08-25 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-08-24 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-08-24 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-08-24 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-08-24 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-08-24 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-08-24 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-08-24 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-08-24 |
| `May 2026/purple-fox/worktrees/demo-deepdive` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-07-13 |
| `Shikho` | Shikho · domain entry — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-25 |
| `Shikho/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/personal-os-workspace` | 2026-08-17 |
| `Shikho/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/personal-os-workspace` | 2026-08-24 |
| `Shikho/Reporting/marketing-data-hub` | marketing-reporting-hub — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/marketing-reporting-hub` | 2026-08-24 |
| `Shikho/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-23 |
| `Shikho/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/call-now-landing` | shikho-call-landing — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/shikho-call-landing` | 2026-08-24 |
| `Shikho/claude-ads-skill` | Claude Ads: Paid Advertising Audit & Optimization Skill — This repository contains **Claude Ads**, a Tier 4 Claude Code skill for comprehensive | `AgriciDaniel/claude-ads` | 2026-05-02 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc` | Workspace root · Shahriar's Claude work — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho` | Shikho · domain entry — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
