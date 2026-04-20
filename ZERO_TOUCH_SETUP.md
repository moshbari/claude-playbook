# Zero-touch setup — what Mosh needs configured once so Claude can do everything

_Last updated: 2026-04-20_

## The contract

Mosh's goal: "I say `build X`, Claude writes code, pushes to GitHub, it goes live. I never touch the server, Coolify, DNS, or the terminal."

That contract only holds if five specific things are set up once on Mosh's account. Missing any of them means Claude falls back to asking Mosh for clicks — which is the failure mode this file is built to prevent.

Run this check at the start of every new project. Flag missing items up front, don't discover them halfway through.

## The five items

### 1. Coolify GitHub App (3 min)

Without it, deploys require POST to `/api/v1/deploy?uuid=...` — which means a valid API token with a widened IP allowlist. With it, `git push` triggers the deploy automatically, signed, no token-in-sandbox needed.

How to install (Mosh does this once):
- Open `http://<server-ip>:8000/` (Coolify dashboard)
- Sources → Add → GitHub App
- Continue with GitHub, authorize, install on his account
- "All repositories" scope (so Claude can create new repos and wire them up without re-installing)

After this: every Claude-created repo auto-deploys on push. No manual API calls.

### 2. Cloudflare API token (5 min)

Namecheap has no public DNS API worth using. Cloudflare free tier does. Move each domain Mosh wants Claude to automate onto Cloudflare nameservers (Namecheap stays the registrar), then generate an API token scoped to those zones.

- dash.cloudflare.com → Add site → enter domain → Free plan → note nameservers
- Namecheap → domain → Nameservers → Custom DNS → paste Cloudflare nameservers
- Cloudflare → My Profile → API Tokens → Create Token → template "Edit zone DNS" → include all Cloudflare-managed zones
- Save token into `secrets.txt` (see item 5)

Mosh's current state (as of 2026-04-20):
- `99dfy.com` on Cloudflare ✅
- `bizapp.club` — check. If still Namecheap-only, flag it as a gap.

### 3. SSH key for Claude's sandbox (2 min)

For emergencies where a Coolify API path is broken and Claude needs to touch a file directly on the server.

Each new Claude session: Claude generates a keypair in its sandbox and prints the public key. Mosh runs once on his Mac:

```
ssh root@<server-ip>
# paste server password from Apple Notes
echo 'PASTE-PUBLIC-KEY' >> ~/.ssh/authorized_keys
exit
```

One-time per Mac Claude runs on.

### 4. Classic GitHub PAT (3 min)

Fine-grained PATs can't create repositories or register webhooks. Classic PATs with `repo` + `admin:repo_hook` can. Both failure modes bit the apps.bizapp.club build.

- github.com/settings/tokens → Generate new token (Classic)
- Scopes: `repo` (full) + `admin:repo_hook`
- Expiration: 1 year or "no expiration"
- Save into `secrets.txt`

Also: commit author email must be the GitHub noreply form — `<userid>+<username>@users.noreply.github.com` — or pushes fail with GH007 email-privacy block. See `INFRA_HETZNER_COOLIFY.md` for the email-privacy note.

### 5. `secrets.txt` in a folder Claude mounts (5 min)

The single file Claude reads at session start. Put it in a folder you always select as Claude's working directory (e.g. `~/Documents/claude-server`).

```
# Hetzner + Coolify server
SERVER_IP=<ip>
COOLIFY_URL=http://<ip>:8000
COOLIFY_API_TOKEN=<from Coolify UI → Keys & Tokens>
SERVER_SSH_USER=root

# GitHub
GITHUB_USERNAME=moshbari
GITHUB_NOREPLY_EMAIL=<id>+moshbari@users.noreply.github.com
GITHUB_PAT_CLASSIC=<from item 4>

# Cloudflare DNS (covers all Cloudflare-managed zones)
CLOUDFLARE_API_TOKEN=<from item 2>
CLOUDFLARE_ZONES=99dfy.com,bizapp.club

# Parent domains available for new projects (comma-separated)
PARENT_DOMAINS=99dfy.com,bizapp.club

# GoHighLevel — one block per sub-account Mosh wants Claude to upload to
GHL_SUBACCOUNT_1_NAME=<friendly name>
GHL_SUBACCOUNT_1_LOCATION_ID=<from GHL URL>
GHL_SUBACCOUNT_1_PIT=<pit-xxxx from Settings → Private Integrations>

# GHL_SUBACCOUNT_2_NAME=...
# GHL_SUBACCOUNT_2_LOCATION_ID=...
# GHL_SUBACCOUNT_2_PIT=...
```

**Nothing from this file goes in the playbook or any committed repo.** The playbook has the shape; only Mosh's Mac has the values.

## What Claude asks at the start of every new project

Even with all five items set up, Claude still asks three questions before writing code. These are user-visible choices, not things Claude should default silently:

1. **Which domain?** Options come from `secrets.txt` `PARENT_DOMAINS` plus "new brand domain" plus "subdomain of existing brand." Default suggestion: a subdomain of `99dfy.com`, but never assume.
2. **Does the app need media storage?** If yes: which GHL sub-account, and Mosh creates a folder in that sub-account's Media Library with a name he chooses. Claude resolves the folder ID at runtime from the GHL API.
3. **Any user-visible choices to confirm up front?** App name, URL shape, key UX. Only the things Mosh should decide.

Everything else Claude handles.

## What Claude does first in every session

1. Fetch this repo (the playbook). Read files relevant to the task — minimum: `CLAUDE.md`, `PROJECTS.md`, this file, and whichever specialist files apply.
2. Read `secrets.txt` from the mounted folder.
3. Before quoting or using any credential, verify it against the live service:
   - Coolify: `GET /api/v1/teams` (any cheap authenticated GET)
   - GitHub: `GET /user`
   - Cloudflare: `GET /user/tokens/verify`
   - GHL: `GET /medias/files?altId=<loc>&altType=location&type=folder&limit=1`
4. If any check fails, tell Mosh before doing anything that depends on that credential.

Never quote a secret from memory. Memory records what was true when it was written; live services change.

## What stays manual, forever

1. Telling Claude what to build.
2. Creating the GHL media folder per project (Mosh does this in the GHL UI; takes 30 seconds).
3. Answering 2-option UX questions.
4. Reviewing before Claude publishes anything under Mosh's name (posts, emails).
5. Rotating tokens when they expire (Cloudflare, GitHub PAT, GHL PIT — all annually).

Everything else Claude handles.
