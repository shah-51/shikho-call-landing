# CHANGELOG — Shikho Call-Now Landing

## 2026-08-03 — Initial page shipped
Single-file static landing (`index.html`) for CleverTap "call now" push notifications.
Tap the button → dialer opens with 16780 pre-filled. Deployed to `shikho-call.vercel.app`.
Fonts self-hosted (Hind Siliguri woff2, three weights) after discovering render-blocking
Google Fonts caused blank first paint on BD mobile. Vercel caching headers set for fonts
and logo (1-year immutable).
