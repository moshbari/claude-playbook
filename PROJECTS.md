# Projects

_Last updated: 2026-04-18_

Active and shipped projects. Claude: when I mention a project by name, check here first.

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
