# Coolify API quirks

_Last updated: 2026-04-20_

## Why this exists

Every time we try to drive Coolify via its REST API, we hit the same field-name and sequencing traps. Coolify's OpenAPI-generated docs look authoritative but lie about half of these. This file is the ground truth — every item here was verified by a failing API call on apps.bizapp.club.

## Server-side env the API needs

- `COOLIFY_URL` = `http://<server-ip>:8000`
- `COOLIFY_API_TOKEN` — generate in Coolify UI → Keys & Tokens. Bearer-style.
- Token lives in `secrets.txt` on Mosh's Mac. **Never commit it anywhere.**

Every request:

```
Authorization: Bearer <COOLIFY_API_TOKEN>
Content-Type: application/json
```

## API IP allowlist — the first wall you hit

Settings → API → Allowed IPs. Default is server-local ranges only:

```
127.0.0.1,172.16.0.0/12,10.0.0.0/8
```

External clients (Claude's sandbox, GitHub Actions, anywhere off-box) get connection refused until you widen it.

**On Mosh's server the allowlist is intentionally `0.0.0.0/0`** (decided 2026-04-20 — see `ZERO_TOUCH_SETUP.md` "Path A"). Do NOT prompt Mosh to revert it. The bearer token is the auth boundary; IP is not. Deploys happen via GitHub App webhook so the token is only used for setup calls (app creation, env vars, domain attaches), and those calls happen from whatever surface Claude is on.

## Update an application's domain (the one that cost the most time)

`GET /api/v1/applications/{uuid}` returns the domain field as `fqdn`. `PATCH /api/v1/applications/{uuid}` expects the field as `domains` — passing `fqdn` returns `"This field is not allowed."`

```json
PATCH /api/v1/applications/{uuid}
{
  "domains": "https://a.example.com,https://b.example.com"
}
```

- Comma-separated string, not array.
- Each entry **must** have `https://` prefix (or Coolify treats it as invalid).
- No trailing slashes.

After PATCHing domains, Traefik labels on the container are **not regenerated automatically**. The DB has the new FQDN; the running container still answers only on the old one. Symptom: 503 "no available server" or the default Traefik cert.

Fix — call restart explicitly:

```
POST /api/v1/applications/{uuid}/restart
```

Or trigger a redeploy (slower but also works). Restart is faster.

## Deploy

```
POST /api/v1/deploy?uuid=<app-uuid>&force=true
```

Returns a `deployment_uuid`. Poll:

```
GET /api/v1/deployments/{deployment_uuid}
```

`status` values: `in_progress`, `finished`, `failed`. Don't assume success from the initial POST — Coolify queues the deploy and failures show up only in the deployment record.

## Env vars

```json
POST /api/v1/applications/{uuid}/envs
{
  "key": "FOO",
  "value": "bar",
  "is_preview": false,
  "is_literal": false
}
```

Do **NOT** include `is_build_time` — it's documented in some places but the API rejects the payload with a validation error. If you need build-time vars (next build, etc.), declare them as ARG+ENV in your Dockerfile instead of toggling a Coolify flag.

### Multi-line env var values break Docker's parser at build time

This bit on the ShareZPresso APNs key (2026-05-25). Coolify injects EVERY env var as a Dockerfile `ARG` instruction at build time, even when you don't ask for build-time. A multi-line value (e.g. a `.p8` PEM, an SSH key, a multi-line cert) makes `ARG FOO='-----BEGIN…` open a quote that never closes, and the next line of the PEM becomes an "unknown instruction" — Docker fails the whole build in <10 seconds. The Coolify API doesn't surface this error; you have to read the live deploy log in the Coolify UI.

Symptoms:
- Deploy fails consistently in 6–10 seconds
- `GET /api/v1/deployments/{uuid}` returns `status: failed` with NO logs
- Other apps on the same Coolify deploy fine
- UI deploy log shows `dockerfile parse error on line N: unknown instruction: <garbled string>`

Fix: never put a multi-line value in a Coolify env var. Base64-encode it instead and decode in your app:

```js
// In your app code (Node example):
const raw = process.env.MY_PEM_BASE64
  ? Buffer.from(process.env.MY_PEM_BASE64, 'base64').toString('utf8')
  : process.env.MY_PEM || '';
```

```bash
# When setting the env var:
base64 < my.p8 | tr -d '\n' | pbcopy
# Then paste as the single-line value of MY_PEM_BASE64 in Coolify.
```

The `is_literal: true` flag does NOT save you here — Coolify still injects as ARG regardless, and ARG values cannot span multiple lines no matter how they're quoted.

### Stop+Start cycle on a broken deploy state can take the running container down

If a deploy keeps failing and you try `POST /applications/{uuid}/stop` to clear stuck state, the OLD running container is terminated and the new builds keep failing for the same reason — you've now lost your last working container with no way to revive it via API short of fixing the underlying problem. Lesson: diagnose the failure BEFORE trying stop/start. Read the deploy log in the Coolify UI first.

## Storage / volumes

`type` only accepts `persistent`. Every other value returns validation error:

- `volume` → rejected
- `bind` → rejected
- `file` → rejected
- `docker-volume` → rejected
- `named-volume` → rejected

Use `persistent` and specify `host_path` + `mount_path`.

## Webhook-based deploys

`POST /webhooks/deploy` on an application checks `X-Hub-Signature-256` against a per-application secret. Unsigned payloads return `"Invalid signature."` If you want push-to-deploy without solving HMAC-in-Claude's-sandbox, install the **Coolify GitHub App** on the repo (Coolify → Sources → Add → GitHub App). It handles the signature automatically and removes the need for API-token deploys forever.

## Summary — the quirks table

| # | What the docs / intuition say | What actually works |
|---|---|---|
| 1 | PATCH uses `fqdn` | Use `domains` (comma-separated, `https://` prefixes) |
| 2 | Changing domain activates it immediately | Need explicit restart or redeploy for Traefik to pick it up |
| 3 | Env var payload takes `is_build_time` | Omit it; use Dockerfile ARG+ENV for build-time |
| 4 | Storage `type` can be `volume`/`bind`/etc. | Only `persistent` |
| 5 | Deploy response tells you success | Poll `/deployments/{uuid}` — initial POST only queues |
| 6 | Webhook deploys are simple | They require a per-app signature; use the GitHub App instead |
| 7 | API is accessible from anywhere with a valid token | IP allowlist blocks external clients by default |

## Reference

First cracked on the apps.bizapp.club project (Apr 2026). The flat-URL migration and the wildcard Traefik fix exercised almost every quirk in this file.
