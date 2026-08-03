# LEARNINGS — Shikho Call-Now landing

## 2026-08-03 — Render-blocking Google Fonts = "forever to load" on BD mobile
**Gotcha:** page tested fine on desktop/my network but users on Bangladesh mobile
reported it "taking forever." The HTML was 9.8KB and served in ~0.4s from Vercel's
Mumbai (bom1) edge — not the problem. The blocker was the render-blocking
`<link>` to `fonts.googleapis.com` + the Bengali woff2 pulls from
`fonts.gstatic.com`; that route is slow/throttled on BD mobile, and first paint
sat blank until it responded.
**Takeaway:** for any public page targeting BD mobile, **self-host fonts
same-origin** and drop the Google Fonts `<link>` entirely. Grab the exact
Bengali-subset woff2 URLs from the css2 API (fetch it with a Chrome UA so you get
woff2, filter to the `U+0980` unicode-range block), download per weight (~73KB
each for Hind Siliguri), inline `@font-face` with `font-display: swap`, and set a
1-year immutable Cache-Control. Latin numerals can just use `system-ui` — one
fewer file. Result: fully same-origin, first paint independent of any third party.

## 2026-08-03 — CSS `scale` property frees `transform` for press feedback
Wanted a looping "breath" swell on the CTA *and* a translate-down on `:active`.
Animating `transform` for the swell kills the `:active` transform (animation wins).
Fix: drive the swell with the independent **`scale:`** property and keep
`transform: translateY()` for the press — they compose, so both work. (iOS 14.5+ /
Chrome 104+; degrades to no-swell on older, harmless.)

## Design notes
- Multiple micro-animations read as "alive" not "busy" only if their periods
  differ (here: 2.0 / 2.4 / 2.8 / 3.6s) so they never all fire together.
- Badge copy on a reused notification landing page must be campaign-neutral —
  "শিখো হেল্পলাইন" stays true whether the push is admission, discount, or offer.
