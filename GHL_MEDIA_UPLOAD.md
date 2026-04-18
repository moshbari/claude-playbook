# GoHighLevel media library upload

_Last updated: 2026-04-18_

## Why this exists

Every project that needs permanent image/file hosting ends up routing through my GoHighLevel sub-account's media library. AI Image Creator did it. EveryLink does it. Video pages will do it. Each time, we burned hours on the same traps: wrong endpoint, wrong auth, wrong folder parameter, env vars not reaching the container, "the upload worked" but the file landed at root. This file is the final answer so we never rediscover it.

## The one endpoint that works

```
POST https://services.leadconnectorhq.com/medias/upload-file
```

Headers (both required, not optional):

- `Authorization: Bearer <PIT>` — Private Integration Token, format `pit-<uuid>`
- `Version: 2021-07-28` — yes, that literal date string, every request

Multipart form fields:

- `file` — the actual bytes
- `hosted=false` — string, not boolean
- `name=<plain filename>` — just `photo.png`, never `folder/photo.png`
- `altId=<LOCATION_ID>` — the GHL sub-account location id
- `altType=location`
- `parentId=<FOLDER_ID>` — optional, for folder placement (see below)

Response:

```json
{
  "fileId": "...",
  "url": "https://assets.cdn.filesafe.space/<locationId>/media/<uuid>.<ext>"
}
```

Save `url`. That's the permanent CDN link — never expires, publicly accessible, embed it in `<img src>`. `fileId` only matters if you plan to delete the asset later.

## Folder placement — two variants, both work, don't mix them

We have two working implementations that use different field names. Pick one and stick to it.

- **curl path (EveryLink, recommended):** pass `parentId=<FOLDER_ID>` as a **form field**.
- **fetch path (AI Image Creator):** append `?folderId=<FOLDER_ID>` as a **query parameter** on the URL.

What does NOT work: `folderId` as a form field. The API returns HTTP 200 with a real `fileId`, but the file lands at root. We hit this twice — "the upload looks successful, why isn't it in the folder?" is the signature. A 200 response is not proof of correct folder placement.

## curl vs fetch — when to pick each

Node's `fetch()` returns `401 "IAM Service"` on some GHL endpoints even with a valid PIT. It's not a code bug, it's a GHL platform quirk. The pattern:

- Default to `fetch()` (cleaner, typed, no shell).
- If you see the IAM Service 401, swap to `curl` via `child_process.execSync`.
- `/medias/upload-file` happens to work with both in current testing — but I'd lean curl for any project where reliability matters more than code elegance. The EveryLink repo's `src/app/api/ghl-upload/route.ts` is the reference.

## How to find the folder ID (it's not the folder name)

`GHL_FOLDER_ID` must be an actual id, never a name. Get it by listing folders:

```bash
curl "https://services.leadconnectorhq.com/medias/files?altId=<LOCATION_ID>&altType=location&type=folder" \
  -H "Authorization: Bearer <PIT>" \
  -H "Version: 2021-07-28"
```

Match by name in the response, grab the id. Gotcha: some GHL responses expose it as `_id` instead of `id`. Check both fields when parsing.

## Endpoints that do NOT work (each verified by failing)

- `/medias/upload` (no `-file` suffix) — different endpoint, PITs not accepted.
- `/medias/upload-signed-url` — only for conversation attachments.
- `rest.gohighlevel.com/v1/...` — v1 API is dead, 404.
- `backend.leadconnectorhq.com/...` — internal GHL infra, not public.

## Required env vars (every project)

- `GHL_API_KEY` — the PIT token (`pit-<uuid>`)
- `GHL_LOCATION_ID` — sub-account location id
- `GHL_FOLDER_ID` — folder id (optional; root if omitted)

Pick one name per project and stick with it. I've seen `GHL_FOLDER_ID` and `GHL_MEDIA_FOLDER` used interchangeably — don't mix them inside a single codebase.

## Railway / Docker — the env var trap that killed hours

This is the single hardest bug we hit. The code compiles and deploys fine, `isGHLConfigured()` silently returns false at runtime, uploads are silently skipped, no error logs. See `DEBUGGING_DEPLOYMENTS.md` for the verification sequence that catches this.

Specific to GHL:

- **Nixpacks does not install `curl`.** If the project uses curl for uploads, switch to a Dockerfile with `apt-get install -y curl`. A Nixpacks build passes fine, then crashes at runtime the first time a user uploads.
- **Dockerfiles don't inherit Railway env vars at build time.** Declare both `ARG` and `ENV` for every var `next build` reads. If you skip this for Supabase (common), `next build` blows up with "URL and API key required to create Supabase client."
- **docker-compose needs each env var listed explicitly** in the service's `environment:` block. Having them in host `.env` is not enough — that file is only used for `${VAR}` substitution in the YAML.

## Checklist before calling an upload integration shipped

- [ ] `docker exec <app> env | grep GHL` shows all three vars inside the container.
- [ ] A real upload test produces a row in the DB with a `https://assets.cdn.filesafe.space/...` URL (not base64, not the original OpenAI URL).
- [ ] The GHL media library UI shows the file in the **right folder**, not at root.
- [ ] `try/catch` around the upload falls back gracefully to the original URL on failure. A broken GHL integration should degrade, not block the user.

## Reference implementations in our repos

- **curl + form field `parentId` (recommended):** `src/app/api/ghl-upload/route.ts` in `moshbari/everylink`. Writes to `/tmp/ghl-uploads/<uuid>_<safename>`, 2-minute timeout, cleans up in `finally`, 100 MB cap, MIME type whitelist.
- **fetch + query param `folderId`:** `src/lib/ghl.ts` in the AI Image Creator project. Web FormData + Blob, separate helpers for Buffer / URL / base64 inputs.
- **Origin story:** the first working version came from `moshbari/video-processor-railway`'s `ghl-uploader.js` (StepWise project).
