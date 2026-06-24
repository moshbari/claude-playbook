# Mosh's Claude Playbook

Personal knowledge base for working with Claude across every surface — Cowork, web, mobile, Claude Code, API. Lessons, rules, and checklists that travel with my GitHub account.

## How this works

Whenever I start a new Claude session on anything — a new chat, a new project, a new device — I can point Claude here and say "read my playbook." From that point on, Claude knows my preferences, my stack, my past mistakes, and how I like to work.

Any Claude (web, mobile, Cowork, Claude Code) with internet access can read this repo. That makes it the single source of truth for our shared knowledge.

## Standing instruction (read this first, Claude)

**At the start of EVERY new project or task on Mosh's account — on any surface — fetch this repo and read the relevant files.** See `CLAUDE.md` for the full standing instruction.

**When we hit a new lesson, commit it back to this repo.** Local memory doesn't travel between Claude surfaces. This repo does.

## How to use (for Claude)

At the start of a new project or task, Claude should fetch this repo's `README.md` and then read the files relevant to what we're doing:

- **Starting ANY new project** → read `ZERO_TOUCH_SETUP.md` first, then `PROJECTS.md`.
- **Any writing, UI, or product work** → read `WRITING_STYLE.md` and `UX_CLICK_ONLY.md`.
- **Any public store listing or app branding (name, publisher, description)** → read `BRANDING.md`.
- **Picking a URL / subdomain structure** → read `URL_CONVENTIONS.md`.
- **Shipping a SaaS with billing or auth** → read `SAAS_LAUNCH_CHECKLIST.md`.
- **Whop billing specifically (webhooks, signatures, checkout URLs)** → read `WHOP_WEBHOOKS.md`.
- **Deploying anything** → read `INFRA_HETZNER_COOLIFY.md`.
- **Driving the Coolify REST API directly** → read `COOLIFY_API_QUIRKS.md`.
- **Uploading images/files to GoHighLevel media library** → read `GHL_MEDIA_UPLOAD.md`.
- **Any Next.js project on Railway (especially with a Dockerfile)** → read `NEXTJS_DOCKER_RAILWAY.md`.
- **Adding a custom domain on Railway (or one stuck on VALIDATING_OWNERSHIP)** → read `RAILWAY_CUSTOM_DOMAINS.md`.
- **Adding a new app to a Supabase project that other apps already use** → read `SUPABASE_SHARED_PROJECT.md`.
- **Any FFmpeg / audio-video pipeline (cut, join, pitch-shift, re-encode, A/V sync, speaker diarization)** → read `FFMPEG_MEDIA_PIPELINES.md`.
- **"It deployed but something silently isn't working"** → read `DEBUGGING_DEPLOYMENTS.md`.
- **Working in Cowork specifically** → read `COWORK_QUIRKS.md`.
- **Adding a new lesson** → append to the right file and bump the "Last updated" line at the top.

## How to use (for me — Mosh)

When Claude and I hit a new lesson — a mistake, a surprise, a thing that took too long to figure out — I tell Claude "add this to the playbook." Claude commits it to this repo. Next project, next Claude, same knowledge.

## Files

| File | What's in it |
|---|---|
| `ZERO_TOUCH_SETUP.md` | The five one-time items I need so Claude can do everything without me clicking. |
| `WRITING_STYLE.md` | 5th-grade reading level rule. Applies to product copy, emails, and chat. |
| `BRANDING.md` | Public branding rule — "Mosh Bari" as the visible personal brand (name/keyword/Google), "ZPresso LLC" as the publisher/legal entity. Use both, in different slots. |
| `UX_CLICK_ONLY.md` | Click-only UX rule. Grandma-with-thick-glasses test. |
| `URL_CONVENTIONS.md` | Flat subdomains over nested. Why `<name>.brand.com` beats `<name>.app.brand.com`. |
| `SAAS_LAUNCH_CHECKLIST.md` | Pre-launch checklist for SaaS with billing webhooks and auth. |
| `WHOP_WEBHOOKS.md` | Whop deep-dive — Svix signature scheme, secret format, checkout URL gotchas, idempotency. |
| `INFRA_HETZNER_COOLIFY.md` | My standard deploy stack. Hetzner + Coolify + Cloudflare DNS + GitHub automation. |
| `COOLIFY_API_QUIRKS.md` | Every field name and sequencing trap in Coolify's REST API. `domains` vs `fqdn`, restart-after-fqdn, storage types, IP allowlist, webhook signatures. |
| `NEXTJS_DOCKER_RAILWAY.md` | Next.js on Railway with a Dockerfile — build-time env vars, lazy-init clients, missing system binaries. |
| `FFMPEG_MEDIA_PIPELINES.md` | FFmpeg audio/video traps — sample-rate pitch-shift bug (resample before `asetrate`), time-selective effects via concat with exact-length pieces (amix OOMs at scale), ffprobe-verify-the-artifact, fail-closed renders, heavy re-encodes → background jobs, YouTube/VFR timeline portability, AssemblyAI diarization. |
| `RAILWAY_CUSTOM_DOMAINS.md` | Railway custom domains — the TXT verification trap, targetPort 502 trap, project-token vs PAT, full GraphQL endpoints. |
| `SUPABASE_SHARED_PROJECT.md` | Multiple apps in ONE Supabase project — table prefix rule, INSERT RLS policy, SMTP bypass via Resend, redirectTo trailing-slash trap. |
| `GHL_MEDIA_UPLOAD.md` | GoHighLevel media library upload — the endpoint, auth, folder traps, env var traps. |
| `DEBUGGING_DEPLOYMENTS.md` | Verification sequence for silent-failure bugs, including DNS + TLS checks before saying "done." |
| `COWORK_QUIRKS.md` | Cowork-specific gotchas — Chrome MCP's two permission gates, sandbox IP drift, when to stop fighting. |
| `PROJECTS.md` | Active and shipped projects — what they are, where they live. |

## Owner

Mosh (engrmoshbari@gmail.com).

_Last updated: 2026-06-11 (RANT Squad Clip Maker: FFMPEG_MEDIA_PIPELINES.md — added §2 correction (volume-gated amix OOMs at ~130 windows; use concat with apad+atrim EXACT-length pieces = sync-safe AND scalable) and §9 fail-closed (never mark a no-file render complete; a failed privacy effect must abort, not ship the un-disguised version). Earlier same day: new FFMPEG_MEDIA_PIPELINES.md — the 44100-vs-48000 `asetrate` pitch-shift bug that drifted lip-sync, volume-gated continuous tracks instead of cut-and-stitch for A/V sync, always ffprobe the output durations + test on real input, heavy re-encodes → background jobs (sync request = "Failed to fetch"), YouTube/VFR timeline portability, AssemblyAI diarization; DEBUGGING_DEPLOYMENTS gained the Railway "exact pre-fix value = deploy not live yet" tell). Earlier: 2026-05-08 (GoNoGo Whop credit-pack: new WHOP_WEBHOOKS.md covers Svix signature scheme + metadata-drop + product-page-vs-checkout URL trap; SAAS_LAUNCH_CHECKLIST gained items 16-24; PROJECTS got GoNoGo). Earlier: 2026-05-03 (GHL: rewrote folder-placement section — `?folderId=` URL query is unreliable too, single recipe is parentId+altId+altType form fields; added symptom→diagnosis lookup table; DEBUGGING gained Git/GitHub gotchas section — GH007 email-privacy push rejection + workspace-folder-not-a-repo trap. Earlier: added ZERO_TOUCH_SETUP, COOLIFY_API_QUIRKS, URL_CONVENTIONS, COWORK_QUIRKS; DEBUGGING_DEPLOYMENTS gained DNS+TLS section and docker-exit-255 retry note; INFRA_HETZNER_COOLIFY got GitHub automation notes + same-SHA-redeploy and log-link quirks; PROJECTS got apps.bizapp.club and multi-domain kickoff rule; Path A locked in — Coolify API allowlist stays at 0.0.0.0/0 by design, do not revert)_
