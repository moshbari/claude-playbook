# Cowork and Claude tooling quirks

_Last updated: 2026-04-20_

Things that only happen when driving Mosh's workflows from Cowork. Not deployment issues — tooling-surface issues. Reading this before trying to automate a UI task from Cowork saves 10-minute permission-hunting sessions.

## Chrome MCP has TWO permission gates, not one

When driving a browser tab via `mcp__Claude_in_Chrome__*` tools from Cowork, there are two independent permissions:

1. **Chrome's native site access** — configured in `chrome://extensions` → Claude extension → Details → Site access. "On specific sites" with the target URL added, or "On all sites." This gate is visible in Chrome settings.
2. **Per-action permission inside the Claude extension itself** — not exposed in Chrome's extension UI. Each tool call surfaces a prompt somewhere (possibly only visible when Mosh clicks the Claude icon in the toolbar, possibly nowhere). If denied or missed, every action returns `"Permission denied by user"`.

**Diagnostic signature — tells you it's gate #2, not gate #1:** `tabs_context_mcp` succeeds (returns the tab list), but `screenshot`, `find`, `navigate`, `computer`, and everything else on the same tab return `"Permission denied by user"` — even after granting full site access in Chrome settings and hard-refreshing.

**Rule: don't fight it.** After two denied attempts, switch to screenshot-driven help. Mosh takes a screenshot, pastes it into chat, Claude reads the UI and gives the next click as text. It's slower than driving the browser, but it is dramatically faster than hunting for an invisible permission prompt.

A single line at the start: **"If I hit permission walls I'll switch to guiding you with screenshots — one click at a time. Don't waste time trying to unblock me."** Mosh will not have to ask why Claude stopped trying.

## Cowork sandbox IP is not stable

Claude's sandbox IP from Cowork shifts — between sessions and sometimes within one. Anything that relies on Claude being allowlisted at a specific IP will fail unpredictably.

Implications:
- Coolify API: keep allowlist wide open (`0.0.0.0/0`), token-protected. See `ZERO_TOUCH_SETUP.md` Path A decision.
- Any firewall rule that allowlists an IP on Mosh's behalf: do not create one that depends on Claude's IP. Use user-side token auth or per-user allowlists only.

## Web-fetch restrictions in Cowork

Cowork's WebFetch/WebSearch tools have content restrictions. When they report a URL is blocked, don't route around them via `curl`/`wget`/Python HTTP libraries — the restrictions apply at the network layer and retry attempts will fail anyway. Either (a) ask Mosh to open the URL himself and paste what's needed, or (b) use a different source.

## What NOT to do when Cowork frustrates you

- Do not silently retry the same blocked tool.
- Do not escalate to bash-level HTTP calls to bypass WebFetch restrictions.
- Do not ask Mosh to re-grant permissions he's already granted.

All three are time-burners that make the session feel worse than the underlying problem.
