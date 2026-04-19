# Projects

_Last updated: 2026-04-19_

Active and shipped projects. Claude: when I mention a project by name, check here first.

## StepWise

- **Domain:** app.heychatmate.com (hosted guides + API), heychatmate.com (marketing, on GHL)
- **Repo:** github.com/moshbari/stepwise (public)
- **Pitch:** Chrome extension for step-by-step workflow documentation. Captures annotated screenshots, generates AI-narrated MP4 videos, publishes guides as hosted HTML pages.
- **Target customer:** Agency owners and SOP-heavy teams who need repeatable how-to content fast.
- **Stack:** Chrome extension (MV3) + Node.js/Express on Contabo VPS (`109.205.182.135`), HestiaCP managed, PM2 process manager, Nginx reverse proxy, FFmpeg for video, OpenAI TTS (narration) + Whisper (voice-to-text).
- **GHL integration:** Videos auto-uploaded to GHL sub-account "AI Wealth Accelerator" (location `MV4qgCBrDVTq6S9QIYNa`), folder "Step Wise 13Feb26". Uses curl + `parentId` form field pattern. See `GHL_MEDIA_UPLOAD.md`.
- **Publish pipeline:** HTTPS proxy at `app.heychatmate.com/stepwise-api/` → Node on port 3600. Deploy webhook on port 3601 (auto-deploy on GitHub push to main).
- **Version control rule:** Every extension change bumps `manifest.json`, the version badge in `editor.html` header, the entry in `server/downloads.html`, and ships a new zip on the downloads page. Never push extension changes without the version bump.
- **Origin of `ghl-uploader.js`:** This is where the GHL pattern was first cracked. The file lives at `/opt/stepwise-video/ghl-uploader.js` on the VPS and is **not yet committed to GitHub**.
- **Status:** Live. v2.4.2 on the Chrome Web Store.

## Printables

- **Domain:** printables.99dfy.com
- **Repo:** github.com/moshbari/printables
- **Pitch:** Type your idea → get an Etsy-ready printable (cover, title, tags, Pinterest pins) in 60 seconds.
- **Target customer:** 35+ folks looking for a 9-to-5 replacement side hustle. "Webinar audience" persona.
- **Billing:** Whop, $37/month, product `prod_nGDbEhFtK7JAO`, company `biz_6lPb8XIs7llD4t`.
- **Auth:** Magic-link email login (NextAuth, no passwords).
- **Stack:** Next.js 14 App Router (standalone build), Prisma + Postgres, Coolify on Hetzner.
- **Status:** Live. Webhook wired and verified 2026-04-18.

## EveryLink

- **Domain:** everylink.click (Namecheap, active Apr 11, 2026 → Apr 11, 2027, WhoisGuard on)
- **Repo:** github.com/moshbari/everylink
- **Pitch:** The marketer's command center. Link-in-bio + native link cloaker + WarriorPlus/JVZoo billing + white-label from day one. Not another Linktree clone — aimed at the IM crowd.
- **Target customer:** Digital marketers running affiliate products and webinars. Speed, analytics, and conversion matter more than aesthetics.
- **Super admin:** engr.mbari@gmail.com
- **Stack:** Next.js, Supabase (DB + Auth), Railway (hosting via Dockerfile — see `NEXTJS_DOCKER_RAILWAY.md`), GHL media library for file hosting (see `GHL_MEDIA_UPLOAD.md`).
- **Domain structure:** `everylink.click` (marketing) · `app.everylink.click` (dashboard) · `*.everylink.click` (user bio pages, wildcard) · `api.everylink.click` (API) · custom domains per white-label tenant.
- **Pricing tiers:** Free 7-day trial · Starter $9–12 · Pro $27–37 · Agency/White-Label $97–197.
- **Billing:** WarriorPlus + JVZoo IPN (not Whop/Stripe — the IM crowd buys through those platforms).
- **Status:** In build.

## Notes for future projects

When starting a new one, always:

1. Pick a subdomain of `99dfy.com` (or a new domain if it's a separate brand).
2. Create a GitHub repo under `moshbari/`.
3. Deploy via Coolify — see `INFRA_HETZNER_COOLIFY.md`.
4. If there's billing, walk `SAAS_LAUNCH_CHECKLIST.md` before calling it shipped.
5. Add the project to this file with the same fields as Printables above.
