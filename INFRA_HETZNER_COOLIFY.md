# Infra — Hetzner + Coolify

_Last updated: 2026-04-18_

## My standard deploy stack

- **Server:** Hetzner, label `coolify-prod`, IP `178.156.240.200`
- **Orchestrator:** Coolify, runs on the same box, web UI at `http://178.156.240.200:8000`
- **Root domain:** `99dfy.com` — DNS on Cloudflare
- **Pattern:** every new project gets a subdomain of `99dfy.com` pointing at the Hetzner IP

## When I say "deploy to my server"

I mean this stack. Don't re-ask about hosting. Pick a subdomain, add the Cloudflare DNS record, deploy via Coolify.

## Rules that have bit me

- **Env vars always go via Coolify UI (or API), never committed to the repo.**
- **New subdomain** (like `printables.99dfy.com`) needs an A record in Cloudflare pointing to `178.156.240.200`. Cloudflare proxy (orange cloud) is fine for most apps.
- **Env var change = Redeploy, not Restart.** Restart reuses the old container. Redeploy rebuilds.
- **GitHub auto-deploy is not on by default.** Set it up in Coolify → Git Source if I want `git push` to trigger deploys. Otherwise every push needs a manual Redeploy click.

## Sandbox access gotchas for Claude

The sandbox network proxy blocks direct HTTP access to my Hetzner IP (`178.156.240.200:8000`). To configure Coolify, Claude needs one of:

1. Drive Chrome on my computer (via Chrome MCP) — works, I'm usually signed in already.
2. Get a Coolify API token and run commands from my local machine.
3. I do the Coolify UI step myself while Claude tells me what to click.

## Useful Coolify spots

- **Configuration → Environment Variables** — where secrets live.
- **Deployments** — list of recent deploys with commit SHAs. First stop if "my change isn't live."
- **Terminal** — in-container shell. Good for running `node -e "..."` or Prisma commands against the live DB.
- **Webhooks** (in Git Source) — where to paste the provider URL for GitHub auto-deploy.

## Related projects running on this stack

- `printables.99dfy.com` — Printables SaaS
- `storyteller.click` — (separate hosting? confirm before assuming)
- `heychatmate.com` — (separate hosting? confirm)
