# Workspace Digest (offline hub snapshot)

> **Generated 2026-08-23 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-08-23 · Rendering AI/user-authored HTML for others = logic-less template + JS-disabled render (both, not either)
**Source:** whatsapp-group-reporting (live team reports / Option B). **Who needs it:** anything that renders HTML authored by an AI or another user and shows it to a third party — data-agent report HTML, Cartora/purple-fox generated pages, any "publish a report/page" feature.
When one person's HTML template runs on another person's machine (or server), treat it as untrusted. Two independent layers: (1) fill data with a **logic-less** template engine (a Mustache subset — substitution only, no eval/Function/expressions); a data value that looks like `{{tag}}` must print inertly, not re-parse. (2) Render with **JavaScript disabled** (`page.setJavaScriptEnabled(false)` in Puppeteer) so a smuggled `<script>` can't run or exfiltrate. HTML-escape by default; require an explicit `{{{raw}}}` to opt into raw HTML. Keep the data pull on the consumer's side with THEIR credentials so the publishing service never holds data or secrets. A ~90-line engine (render/lib/recipe.mjs) covers it — no template-lib dependency needed.

### 2026-08-23 · On a shared droplet, a crash-looping service reads as "active" — verify by its OWN health payload + pick a free port
**Source:** whatsapp-group-reporting (relay deploy to send.shah.works). **Who needs it:** anything deployed on the shared droplet (152.42.212.147) — data-agent, atlas-pdf, purple-fox, any future systemd service.
A new systemd service showed `active (running)`, but the port was already bound by another service: our process crash-looped on `EADDRINUSE` while `Restart=always` flapped, and the momentary "active" was a restart window. `curl localhost:<port>/health` returned the INCUMBENT's payload, not ours. **Rules:** (1) before picking a port, enumerate what's bound — `ss -ltnp`, or read `/etc/caddy/Caddyfile` (every `reverse_proxy 127.0.0.1:<port>` line is a taken port). Current map: 8787=atlas-pdf, 8788=data-agent, 8789=demo-data-agent, **8790=report-sender-relay**. (2) Verify a service by hitting its own health JSON and reading `journalctl -u <svc>`, never trust systemd's status word alone.

### 2026-08-23 · Cloudflare DNS records for shah.works can be created headless via the vault API token (grey-cloud for Caddy TLS)
**Source:** whatsapp-group-reporting (send.shah.works A record). **Who needs it:** any project adding a shah.works subdomain (new droplet service, portfolio, Pages) — skip the dashboard.
`CLOUDFLARE_API_TOKEN` is in `~/.secrets/career-ops-secrets.env`. Flow: `GET /user/tokens/verify` → `GET /zones?name=shah.works` (zone id) → read a sibling record to copy its settings → `POST /zones/<id>/dns_records`. **Key detail:** the droplet-backed `*.shah.works` records are **proxied=false (DNS-only / grey cloud)** because Caddy does its own auto-TLS (Let's Encrypt) and needs the origin reachable directly — match that, don't turn on the orange cloud. Also: appending a Caddy vhost twice → `caddy validate` fails with "ambiguous site definition"; rewrite the whole Caddyfile from known-good content and `validate && reload` so a bad config never loads.

### 2026-08-19 · Ad-platform "yesterday" is the last complete day in the ACCOUNT timezone — not the server's date
**Source:** report-engine / whatsapp-group-reporting (Ad Spend report). **Who needs it:** any paid-ads reporting (paid-ads-analytics, data-agent ad-spend, Governance decks) computing "yesterday" or "MTD-till-yesterday".
Meta `datePreset:yesterday` and Google `DURING YESTERDAY` both returned 2026-08-18 while the workstation clock said today was 08-18 — the ad platforms were a day ahead (account TZ). A hardcoded MTD (1st–17th) then undercounts by a day and a cross-window "total" mixes days. **Rule:** resolve "yesterday" from the platform's OWN returned date (Google `segments.date` / Meta `date_start`), then derive monthStart / 7-day-start / MTD from THAT date and pass explicit ranges to every platform so all windows align. Never assume the ad platform's day equals the server's day.

### 2026-08-19 · Never bundle a data-source token into a PUBLIC release — use a per-install encrypted connection
**Source:** whatsapp-group-reporting (Data Agent connection, v0.7.0). **Who needs it:** any distributed app/installer shipping to users that needs API keys.
The release feed is public, so a token baked into the build leaks. Pattern (mirrors the Metabase key): store the token per-install via the OS secure store (Electron `safeStorage` → an encrypted file in userData), entered once in a Connections UI; an env var wins for dev. Mint it **least-privilege** (read-only, scoped to only the sources the report needs) so a leaked install token can't send or reach other data. Connectors take creds **as params** (from the source object), never module-level env, so one connector works with per-install creds.

### 2026-08-18 · Window every rate to the report's own time frame — a period metric on a "today so far" view lies
**Source:** report suite (funnel live). **Who needs it:** any analytics/reporting/dashboard project mixing intraday and period views.
A conversion/ratio is a settled-period quantity. Shown on a "so far today" report it misleads: stage 2 lags stage 1 within a day (e.g. an activation step that lands in the evening reads ~0% conversion at 6pm), and a same-day numerator/denominator are often two independent volumes, not a true cohort rate. **Rule:** window each rate to the report's own frame (live=today, daily=MTD, monthly=the month); if a metric has no honest meaning in that window, omit it rather than show a technically-correct-but-misleading number.

### 2026-08-18 · A \n-anchored string replace silently fails on CRLF files — doc entries vanish under a "clean" commit
**Source:** `spreadsheet-to-telegram-report` / `whatsapp-group-reporting` (report suite reskin). **Who needs it:** ANY project that edits CHANGELOG/DECISIONS/LEARNINGS (or any markdown) with a hand-rolled node/python script on Windows — nearly all of them, given the docs-enforcement rule.
A scripted `str.replace("# Changelog\n\n", header+entry)` matches nothing when the file uses CRLF (`\r\n`): replace returns the string unchanged, `writeFileSync` writes identical bytes, `git add`/commit see no diff, so the commit "succeeds" while the entry never landed. Six doc entries were lost this way in one session before it was caught. **Rule:** when scripting doc edits, match with a CRLF-tolerant regex (`/\r?\n/`) or insert before the first `## ` entry heading; PRESERVE the file's detected newline on write; and ALWAYS grep the file back for the new text before trusting the commit. The `capture.py` helper avoids this — hand-rolled edits do not.

### 2026-08-18 · A product "read-only" / analyst tier must be gated by ROLE, not by a single "canSend" flag
**Source:** `data-agent`. **Who needs it:** any multi-user tool with tiered access — outbound, Cartora,
purple-fox, any MCP/product with per-user tokens.
A per-user record had `sources[]` (query scope), `canSend`+`destinations[]` (dispatch), and `role`. "Read-only"
was assumed = `canSend:false` — but report **create/run/delete** were gated only by *ownership*, so a
`canSend:false` user could still write server-side report specs. `canSend` gates dispatch, not object
management — different powers. And the role NAME lied: `analyst` was minted send-capable by default
(`canSend:true, destinations:["*"]`), so the "read" tier could actually send anywhere. **Rule:** define
"query-only" as ONE explicit predicate over the role (`isQueryOnly`/`assertCanManageReports`) and assert it
at the top of EVERY mutating/dispatching handler — never infer read-only from a scatter of booleans, and
never trust a role's name over its enforcement. Bonus: keep the data path itself read-only (connectors only
`SELECT`) so a privileged DB role behind them can never become a user-reachable write path.

### 2026-08-18 · Neon least-privilege: API/console roles are ALWAYS `neon

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/.claude/worktrees/agitated-stonebraker-c51dbf` | Career-Ops -- AI Job Search Pipeline — This system was built and used by [santifer](https://santifer.io) to evaluate 740+ job offers, generate 100+ tailored CVs | `shah-51/career-ops-private` | 2026-06-23 |
| `April 2026/Career Ops/.claude/worktrees/quizzical-bhabha-7b335c` | Career-Ops -- AI Job Search Pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/career-ops-private` | 2026-08-02 |
| `August 2026/cartora-outbound` | Cartora Outbound Platform — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/cartora-outbound` | 2026-08-09 |
| `June 2026` | June 2026 workspace root — pnpm monorepo housing the **Ad Ops OS** suite — a performance media operating system for B2C teams running $30K-150K/mo on Google and | `shah-51/personal-os-workspace` | 2026-08-03 |
| `June 2026/adops-os` | Ad Ops OS · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-08-03 |
| `June 2026/adops-os-budget` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-budget` | 2026-08-03 |
| `June 2026/adops-os-core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-core` | 2026-07-04 |
| `June 2026/adops-os-optimize` | adops-os-optimize · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-optimize` | 2026-07-04 |
| `June 2026/adops-os-platform-sync` | adops-os-platform-sync · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-platform-sync` | 2026-07-04 |
| `June 2026/adops-os-reports` | adops-os-reports · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os-reports` | 2026-07-04 |
| `June 2026/adops-os/apps/app` | adops-os-budget · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/adops-os/packages/core` | adops-os-core · project context — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/adops-os` | 2026-06-06 |
| `June 2026/mobile-ready-doc` | PhoneReadyDoc — Free, 100% client-side web tool that converts a desktop-built PDF into one that reads cleanly on a phone by trimming whitespace and magnifying t | `shah-51/personal-os-workspace` | 2026-07-19 |
| `June 2026/mobile-ready-doc/files_unzipped/phonereadydoc/phonereadydoc` | phonereadydoc — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/phonereadydoc` | 2026-07-04 |
| `Luna Bella` | Luna Bella — Product catalog research and data project: 67 cosmetic/personal-care SKUs built from 153 phone photos. | `shah-51/personal-os-workspace` | 2026-07-19 |
| `March 2026/shah-portfolio` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-08-23 |
| `March 2026/shah-portfolio/.claude/worktrees/published-case-studies-posts-cbc500` | CLAUDE.md, shah-portfolio (shah.works) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shah-portfolio-site` | 2026-08-23 |
| `May 2026/Adjust Replacement - BigQuery` | Adjust Replacement · BigQuery R&D — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-03 |
| `May 2026/Content Ops` | Content Ops — Shahriar's daily idea pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/content-ops` | 2026-08-23 |
| `May 2026/app-review-manager` | App Review Manager — AI-powered Play Store and App Store review monitoring, sentiment classification, reply drafting, and weekly insight reports, sold as a prod | `shah-51/app-review-manager` | 2026-07-04 |
| `May 2026/outbound` | Outbound · cold outreach pipeline — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/outbound` | 2026-08-09 |
| `May 2026/playstore_review` | Play Store Review Analysis · Shikho + EdTech competitors — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/shikho-playstore-review` | 2026-07-23 |
| `May 2026/purple-fox` | Cartora Social — product plan folder — Planning docs, specs, and build briefs for **Cartora Social** — a reporting-first organic-social analytics SaaS (FB + IG, | `shah-51/personal-os-workspace` | 2026-08-16 |
| `May 2026/purple-fox/platform` | CLAUDE.md - purple-fox-social-platform — Part of Shahriar's personal-OS workspace. Skim `_shared/cross-relevance.md` and the | `shah-51/purple-fox-social-platform` | 2026-07-04 |
| `May 2026/purple-fox/social-dashboard` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-08-16 |
| `May 2026/purple-fox/social-site` | CLAUDE.md - social-site (Purple Fox Social product website) — The static marketing website for **Social**, a product of Purple Fox Communications. A 3-page vani | `shah-51/purple-fox-social-site` | 2026-07-13 |
| `May 2026/purple-fox/staging-work` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `Shikho-Edtech/social-dashboard-staging` | 2026-07-12 |
| `May 2026/purple-fox/worktrees/demo-deepdive` | social-dashboard · Purple Fox Social product (dashboard) — The interactive analytics dashboard for the Purple Fox Social product. A Next.js 14 | `shah-51/social-dashboard` | 2026-07-13 |
| `Shikho` | Shikho · domain entry — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-23 |
| `Shikho/Brand Guidelines` | Brand Guidelines · Shikho v1 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Customer Lifecycle Management` | Customer Lifecycle Management · Shikho — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Governance` | Governance · Monthly Digital Marketing decks — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/Meta Ads MCP Article` | Meta Ads MCP Article (published) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/MyGP HSC28 Acquisition` | MyGP HSC'28 Acquisition — A paid-media campaign against a **partner-supplied phone list**: 231,000 SSC'26 candidates who | `shah-51/personal-os-workspace` | 2026-08-17 |
| `Shikho/Paid Ads Video Performance` | Paid Ads Video Performance · Q1'26 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-03 |
| `Shikho/Reporting` | Reporting · GPA5 WhatsApp report bot — Automates a previously-manual ritual: a formatted report lives in a Google Sheet, | `shah-51/personal-os-workspace` | 2026-08-18 |
| `Shikho/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-23 |
| `Shikho/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
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
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |
| `Shikho/data-agent/.claude/worktrees/hello-5ec8dc/Shikho/shikho-organic-social-analytics` | CLAUDE.md — shikho-organic-social-analytics (master) — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-08-05 |

---
_To refresh every repo's snapshot: run `python scripts/distribute-hub-digest.py` from the
workspace, then commit+push touched repos. Maintained by the daily `cross-pollinate` routine._
