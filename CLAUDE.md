# shikho-call-landing — the "Call Now" landing page

A one-purpose mobile page for **CleverTap "call now" push notifications**. A user taps the
notification, lands here, taps the big button, and their dialer opens with the Shikho hotline
(**16780**) pre-filled. Live at `https://shikho-call.vercel.app`. Repo: `shah-51/shikho-call-landing`.

Deliberately tiny: **no framework, no build step, no dependencies.** `index.html` is the whole page —
markup, CSS, and a small click-tracking script in one file. Double-click it to preview; there is
nothing to run.

## Files

| File | What it is |
|---|---|
| **`index.html`** | ⭐ the entire page — the only file you edit |
| `shikho-logo.svg` | the logo at the top |
| `hs-500 / hs-600 / hs-700.woff2` | self-hosted Hind Siliguri (Bangla). **Keep them.** |
| `favicon.ico`, `vercel.json` | tab icon; caching rules for fonts/logo |

The non-HTML files work by filename — **don't rename them.**

## The one thing not to undo

**Fonts are self-hosted on purpose.** Render-blocking Google Fonts made this page take "forever to
load" on Bangladeshi mobile networks — the audience is users tapping a push notification on a phone,
often on a slow connection, and a slow page here means a lost call. Do not "simplify" back to a
Google Fonts `<link>`. Full detail in `LEARNINGS.md` (2026-08-03).

Also from LEARNINGS: the CSS `scale` property is used instead of `transform: scale()` so `transform`
stays free for press feedback on the button.

## When working here

- The audience is **mobile, Bangla-first, on a slow network**. Every change is judged on how fast the
  page paints and how obvious the call button is — not on visual sophistication.
- **Changing the phone number means replacing `16780` in several places**: two `tel:16780` links, the
  button label, the hint line, and the tracking event. Grep for it; missing one leaves a dead button.
- This is a Shikho-domain deliverable: Shikho Design System tokens apply (indigo, rounded,
  Bangla-first), and the confidentiality posture in `Shikho/CLAUDE.md` applies to anything added.
- Non-technical edits are expected to happen by pasting `index.html` into an LLM and pasting the
  result back (README documents that flow) — so **keep the file self-contained and readable**. Do not
  split it into partials or introduce a build step.

<!-- hub-pointer --> (auto-managed by distribute-hub-digest.py; do not edit between markers)
## Cross-project knowledge hub
This repo belongs to Shahriar's personal-OS workspace. If you opened it standalone (no
`_shared/` in a parent folder), read **`WORKSPACE-DIGEST.md`** in this repo root for a recent
snapshot of cross-project findings. The live hub is `shah-51/personal-os-workspace/_shared/`
(cross-relevance.md, learnings-index.md, WORKSPACE-MAP.md). Record notable findings in this
repo's CHANGELOG / DECISIONS / LEARNINGS so the next hub refresh carries them onward.
<!-- /hub-pointer -->
