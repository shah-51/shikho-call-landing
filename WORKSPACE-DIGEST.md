# Workspace Digest (offline hub snapshot)

> **Generated 2026-08-25 by `personal-os-workspace/scripts/distribute-hub-digest.py`.**
> This is a COMMITTED SNAPSHOT of the cross-project knowledge hub, copied into this repo
> so a standalone clone still sees what's happening across the workspace. Do NOT hand-edit.
> The live hub lives in `shah-51/personal-os-workspace/_shared/` (cross-relevance.md,
> learnings-index.md, WORKSPACE-MAP.md). When working from the full workspace, prefer those.

## Findings that travel here (cross-relevance)

### 2026-08-25 · One deploy failed SIX times — the anatomy, and why documentation could not stop it
**Source:** whatsapp-group-reporting (relay deploy). **Who needs it:** **every project with a droplet/remote deploy** — data-agent, atlas-pdf, purple-fox, report-sender-dashboard — and **every session that writes a runbook and expects it to be followed**.
A single one-file deploy took **six attempts**. Anatomy, because each failure was a *different* mechanism and only the last two were the "obvious" kind:
1–3. **Droplet commands and a local `scp` pasted into the wrong shell.** A deploy spans two machines and the failure is **asymmetric**: droplet commands in PowerShell fail loudly (`chown` not recognised), but a local `scp D:\...` inside an SSH session fails **quietly** — scp uploads nothing, then the follow-up `cp /tmp/file …` succeeds against a **stale file from an earlier upload**, so chown + restart + health all go green while the old code runs.
4. **`scp` with a Windows drive path, locally.** OpenSSH parses any `X:` prefix as `host:path`, so `D:\a\b` becomes host `D` (`stat local "D"`). Quoting does not help — the ambiguity is in scp's own argument parsing. Only fix: `cd` first, pass **bare filenames**.
5. **A false STALE from the verifier itself.** The check's `curl -d '{"jsonrpc":…}'` crossed PowerShell quoting → ssh → remote shell; the embedded double quotes did not survive, curl posted garbage, and grep searched an error response. Every system fact contradicted the verdict (file on disk correct, service active, right PID on the port, clean log) and it nearly triggered an hour hunting a nonexistent orphan process. **Tell: the quote-free `curl localhost/health` in the same script worked.**
6. Success — after the script was made to refuse the wrong shell, abort on scp failure, and base64 its remote payload.
**Three rules worth taking wholesale:**
- **Never verify a deploy by whether the service restarted.** A stale file restarts just as happily. End every deploy by grepping the **live** service for a string that exists ONLY in the new code.
- **When a check disagrees with the system, test the check before acting on its verdict.** A false negative is not the safe direction — it burns time *and* destroys trust in every future green result.
- **Never send quoted JSON through more than one shell.** Base64 it (`[A-Za-z0-9+/=]` survives every layer) or write it to a file.
**The meta-finding — this is the part that generalises furthest:** the runbook's Rule 0 already documented failures 1–3, and it was violated three times **by the person who wrote it, in the same session, with the doc in context**. Prose did not prevent recurrence. What did: `scripts/deploy-relay.ps1`, which **refuses** to run inside SSH, aborts if scp fails, and verifies against new code. One tool, zero recurrences. See the entry below on escalating a recurred learning to enforcement.

### 2026-08-25 · A learning that RECURS needs ENFORCEMENT, not a third write-up (with the hook to do it)
**Source:** workspace DECISIONS 2026-08-25 + `~/.claude/hooks/known-traps.ps1`. **Who needs it:** **every project and every session** — this is a workspace-wide discipline, not a domain finding.
**The measurement that forced it:** in one session, three separate learnings that were *already written down* were walked into again — the Windows cp1252 console trap (recorded in THIS file from an earlier project), the PowerShell ANSI-read trap (twice), and the deploy runbook's own Rule 0 (three times, by its author). `learnings-index.md` holds 6,600+ entries; nobody reads 6,600 entries. **Tier 1 is an archive, not a defence.**
**Three tiers, weakest to strongest** — the same shape as teaching an AI agent, because it is the same problem of instructing a fallible actor: (1) **write it down** — needs someone to look at the right moment; (2) **inject it contextually** — a hook that fires *on the risky action* and states the fix; (3) **make it impossible** — a tool that refuses the wrong thing.
**The rule now in the global CLAUDE.md, so it loads in every session:** the FIRST occurrence gets a LEARNINGS entry. The **SECOND occurrence is evidence that entry does not work**, so it escalates to tier 2 or 3. *"Write it up again, more clearly"* is not a response to recurrence — it is the thing that already failed.
**Shipped:** `~/.claude/hooks/known-traps.ps1` (PreToolUse on Bash|PowerShell) blocks four recurred, mechanically-detectable traps with the fix in the message — scp with a Windows drive path, an `.env` scp'd without CRLF normalisation, `Get-Content -Raw` without `-Encoding`, quoted JSON inside an ssh command. **Add a trap only when it has actually recurred AND is certain to fail.**
**Precision over recall, learned the hard way:** the hook **false-positived on the very commit that introduced it**, because the commit message *quoted* the trap it describes. A guard that blocks legitimate work teaches people to route around it, which kills every future block. Fixed by requiring command position and skipping text-carrying commands (commit/tag messages, heredoc bodies); 10/10 tests including both false-positive regressions.
**The honest limit:** this catches *mechanically detectable* mistakes only. Judgement errors — wrong architecture, misread requirement — cannot be pattern-matched and stay in prose. Do not fake enforcement with fuzzy patterns; that is how a guard becomes noise and then gets disabled.

### 2026-08-25 · Instructions alone don't work — agents also need a MIRROR and FEEDBACK (three tiers)
**Source:** whatsapp-group-reporting (relay `get_report` + `designCheck`). **Who needs it:** **every MCP server we ship** — data-agent above all (21 connectors, many users on their own AI accounts), plus cartora, purple-fox, any future `/mcp`.
Follows the entry below (teach from the server). **Three rounds of improving the `instructions` text still produced bad output**, and the reason is structural, not a wording problem: instructions are advice given *before* the mistake, and the agent had **no way to see its own output**. Concretely: an agent designed two rich reports in chat, published templates containing none of the design, and had no tool that could show it what it had actually published — `list_my_reports` returned only id/title/kind. It was writing blind and could not self-diagnose even in principle.
**Three ascending tiers of teaching an agent — most systems ship only tier 1, then blame the model:**
1. **Tell it** — `instructions` + tool descriptions. Advice, pre-mistake, and a client *may* ignore it.
2. **Let it look** — an inspection tool (here `get_report(id)`, returning the stored artifact). This turns "revise X" from *rewrite from memory* into *edit the real thing* — and an agent's memory is of what it produced **in chat**, which is demonstrably not what it wrote to your system.
3. **Tell it when it is wrong** — a check embedded in the tool's own **result** (here `designCheck`, reporting factually whether a template contains a repeating section / inline SVG / data-driven styles). Strongest channel available: a model **cannot skip reading the result of its own call**, and the feedback lands while the author is still present to act.
**Design rules that made the check work rather than get ignored:** it never blocks (a legitimate output may trip it, and a rule routinely overridden teaches agents to ignore the whole category); it states *checkable facts* then the remedy, never a judgement like "this is poor" that an agent cannot act on; and it fires at **write time**, not at read/send time when the artifact is already in front of stakeholders.
**Corollary worth adopting everywhere:** if a mistake is mechanically detectable, detect it and say so in the result of the call that made it. Do not put it in a README and hope. Full standard, incl. six rules for changing such a contract: `whatsapp-group-reporting/docs/AGENT-CONTRACT.md`.

### 2026-08-25 · Teach client agents FROM the MCP server (`instructions`)

## Recent across all projects (last 14 days)

_(nothing in window)_

## Project & repo map

| Project (path) | What it is | GitHub repo | Last touched |
|---|---|---|---|
| `April 2026/Career Ops/career-crm` | career-crm — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/career-crm` | 2026-08-25 |
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
| `Shikho/Reporting/marketing-data-hub` | marketing-reporting-hub — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/marketing-reporting-hub` | 2026-08-25 |
| `Shikho/Shikho Design System` | Shikho Design System v1.0 — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-07-23 |
| `Shikho/Shikho SEO_AEO_GEO` | Shikho SEO / AEO / GEO · 90-day search dominance plan — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/Uninstall Feedback Loop` | Uninstall Feedback Loop · March 2026 churn analysis — This project is one of many in Shahriar's workspace. A shared **knowledge hub** tracks | `shah-51/personal-os-workspace` | 2026-06-10 |
| `Shikho/call-now-landing` | shikho-call-landing — <!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers) | `shah-51/shikho-call-landing` | 2026-08-25 |
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
