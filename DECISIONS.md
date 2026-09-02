# DECISIONS — Shikho Call-Now Landing

## 2026-08-03 — Self-host fonts, no Google Fonts link
Dropped `<link>` to `fonts.googleapis.com`. Downloaded Hind Siliguri woff2 per weight,
inlined `@font-face` with `font-display: swap`, served same-origin.
**Why:** render-blocking Google Fonts caused blank first paint on Bangladeshi mobile networks —
the primary audience tapping a push notification on a slow connection. First paint must be
independent of any third-party route. Latin numerals fall back to `system-ui` (one fewer file).

## 2026-08-03 — CSS `scale` property for the breath animation, not `transform: scale()`
The CTA button has a looping breath swell and a translate-down on `:active`. Driving the swell
via `transform` would have overridden the `:active` translate (animation wins the cascade).
**Why:** the `scale` CSS property (Chrome 104+ / iOS 14.5+) is a separate composited property
from `transform`, so both effects compose. Degrades to no-swell on older browsers, harmlessly.

## 2026-08-03 — No framework, no build step
Pure `index.html` with inline CSS and a small click-tracking script. No npm, no bundler.
**Why:** the page does exactly one thing. A build pipeline adds nothing except friction and
surface area. Non-technical editors can paste the file into an LLM and paste the result back.
