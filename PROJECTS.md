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

## Notes for future projects

When starting a new one, always:

1. Pick a subdomain of `99dfy.com` (or a new domain if it's a separate brand).
2. Create a GitHub repo under `moshbari/`.
3. Deploy via Coolify — see `INFRA_HETZNER_COOLIFY.md`.
4. If there's billing, walk `SAAS_LAUNCH_CHECKLIST.md` before calling it shipped.
5. Add the project to this file with the same fields as Printables above.
