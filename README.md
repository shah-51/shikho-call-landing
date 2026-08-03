# Shikho — "Call Now" landing page

A one-purpose mobile page for **CleverTap "call now" push notifications**. A user
taps the notification, lands here, taps the big button, and their phone dialer
opens with the Shikho hotline (**16780**) pre-filled.

**Live:** https://shikho-call.vercel.app

![what it looks like](https://shikho-call.vercel.app) <!-- open the URL to preview -->

---

## What's in this repo

| File | What it is |
|---|---|
| **`index.html`** | ⭐ **The whole page** — HTML + CSS + a tiny click-tracking script. This is the only file you edit. |
| `shikho-logo.svg` | The Shikho logo at the top |
| `hs-500 / hs-600 / hs-700.woff2` | Self-hosted Bangla fonts (Hind Siliguri). Keep them — they make the page load fast on mobile. |
| `favicon.ico` | Browser-tab icon |
| `vercel.json` | Caching rules for the fonts/logo |
| `LEARNINGS.md` | Build notes (why fonts are self-hosted, etc.) |

**You only ever edit `index.html`.** The other files just sit next to it and keep
working by their filenames — don't rename them.

---

## How to modify it with ChatGPT

1. Open `index.html`, copy **all** of it.
2. Paste into ChatGPT with your request, e.g.
   *"This is my landing page. Change the headline to X and make the button green.
   Give me back the full file."*
3. Copy ChatGPT's result back into `index.html`, save.
4. Preview by **double-clicking `index.html`** (it opens in your browser — no server needed).

Common quick edits (all in `index.html`):
- **Phone number:** replace every `16780` (two `tel:16780` links, the button, the hint line, the tracking event).
- **Headline:** the `<h1>` inside `.card`.
- **Badge line:** the `<span class="badge">` text.
- **Sub-line / trust row:** the `.sub` paragraph and `.assure` block.

---

## How to make it live (pick one)

**A. Netlify Drop (easiest, no account setup):** go to https://app.netlify.com/drop
and drag this whole folder onto the page. You get a live URL in ~30 seconds.
Re-drag to update.

**B. Vercel (what the original uses):** create a free account at vercel.com →
"Add New Project" → import this GitHub repo → Deploy. Every `git push` auto-updates
the live site.

**C. GitHub Pages:** repo Settings → Pages → deploy from the `main` branch → root.
Live at `https://shah-51.github.io/<repo-name>/`.

All three host plain static files — there is **no build step**. What you see in the
folder is exactly what goes live.

---

## Notes

- A guarded `call_now_click` event fires on tap (pushes to `dataLayer` / `gtag` if
  present). Add a GTM or gtag snippet later and click-tracking works with zero code change.
- Fonts are **self-hosted on purpose** — loading them from Google was slow on
  Bangladesh mobile networks. See `LEARNINGS.md`. Keep the `.woff2` files.
- The copy is deliberately generic so the same page works for any campaign
  (admission, discount, offer). Tailor the `.card` text if a campaign needs it.
