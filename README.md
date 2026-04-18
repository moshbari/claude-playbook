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

- **Any writing, UI, or product work** → read `WRITING_STYLE.md` and `UX_CLICK_ONLY.md`.
- **Shipping a SaaS with billing or auth** → read `SAAS_LAUNCH_CHECKLIST.md`.
- **Deploying anything** → read `INFRA_HETZNER_COOLIFY.md`.
- **Adding a new lesson** → append to the right file and bump the "Last updated" line at the top.

## How to use (for me — Mosh)

When Claude and I hit a new lesson — a mistake, a surprise, a thing that took too long to figure out — I tell Claude "add this to the playbook." Claude commits it to this repo. Next project, next Claude, same knowledge.

## Files

| File | What's in it |
|---|---|
| `WRITING_STYLE.md` | 5th-grade reading level rule. Applies to product copy, emails, and chat. |
| `UX_CLICK_ONLY.md` | Click-only UX rule. Grandma-with-thick-glasses test. |
| `SAAS_LAUNCH_CHECKLIST.md` | Pre-launch checklist for SaaS with billing webhooks and auth. |
| `INFRA_HETZNER_COOLIFY.md` | My standard deploy stack. Hetzner + Coolify + Cloudflare DNS. |
| `PROJECTS.md` | Active and shipped projects — what they are, where they live. |

## Owner

Mosh (engrmoshbari@gmail.com).

_Last updated: 2026-04-18_
