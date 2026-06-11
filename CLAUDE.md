# CLAUDE.md — Standing instruction for every project

This file is here so that when Claude Code or any Claude-aware tool pulls this repo, it picks up the rules automatically.

## Rule 1 — Read the playbook at the start of every new project or task

Before writing code, drafting copy, or making decisions on one of Mosh's projects, fetch and read the files in this repo that apply:

- Starting any new project (kickoff checklist + questions to ask) → `ZERO_TOUCH_SETUP.md`
- Writing, UI, marketing copy → `WRITING_STYLE.md` + `UX_CLICK_ONLY.md`
- Picking a URL or subdomain structure → `URL_CONVENTIONS.md`
- SaaS with billing or auth → `SAAS_LAUNCH_CHECKLIST.md`
- Whop billing specifically (webhooks, signatures, checkout URLs) → `WHOP_WEBHOOKS.md`
- Deploy / infra work (Hetzner + Coolify) → `INFRA_HETZNER_COOLIFY.md`
- Driving the Coolify REST API directly → `COOLIFY_API_QUIRKS.md`
- Next.js on Railway / Dockerfile projects → `NEXTJS_DOCKER_RAILWAY.md`
- Any FFmpeg / audio-video work (cut, join, pitch, re-encode, sync, diarization) → `FFMPEG_MEDIA_PIPELINES.md`
- Any GoHighLevel media upload work → `GHL_MEDIA_UPLOAD.md`
- Deploy looks green but something silently isn't working → `DEBUGGING_DEPLOYMENTS.md`
- Working from Cowork specifically (Chrome MCP walls, sandbox IP drift) → `COWORK_QUIRKS.md`
- Project context ("what is this?") → `PROJECTS.md`

Apply the rules from that moment on. No permission needed — just do it.

## Rule 2 — Save new lessons back to this repo

When a new lesson comes up — a mistake made, a surprise hit, a thing that took too long to figure out — commit it to the right file in this repo and push.

If Mosh says any of these, treat it as "save to playbook":
- "add this to the playbook"
- "remember this for next time"
- "let's not make this mistake again"
- "note this down"

Don't write it only to local / session memory. Local memory doesn't travel to other Claude surfaces. The repo does.

## Rule 3 — The repo is the source of truth

If local memory (Cowork auto-memory, chat memory, etc.) disagrees with what's in this repo, the repo wins. Update local memory to match.

## Files

See `README.md` for the current file index.
