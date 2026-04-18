# Next.js + Docker + Railway — build and deploy pitfalls

_Last updated: 2026-04-18_

## Why this exists

Switching a Next.js Railway project from Nixpacks to a Dockerfile — which we do any time we need a tool Nixpacks doesn't ship (curl, specific system libs) — breaks the build in the same two ways every time. And then once the build works, the container runs with missing env vars and silently misbehaves. This file is the fix list.

## Trap 1 — Build-time env vars don't reach the Dockerfile

Nixpacks injects Railway env vars into the build automatically. A Dockerfile doesn't. Anything `next build` reads (Supabase URL, API keys for SSG pages, `NEXT_PUBLIC_*` values) must be explicitly passed.

Declare both `ARG` and `ENV` in the Dockerfile, before the `RUN npm run build` step:

```dockerfile
ARG NEXT_PUBLIC_SUPABASE_URL
ARG NEXT_PUBLIC_SUPABASE_ANON_KEY
ARG GHL_API_KEY
ARG GHL_LOCATION_ID
ARG GHL_FOLDER_ID

ENV NEXT_PUBLIC_SUPABASE_URL=$NEXT_PUBLIC_SUPABASE_URL
ENV NEXT_PUBLIC_SUPABASE_ANON_KEY=$NEXT_PUBLIC_SUPABASE_ANON_KEY
ENV GHL_API_KEY=$GHL_API_KEY
ENV GHL_LOCATION_ID=$GHL_LOCATION_ID
ENV GHL_FOLDER_ID=$GHL_FOLDER_ID

RUN npm run build
```

Then in Railway, set `Build Arg` for each one (not just a runtime env var). A runtime-only env var will not be present during `next build`.

**Symptom if you skip this:** build fails with `"URL and API key required to create Supabase client"` or similar "required env var missing" errors during static page generation.

## Trap 2 — Module-level service clients

Any file that does this at the top level breaks the build:

```ts
// DON'T — evaluated at import time during build
const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_KEY!);
```

If the env vars aren't present during build (which happens on every cold Dockerfile build until you fix trap 1), the client constructor throws and the build dies.

Fix: lazy-init. Only construct inside the request handler:

```ts
// DO — evaluated at request time, env vars guaranteed present
function getSupabase() {
  return createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_KEY!);
}

export async function GET() {
  const supabase = getSupabase();
  // ...
}
```

When converting a project to a Dockerfile, grep for `createClient(` across the repo and wrap every top-level call. Applies to Supabase, Prisma, Stripe, any SDK that touches env at construction.

## Trap 3 — Nixpacks is missing tools you assumed were there

Nixpacks does **not** install `curl`, `ffmpeg`, `imagemagick`, or most system binaries by default. If the code calls out to one of these via `child_process.execSync`, you'll get "command not found" at runtime — with no warning during deploy.

When the code depends on a system binary:

1. Switch to a Dockerfile (Nixpacks can't reliably install arbitrary apt packages).
2. `apt-get update && apt-get install -y <tool>` in the Dockerfile.
3. Redeploy on Railway.

This is why the GHL upload integration uses a Dockerfile — it shells out to `curl`.

## Trap 4 — Silent runtime env var misses (docker-compose only)

This one is not Railway-specific but bites on any Docker project. Having `FOO=bar` in a host `.env` file does NOT put `FOO` into a running container. The host `.env` is only used by docker-compose for `${VAR}` substitution in the YAML.

Every env var the container needs must be listed in the service's `environment:` block:

```yaml
services:
  app:
    environment:
      GHL_API_KEY: ${GHL_API_KEY}
      GHL_LOCATION_ID: ${GHL_LOCATION_ID}
      GHL_FOLDER_ID: ${GHL_FOLDER_ID}
```

Verify after deploy:

```bash
docker exec <app-container> env | grep GHL
```

Empty output = the var never reached the container. Runtime code that branches on missing env (`if (!apiKey) return`) will silently skip features with no error logs anywhere.

## Quick self-audit checklist

Before marking any Dockerized Next.js deploy as "shipped":

- [ ] Every env var `next build` reads is declared as `ARG` + `ENV` in the Dockerfile and set as Build Arg in Railway.
- [ ] `grep -rn "createClient(" src/` shows no top-level calls — all inside functions.
- [ ] `docker exec <container> env | grep <PREFIX>` shows every expected runtime env var.
- [ ] First real request exercises the code path that uses env vars, and the logs confirm it ran (not just "no error").
- [ ] Any system binary the code calls (`curl`, `ffmpeg`, etc.) is installed in the Dockerfile.
