# Projects

_Last updated: 2026-05-08_ (GoNoGo added)

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

## apps.bizapp.club

- **Domain:** `apps.bizapp.club` (dashboard) + `<name>.bizapp.club` (flat child-app URLs)
- **Repo:** github.com/moshbari/apps-bizapp-club (public)
- **Pitch:** Coolify-native rebuild of the original HestiaCP app-deployer. Single-container Node/Express that serves the dashboard plus Host-header-routed child apps (Node apps and static HTML apps).
- **Stack:** Node/Express, file-backed JSON store, scrypt password hashing, Coolify on Hetzner, Traefik for TLS.
- **Parent domain split:** `PARENT_DOMAIN=apps.bizapp.club` (dashboard host), `APP_DOMAIN=bizapp.club` (child-app suffix). They differ by design — dashboard lives under `apps.` but child apps are flat under the root. See `URL_CONVENTIONS.md`.
- **DNS:** Namecheap registrar, manual A records (no Cloudflare for this zone yet). `apps.bizapp.club` A → server IP; `*.bizapp.club` wildcard A → server IP; legacy `*.apps.bizapp.club` wildcard can be cleaned up. See `COOLIFY_API_QUIRKS.md` for the domain-update dance.
- **Reserved subdomains:** `apps`, `www`, `mail`, `api`, `admin`, `ftp`, `smtp`, `pop`, `imap`, `ns1`, `ns2`.
- **Admin:** username `admin` (not an email). Password lives in `secrets.txt`.
- **Second parent domain, not a replacement for `99dfy.com`.** When starting a new project, ask Mosh which parent domain to use.
- **Status:** Live. Deployed via Coolify API (IP allowlist widens + reverts pattern). Next: install Coolify GitHub App for push-to-deploy.

## listen.bizapp.club

- **Domain:** `listen.bizapp.club` (child app under the `bizapp.club` parent, covered by the existing `*.bizapp.club` wildcard — no manual DNS record needed)
- **Repo:** github.com/moshbari/listen-bizapp-club (private)
- **Pitch:** Upload an iPhone voice memo, get a share link. Recipients tap the link and play in-browser. No download, no login for listeners — solves the "WhatsApp can't open the voice memo" problem.
- **Target customer:** Mosh personally, for sharing recorded thoughts with friends/family over WhatsApp and other messengers.
- **Stack:** Node.js 20 + Express, SQLite (`better-sqlite3`) on a Coolify persistent volume, ffmpeg for transcode + split, curl for GHL uploads (same pattern as StepWise), Docker on Coolify/Hetzner.
- **Auth:** Single shared password for uploads (signed cookie session). Listen pages are public, unguessable 8-char nanoid slug at `/p/<slug>`. `X-Robots-Tag: noindex` on listen pages.
- **Pipeline:** any audio in → ffmpeg transcode to **64 kbps mono 22.05 kHz MP3** → if > 24 MB, split into 40-minute parts via `ffmpeg -f segment -c copy` → upload each part to GHL media library → store slug + part URLs in SQLite → render listen page with `<audio controls controlslist="nodownload">` that auto-advances via the `ended` event.
- **GHL integration:** sub-account `AI Wealth Accelerator` (location `MV4qgCBrDVTq6S9QIYNa`), folder `69e5b3892c135a8c837d45a5`. Hardcodes `;type=audio/mpeg` on the curl form field because `node:20-slim` has no `/etc/mime.types` — see `GHL_MEDIA_UPLOAD.md` row #9.
- **Why GHL and not S3:** reuses the sub-account's CDN (`assets.cdn.filesafe.space`), zero new infra bills, and Mosh already pays for GHL. `audio/mpeg` files are publicly streamable from the CDN with `Accept-Ranges: bytes` and `Access-Control-Allow-Origin: *` — verified in production.
- **Coolify app UUID:** `j5fxh5htxnrjozyjgn54neog`. Persistent volume mounted at `/app/data` for the SQLite DB. Deploy via `POST /api/v1/deploy?uuid=<uuid>&force=true`.
- **Not committed to repo:** `.env` (Coolify owns the runtime env).
- **Status:** Live. Verified end-to-end 2026-04-20 — MP3 streams from GHL CDN with `content-type: audio/mpeg` and range support.

## GoNoGo

- **Domain:** `gonogo.99dfy.com` (frontend, Lovable) + `gonogo-api.99dfy.com` (orchestrator API, Coolify)
- **Repos:** `github.com/moshbari/gonogoyes` (frontend, private) + `github.com/moshbari/gonogo-orchestrator` (backend, private)
- **Pitch:** Type a business idea, get a GO / CAUTION / NO_GO verdict in 60 seconds. 5 AI agents (Market Researcher, Competition Scout, Financial Modeler, Customer Profiler, Team Lead) analyze in parallel against real local data — US Census, Google Places, BLS, Google Trends.
- **Target customer:** First-time business owners. Bangladeshi diaspora in US community hubs (MI, NJ, NY, TX, CA, VA, IL) is the seed audience.
- **Why it exists:** Marketing engine for the $497 workshop. The app itself isn't the profit center — cheap shareable verdicts pull people into the workshop funnel.
- **Stack:** Lovable React/Vite frontend + Node/Express orchestrator on Coolify (Hetzner) + Supabase Postgres (shared `yuqeqricenygpsrdfswx` project, dedicated `gonogo` schema only) + Anthropic API (Sonnet specialists, Opus team lead) + Resend SMTP.
- **Auth:** Supabase magic-link email + Google OAuth (PKCE).
- **User tiers:** trial (3 free runs) / full (paid) / admin (unlimited) / disabled. `gonogo.user_profiles` table holds `runs_used`, `runs_limit`, `role`.
- **Billing:** Whop, $7 / 10-credit pack (live test was $1), product `prod_3cCKEsbBOPW1e`, plan `plan_0pvcREpBai6z8`, company `biz_6lPb8XIs7llD4t`. See `WHOP_WEBHOOKS.md` for the full integration story.
- **Webhook URL:** `https://gonogo-api.99dfy.com/whop/webhook` (path matters — see `WHOP_WEBHOOKS.md`).
- **Critical Supabase rule:** project `yuqeqricenygpsrdfswx` is shared with FreedomClockV25Sep25 and other apps. **Only touch the `gonogo` schema.** Never modify `public` or any pre-existing table, policy, function, or auth setting.
- **Coolify app uuid:** `csajucyjxzb8nztpy9o75vji`.
- **Status:** Live. End-to-end verified 2026-05-08 — real $1 Whop purchase → Svix webhook → +10 credits landed automatically on a trial account, role auto-promoted to `full`.

## Notes for future projects

When starting a new one, always:

1. **Ask Mosh which parent domain.** Options: `99dfy.com` (default for most tools), `bizapp.club` (for bizapp-branded tools), a new domain he'll register for a standalone brand, or a subdomain of an existing brand like `heychatmate.com`. Never assume.
2. **Ask whether it needs GHL media storage.** If yes, ask which GHL sub-account and have Mosh create a named folder in that sub-account's Media Library. Claude resolves the folder ID at runtime.
3. Create a GitHub repo under `moshbari/` (classic PAT required — see `INFRA_HETZNER_COOLIFY.md`).
4. Deploy via Coolify — see `INFRA_HETZNER_COOLIFY.md` and `COOLIFY_API_QUIRKS.md`.
5. If there's billing, walk `SAAS_LAUNCH_CHECKLIST.md` before calling it shipped.
6. Verify DNS from 3 resolvers, TLS issuer, and body content before telling Mosh it's live — see `DEBUGGING_DEPLOYMENTS.md`.
7. Add the project to this file with the same fields as Printables above.
