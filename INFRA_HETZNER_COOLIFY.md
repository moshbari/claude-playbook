# Infra — Hetzner + Coolify

_Last updated: 2026-04-20_

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

## Small Coolify quirks worth knowing

- **"Developer view" for bulk env var edits.** On the Environment Variables page, there's a "Developer view" toggle next to "+ Add". It swaps the per-variable form for a single dotenv textarea containing every var. Paste or append lines, click "Save All Environment Variables" — done. Much faster than clicking "+ Add" three times and filling two fields each.
- **The "latest configuration has not been applied" banner is noisy.** If you save env vars and click Redeploy within the same second, the banner shows up during the in-progress deploy and then clears when it finishes. It does NOT mean the new deploy is using stale env vars — Coolify picks up the saved values as long as they were saved before you clicked Redeploy. Ignore the banner until the deploy completes; only worry if it's still there afterwards.

## GitHub automation — the tokens and settings that actually work

Not all GitHub personal access tokens are equal. These are the traps hit on the apps.bizapp.club build:

- **Fine-grained PATs cannot create repositories** via the API. The UI lets you generate one with `Contents: Read and write` permission, but `POST /user/repos` still returns 403. Use a **classic PAT** with the `repo` scope to create repos programmatically.
- **Fine-grained PATs cannot register webhooks.** Same 403 mode. `admin:repo_hook` scope on a classic PAT is required.
- **Commit author email must be the GitHub noreply form.** Pushing with `engrmoshbari@gmail.com` as author triggers GH007 ("email address is not in your verified list") and the push is rejected. Use `<github-user-id>+<username>@users.noreply.github.com` — find the ID in github.com/settings/emails under "Keep my email addresses private."
- **Store the classic PAT as `GITHUB_PAT_CLASSIC` in `secrets.txt`.** Never commit it. See `ZERO_TOUCH_SETUP.md` for the full secrets file template.

## Coolify GitHub App vs API-token deploys

Two ways to trigger deploys from Claude's sandbox:

1. **API-token path** — `POST /api/v1/deploy?uuid=<app>&force=true`. Requires a valid API token AND the API IP allowlist widened to include Claude's sandbox (which varies). High friction.
2. **GitHub App path** — install Coolify's official GitHub App on your account (Coolify → Sources → Add → GitHub App). Once installed, every `git push` to the connected repo auto-deploys. No API token involved in the deploy. No IP allowlist to widen.

Pick path 2 for any repo that will see more than one deploy. See `ZERO_TOUCH_SETUP.md` for the install steps.

## Related projects running on this stack

- `printables.99dfy.com` — Printables SaaS
- `apps.bizapp.club` — Coolify-native app-deployer dashboard. Second parent domain on the same box. Child apps at flat `<name>.bizapp.club`.
- `storyteller.click` — (separate hosting? confirm before assuming)
- `heychatmate.com` — (separate hosting? confirm)
