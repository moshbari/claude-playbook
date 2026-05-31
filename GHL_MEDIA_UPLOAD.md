# GoHighLevel media library upload

_Last updated: 2026-05-31_

## Why this exists

Every project that needs permanent image/file/video hosting ends up routing through my GoHighLevel sub-account's media library. StepWise did it (videos). AI Image Creator did it (images). EveryLink does it (assets). Each time, we burned hours on the same traps: wrong endpoint, wrong auth, wrong folder parameter, env vars not reaching the container, "the upload worked" but the file landed at root. This file is the final answer so we never rediscover it.

## The one endpoint that works

```
POST https://services.leadconnectorhq.com/medias/upload-file
```

Headers (both required, not optional):

- `Authorization: Bearer <PIT>` — Private Integration Token, format `pit-<uuid>`
- `Version: 2021-07-28` — yes, that literal date string, every request. Leave it out and calls fail silently.

Multipart form fields:

- `file` — the actual bytes
- `hosted=false` — string, not boolean. `hosted=true` with a URL returns HTTP 400 "Unsupported content type". Always upload bytes directly.
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

Save `url`. That's the permanent CDN link — never expires, publicly accessible, embed it in `<img src>` or `<video src>`. `fileId` only matters if you plan to delete the asset later.

Max size: **500 MB for videos, 25 MB for other file types.**

## Getting the three credentials (5-minute setup for any new project)

1. Log into GHL Agency account.
2. Switch to the sub-account you want to use.
3. **Settings → Integrations → Private Integrations → Create Integration.**
4. Name it (e.g., project name).
5. Under Scopes, tick only: **View Medias** and **Edit Medias**.
6. Save. Copy the token that appears (starts with `pit-`).
7. **Location ID** is in your browser URL: `app.gohighlevel.com/v2/location/THIS_PART_HERE/...`
   - **Do NOT eyeball it off a screenshot.** Location ids are random base62 and routinely contain `I`/`l`/`1` and `O`/`0`, which are indistinguishable in most UI fonts. On Book Factory (2026-05-31) we read `...Q`**`l`**`YNa` off a screenshot when the real id was `...Q`**`I`**`YNa`; uploads silently went to root and folder listing 401'd for a full debugging loop. Copy-paste the id from the address bar, or let the API tell you the truth: the upload/list 401 error literally prints it — `"Your token grants access to location <REAL_ID> but you're requesting <WRONG_ID>"`. Trust that string over your eyes.
8. Go to Media Library in that sub-account and create your target folder. Note the exact name.

## Folder placement — one recipe, works for curl AND fetch

There is exactly one reliable way to land a file in a specific folder. Send these three fields together as **multipart form fields** (in addition to `file`, `hosted=false`, `name`):

- `parentId = <FOLDER_ID>`
- `altId    = <LOCATION_ID>`
- `altType  = "location"`

This works whether you use curl-via-execSync or Node `fetch()`. Don't omit `altId`/`altType` — they are required for folder routing on PIT-authed uploads, even though the docs imply they're optional.

What does NOT work (each verified):

- `folderId` as a multipart form field — 200 OK, lands at root.
- `?folderId=<ID>` as a URL query param **without** `altId`+`altType` form fields — also 200 OK, also lands at root. AI Image Creator shipped this twice (commits `5154223` and `4649d61`) and silently uploaded to root for two weeks before anyone noticed.
- The `name` field with a folder path prefix (`folder/photo.png`) — slashes are not directory separators, you just get a literal filename containing a slash.

The signature of the bug: "the upload looks successful, the URL works, but the file isn't in my folder." A 200 response with a real `fileId` is **not** proof of correct folder placement.

Belt-and-suspenders option: also include `?folderId=<ID>` on the URL. It's harmless when present alongside the form fields and rules out one more class of weirdness.

**Do NOT try to upload first and move after.** Every update/move endpoint on `/medias/` returns 404 for PIT tokens. Confirmed failures on `PUT /medias/{fileId}`, `PUT /medias/files/{fileId}`, `PATCH /medias/files/{fileId}`, `POST /medias/files/bulk-update`. Always include `parentId` at upload time.

## curl vs fetch — when to pick each

Node's `fetch()` returns `401 "IAM Service"` on some GHL endpoints even with a valid PIT. It's not a code bug, it's a GHL platform quirk. The pattern:

- Default to `fetch()` (cleaner, typed, no shell).
- If you see the IAM Service 401, swap to `curl` via `child_process.execSync`.
- For `/medias/upload-file` on **Node.js servers**, lean curl. Node's native `FormData` + `fetch` and the `form-data` npm package both fail with `400 "Unexpected end of form"` — they don't format the multipart boundary the way this API expects. curl via `execSync` has been rock solid across StepWise and EveryLink.
- The EveryLink repo's `src/app/api/ghl-upload/route.ts` is the reference implementation.

## How to find the folder ID (it's not the folder name)

`GHL_FOLDER_ID` must be an actual id, never a name. Get it by listing folders.

**Critical:** you MUST pass `type=folder` (or `type=file` when listing files). Leaving `type` out returns `422 "type must be a string, type should not be empty"`.

```bash
curl "https://services.leadconnectorhq.com/medias/files?altId=<LOCATION_ID>&altType=location&type=folder&sortBy=createdAt&sortOrder=desc&limit=100" \
  -H "Authorization: Bearer <PIT>" \
  -H "Version: 2021-07-28"
```

Match by name in the response, grab the id.

**Gotcha that cost us a full debugging session:** the folder id is in `item._id` (with underscore), **not** `item.id`. `item.id` is `undefined`. Always parse as `item._id || item.id` to be safe across response shapes.

## Best practice: resolve the folder id from its NAME at runtime — never hand-hunt it

**The folder id is invisible in the GHL UI.** It is NOT in the URL (the media-library URL is `app.gohighlevel.com/v2/location/<LOCATION_ID>/media-storage` — that's the *location* id, no folder id), NOT on the folder tile, NOT in any panel or breadcrumb. The list-folders call above is the *only* way to get it. This sends every operator — and us — in circles: "which one is the folder id?" There is nothing on screen to copy. (Confirmed again on Book Factory, 2026-05-31: the owner had the right folder open, breadcrumb `Home > bookfactory.99dfy.com`, and still could not find the id, because it is genuinely not shown.)

So **stop asking a human for the folder id.** Ask for the folder **name** — which the user CAN read off the media-library breadcrumb — and resolve the id in code, lazily, on first upload. Cache it for the process lifetime.

Config becomes `GHL_API_KEY` + `GHL_LOCATION_ID` + **`GHL_FOLDER_NAME`**. Keep `GHL_FOLDER_ID` as an optional override that short-circuits resolution (use it only if you have duplicate folder names).

```js
let _folderCache = null;
async function resolveFolderId(c) {
  if (c.folderId) return c.folderId;            // explicit id wins
  if (!c.folderName) return "";                 // no folder → upload to root
  if (_folderCache?.name === c.folderName) return _folderCache.id;
  const url = `https://services.leadconnectorhq.com/medias/files`
    + `?altId=${encodeURIComponent(c.locationId)}&altType=location`
    + `&type=folder&sortBy=createdAt&sortOrder=desc&limit=100`;   // type=folder is mandatory (422 without)
  const data = await curlJson(["-s", "--max-time", "30", url,
    "-H", `Authorization: Bearer ${c.token}`, "-H", "Version: 2021-07-28"]);
  const arr = Array.isArray(data) ? data : (data.files || data.folders || data.medias || []);
  const match = arr.find(it => (it.name || "").toLowerCase() === c.folderName.toLowerCase());
  const id = match ? (match._id || match.id || "") : "";          // _id, not id — see gotcha above
  _folderCache = { name: c.folderName, id };
  if (!id) console.log(`[GHL] folder "${c.folderName}" not found — uploading to root`);  // never fail silently
  return id;
}
```

Feed the resolved id into the upload as `parentId` (with `altId` + `altType=location`, as always).

Why this is the right default:

- **The id is unfindable by a human** — the breadcrumb shows the folder *name*, never the id. Asking for the id guarantees a support loop. The name is the only thing the operator can actually see and copy.
- **One API call, cached** — negligible cost, resolved lazily on the first upload, reused for the process lifetime.
- **Onboarding shrinks to two clicks of input:** the operator pastes the `pit-` token and confirms the folder name they already see on screen. No DevTools, no network-tab spelunking, no id hunting.
- **Reuses the rules above** — `type=folder` mandatory, id is `item._id`. This just wraps them so a human is never in the loop.

Reference: Book Factory's `web/lib/ghl.js` on the Contabo VPS ships exactly this.

## Endpoints that do NOT work (each verified by failing)

- `/medias/upload` (no `-file` suffix) — different endpoint, PITs not accepted.
- `/medias/upload-signed-url` — only for conversation attachments.
- `rest.gohighlevel.com/v1/...` — v1 API is dead, 404.
- `backend.leadconnectorhq.com/...` — internal GHL infra, not public.
- `PUT /medias/{fileId}` and all move/update variants — 404 on PITs. Use `parentId` at upload time instead.
- `@gohighlevel/api-client` SDK `medias.uploadFile()` / `medias.listMediaFiles()` — methods don't exist despite being documented. Skip the SDK, use raw curl or fetch.

## Full failure log (so we don't re-run these experiments)

| # | What we tried | What happened | The fix |
|---|---|---|---|
| 1 | `item.id` for folder ID | Always `undefined` | Use `item._id \|\| item.id` |
| 2 | JSON body for upload | 400 "Unsupported content type" | multipart/form-data only |
| 3 | Node native `FormData` / `form-data` npm package | 400 "Unexpected end of form" | curl via `execSync` |
| 4 | Moving files after upload | 404 on every move/update endpoint | Include `parentId` at upload |
| 5 | `@gohighlevel/api-client` SDK media methods | Methods don't exist | Raw curl / fetch |
| 6 | Listing without `type` param | 422 "type must be a string" | Always include `type=folder` or `type=file` |
| 7 | `hosted=true` URL upload | 400 "Unsupported content type" | Upload file bytes directly |
| 8 | `folderId` as form field, OR `?folderId=` URL query without `altId`+`altType` form fields | 200 OK with real fileId, file lands at root | Send `parentId`+`altId`+`altType=location` as form fields together. Hit this on aipic twice (commits 5154223, 4649d61) — silent for two weeks. |
| 9 | curl `-F 'file=@path'` on `node:20-slim` | INVALID_FILE_TYPE — slim image has no `/etc/mime.types`, curl sends `application/octet-stream` | Set MIME explicitly: `-F "file=@path;type=audio/mpeg"` (or whatever the real type is). Bites any slim base — Alpine, debian-slim, distroless. |

## Symptom → diagnosis lookup

Quick recognition table for the failure modes. If you see the symptom on the left, jump to the fix on the right before debugging from scratch.

| Symptom | Diagnosis |
|---|---|
| Upload returns 200, response includes a real `fileId` and CDN URL, but the file is at the **root** of the media library instead of your folder | You're missing `altId` + `altType` form fields, OR you're using `folderId` (as form field or URL query) instead of `parentId`. Send all three: `parentId`, `altId`, `altType="location"`. |
| `400 Unsupported content type` | Either you sent JSON (must be multipart/form-data) or you sent `hosted=true` with a URL (must be `hosted=false` with bytes). |
| `400 Unexpected end of form` | Node's native `FormData`+`fetch` or the `form-data` npm package — multipart boundary isn't formatted the way GHL expects. Switch to `curl` via `child_process.execSync`. |
| `401 IAM Service` | Node `fetch()` quirk on certain GHL endpoints. Same call works via `curl`. Swap to curl-via-execSync. |
| `401 "Your token grants access to location X but you're requesting Y"` on list/upload | Wrong location id — almost always an `I`/`l`/`1` or `O`/`0` misread from a screenshot. The error prints the REAL id (X); copy it verbatim into `GHL_LOCATION_ID`. Uploads may still succeed (endpoint trusts the token's location) while listing/folder-routing fails — a confusing split symptom. |
| `INVALID_FILE_TYPE` for an obviously valid file | Slim Docker base (`node:20-slim`, Alpine, distroless) has no `/etc/mime.types`, so curl sends `application/octet-stream`. Set MIME explicitly: `-F "file=@path;type=audio/mpeg"`. |
| `422 type must be a string` listing files/folders | Missing `type=folder` (or `type=file`) on the list endpoint. Always include it. |
| Folder ID returns `undefined` when parsing the list response | The id is in `item._id`, not `item.id`. Use `item._id \|\| item.id`. |
| Code/build is fine, no errors, but uploads are silently skipped at runtime | `isGHLConfigured()` is returning false. Env vars didn't reach the container. Check `docker exec <app> env \| grep GHL`. See `DEBUGGING_DEPLOYMENTS.md` step 1. |
| Upload works on your laptop, fails in production | Either MIME (slim base, see above) or env vars not in container. Run the verification sequence in `DEBUGGING_DEPLOYMENTS.md` before assuming the code is wrong. |

## MIME type — must be set explicitly on slim Docker bases

`node:20-slim` (and most slim/Alpine/distroless bases) ship without `/etc/mime.types`. With no lookup table, curl falls back to `application/octet-stream` for every `-F file=@path`. GHL's parser whitelists by MIME and rejects octet-stream with `INVALID_FILE_TYPE`. The traps that make this hard to spot: it works on your laptop (full mime DB), it works in `node:20` (non-slim), and the GHL error message doesn't mention MIME at all.

Always set the type explicitly on the form field:

```bash
-F "file=@/tmp/clip.mp3;type=audio/mpeg"
```

Common audio/video types GHL accepts: `audio/mpeg` (mp3), `video/mp4`, `image/png`, `image/jpeg`, `application/pdf`. **`audio/mp4` for `.m4a` is rejected** — transcode m4a to mp3 before upload (ffmpeg `-c:a libmp3lame`) rather than trying to upload the raw m4a.

Reference: `listen-bizapp-club/lib/ghl.js` hardcodes `;type=audio/mpeg` because it always transcodes to MP3 before upload.

## Required env vars (every project)

- `GHL_API_KEY` — the PIT token (`pit-<uuid>`)
- `GHL_LOCATION_ID` — sub-account location id
- `GHL_FOLDER_NAME` — **preferred:** folder name as shown in the media-library breadcrumb (e.g. `bookfactory.99dfy.com`); the code resolves it to an id at runtime (see "resolve the folder id from its NAME" above). The operator can read this off the screen — the id they cannot.
- `GHL_FOLDER_ID` — optional explicit override; skips name resolution. Use only when folder names collide. Root if both are omitted.

Pick one name per project and stick with it. I've seen `GHL_FOLDER_ID` and `GHL_MEDIA_FOLDER` used interchangeably — don't mix them inside a single codebase.

For VPS projects (like StepWise), we use a `ghl-config.json` file next to the uploader with `chmod 600` permissions instead of env vars. Same three fields: `token`, `locationId`, `folderName` (the module resolves the folder id at runtime).

## Railway / Docker — the env var trap that killed hours

This is the single hardest bug we hit. The code compiles and deploys fine, `isGHLConfigured()` silently returns false at runtime, uploads are silently skipped, no error logs. See `DEBUGGING_DEPLOYMENTS.md` for the verification sequence that catches this.

Specific to GHL:

- **Nixpacks does not install `curl`.** If the project uses curl for uploads, switch to a Dockerfile with `apt-get install -y curl`. A Nixpacks build passes fine, then crashes at runtime the first time a user uploads.
- **Dockerfiles don't inherit Railway env vars at build time.** Declare both `ARG` and `ENV` for every var `next build` reads. If you skip this for Supabase (common), `next build` blows up with "URL and API key required to create Supabase client."
- **docker-compose needs each env var listed explicitly** in the service's `environment:` block. Having them in host `.env` is not enough — that file is only used for `${VAR}` substitution in the YAML.

## Migrating an app from disk to GHL — the universal storage pointer

When a project already stores user files on disk and you're adding GHL after the fact, **do not add a parallel `ghlUrl` column.** Reuse the existing path column (`pdfPath`, `zipPath`, `coverPath`, `filePath`, whatever it's called) and store *either* a URL *or* a disk path in that same field. The read route branches on `startsWith("http")`.

Why this beats the parallel-column approach:

- Legacy rows with disk paths keep working — zero DB migration.
- New rows with URLs skip server I/O — the download endpoint 302s to the CDN.
- One mental model per asset: "this pointer tells you where the file is."
- `tryUploadToGhl()` returns `null` on failure, so the caller does `pdfPath: ghlUrl || localDiskPath` — the disk copy is a free fallback that kicks in if GHL is down or rate-limits.

Write side (after you've built the buffer and written it to disk):

```ts
const ghlUrl = await tryUploadToGhl(buf, safeName, contentType); // null on fail
await prisma.generation.update({
  where: { id },
  data: { pdfPath: ghlUrl || localDiskPath },
});
```

Read side (auth-gated API route):

```ts
const gen = await prisma.generation.findUnique({ where: { id } });
if (!gen.pdfPath) return 404;

// GHL CDN URL — redirect cross-origin, no bytes through our server.
if (gen.pdfPath.startsWith("http://") || gen.pdfPath.startsWith("https://")) {
  return NextResponse.redirect(gen.pdfPath, 302);
}

// Legacy disk path (pre-GHL rows)
if (!fs.existsSync(gen.pdfPath)) return 404;
return new NextResponse(fs.readFileSync(gen.pdfPath), { /* ... */ });
```

StepWise and AI Image Creator each added parallel URL columns when we migrated them — Printables uses this single-column pattern instead, and it's measurably cleaner. Do this on the next migration.

## Verifying a migration landed — the `opaqueredirect` trick

After shipping the migration, run this from the browser devtools on the live app (while logged in as a real user):

```js
fetch('/api/download/<id>', { redirect: 'manual', credentials: 'include' })
  .then(r => ({ type: r.type, status: r.status, location: r.headers.get('location') }))
```

Interpret the result:

- `type: "opaqueredirect"` → the server is 302-ing to a **cross-origin** URL (the GHL CDN at `assets.cdn.filesafe.space`). Migration is live, the endpoint is no longer serving bytes from disk. ✅
- `type: "basic"`, `status: 200` → the endpoint is still reading the file from disk and streaming it through Next.js. Migration didn't take — check env vars, check the update to the read route, check that new uploads are actually writing URLs to the DB.
- `type: "error"` or a network failure → the route crashed. Read the container logs.

Works in 2 seconds, no container shell, no DB query. Add to the post-deploy checklist.

## Checklist before calling an upload integration shipped

- [ ] `docker exec <app> env | grep GHL` shows all three vars inside the container (or `cat /opt/<project>/ghl-config.json` on VPS).
- [ ] A real upload test produces a row in the DB with an `https://assets.cdn.filesafe.space/...` URL (not base64, not the original OpenAI URL).
- [ ] The GHL media library UI shows the file in the **right folder**, not at root.
- [ ] `try/catch` around the upload falls back gracefully to the original URL / local file on failure. A broken GHL integration should degrade, not block the user.
- [ ] Logs prefix every GHL line with `[GHL]` so you can filter: `pm2 logs <app> | grep GHL`.

## Reference implementations in our repos

- **curl + form field `parentId` (recommended, Node on VPS):** `moshbari/video-processor-railway`'s `ghl-uploader.js` (the StepWise origin). Also lives on the VPS at `/opt/stepwise-video/ghl-uploader.js` — **this file exists only on the VPS, never committed to GitHub.** TODO: commit it.
- **curl + form field `parentId` (Next.js serverless):** `src/app/api/ghl-upload/route.ts` in `moshbari/everylink`. Writes to `/tmp/ghl-uploads/<uuid>_<safename>`, 2-minute timeout, cleans up in `finally`, 100 MB cap, MIME type whitelist.
- **curl + form field `parentId` + universal storage pointer (Next.js standalone):** `src/lib/ghl.ts` in `moshbari/printables`. Small module: `isGhlConfigured()`, `resolveFolderId()` (caches for process lifetime), `uploadToGhl()` (throws), `tryUploadToGhl()` (returns null). Called from `src/app/api/generate/route.ts` via `Promise.all` for ZIP + PDF + cover. Read routes (`/api/download/[id]`, `/api/file/[id]/[name]`) branch on `startsWith("http")` and 302 redirect. Reference for any project migrating disk → GHL.
- **fetch + form fields `parentId`+`altId`+`altType` (Next.js, no curl/exec):** `src/lib/ghl.ts` in `moshbari/aipic`. Web FormData + Blob, separate helpers for Buffer / URL / base64 inputs. Use this when the runtime can't shell out to curl (some Node serverless platforms, Cowork sandboxes). Earlier versions of this file used `?folderId=` and silently uploaded to root — fixed 2026-05-03.
