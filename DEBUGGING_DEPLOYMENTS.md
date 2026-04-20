# Debugging deployments — "the code looks fine but something is wrong"

_Last updated: 2026-04-20_

## Why this exists

The worst bugs I've hit on my own projects are the ones where everything looks successful. No error logs. Build passes. Deploy is green. The app even serves pages. But the thing I actually shipped isn't happening — images aren't uploading, webhooks aren't firing, tiers aren't flipping. This is the sequence that flushes them out.

## The pattern — silent failure

The shape is always the same:

1. Code is correct.
2. Build succeeds.
3. Container runs.
4. A branch condition silently skips the feature: missing env var, missing binary, stale container running old code, webhook not registered in the provider.
5. No error is logged because the code didn't "fail" — it just took the "nothing configured, do nothing" path.

If you can't prove the feature ran end-to-end, assume it didn't.

## Verification sequence — run in order, don't skip

Run these in this order after any deploy of a feature that depends on external config.

### 1. Are the env vars actually inside the running container?

```bash
docker exec <container> env | grep <PREFIX>
```

Empty output = env vars didn't make it to runtime. For Docker-based setups, see `NEXTJS_DOCKER_RAILWAY.md` trap 4. For Coolify, verify in the Environment Variables tab and Redeploy (not Restart) after changes.

### 2. Is the code you think is running actually running?

Check the container's start time vs. the most recent commit:

```bash
docker ps                              # shows container age
git log -1 --format='%cI %s'           # last commit time and message
```

If the container is older than the commit, the container is running old code. "It worked before the deploy" means nothing — the old container may still be serving requests. Force a rebuild.

For bundled Next.js / Turbopack output, confirm your string is actually in the bundle:

```bash
docker exec <container> grep -rl '<unique-string-from-new-code>' /app/.next/
```

If it's missing, the build didn't pick up the new code.

### 3. Does the external API actually accept your call?

Before blaming the app, test the third-party API directly from the server with `curl`. If the direct curl fails, the integration has no chance. If the curl succeeds but the app fails, the issue is somewhere between your env vars and your request construction.

### 4. Trigger the feature and tail logs immediately

```bash
docker compose logs <service> --tail=200 | grep -iE '<relevant-keywords>'
```

Silence = the code path didn't run. That's still useful info. It means your trigger didn't reach the code, not that the code is broken.

### 5. Verify the end state, not the intermediate success

HTTP 200 is not proof. `userId` being returned is not proof. What matters:

- For uploads: file appears in the provider's UI, in the correct folder.
- For webhooks: DB row reflects the event.
- For billing: tier flipped in DB AND user sees it after refresh/re-login.
- For emails: the recipient actually got it (check inbox, not just "sent" status).

### The `opaqueredirect` trick for auth-gated redirectors

If you moved storage from local disk to a CDN (S3, GHL, R2) and your app has an auth-gated endpoint that should now 302 to the CDN, this 2-second check tells you whether the migration took. From the browser devtools on the live app, while logged in as a real user:

```js
fetch('/api/download/<id>', { redirect: 'manual', credentials: 'include' })
  .then(r => ({ type: r.type, status: r.status, location: r.headers.get('location') }))
```

- `type: "opaqueredirect"` → the server is 302-ing to a **cross-origin** URL (the CDN host). Migration is live, no bytes going through your server.
- `type: "basic"`, `status: 200` → still serving bytes from disk. Migration didn't take.
- `type: "error"` → the route crashed. Read container logs.

Works because `redirect: 'manual'` on a cross-origin redirect target gives `type: 'opaqueredirect'` with status 0 and no accessible headers — but the *existence* of that type is itself the signal. No container shell, no DB query needed.

## The container age trap (the one that got us)

Hour burned on AI Image Creator: user kept saying "I generated images, they're not in GHL." DB showed newest image at 05:03 UTC. Container was rebuilt at 04:36 UTC. The "new images" were actually from the old container that had served requests for 27 minutes after the rebuild started — racing with the new container as it came up. All those images went through the pre-GHL code path.

Lesson: when the user says "I just did X and it didn't work," always confirm:

- Container start time
- Most recent DB row timestamp
- Current server time
- Latest commit time

If any two are inconsistent, figure out why before debugging the code.

## The copy-paste relay trap

When debugging a project that's deployed on a server Claude can't SSH into, the only way to run diagnostics is via Claude Code on the user's terminal — which means round-tripping commands through chat. Going step-by-step compounds: each diagnostic is a full round-trip.

Better workflow: write **one comprehensive prompt** that runs every diagnostic in sequence and returns everything at once. Copy once, paste once, read the full picture in one response.

Template:

```
SSH into <server> as root. Run all of these and report every output:

1. <diagnostic 1>
2. <diagnostic 2>
3. <diagnostic 3>
...

Do all of them even if an earlier one fails. Share full output of each.
```

## DNS + TLS verification before declaring "done"

The angle where Mosh's browser hits the app is not the angle where my curl hits it. If I confirm a subdomain works from the sandbox and tell Mosh "it's live, try it," his browser may still see NXDOMAIN for up to an hour because of negative-response caching. This sequence catches the gap before the user does.

### DNS from 3 resolvers, not just mine

```bash
for r in 1.1.1.1 8.8.8.8 9.9.9.9; do
  echo "=== $r ==="
  dig +short @$r <subdomain>.<domain> A
done
```

All three should return the expected IP. If one is empty and the others aren't, the user's browser may be hitting the empty one. Wait for propagation or tell the user which resolver to switch to.

### NXDOMAIN negative-cache risk

If you queried the subdomain **before** the DNS record existed, public resolvers cache the negative response for up to the zone's SOA minimum TTL. Namecheap's default SOA minimum is `3601` seconds (~60 min). Cloudflare's is lower (~300s) but still real.

Check the SOA:

```bash
dig +short SOA <domain> | awk '{print $NF}'
```

If the number is 3600+ and the subdomain was queried (and missed) recently, warn the user: "Your browser's DNS may cache the old miss for up to N minutes. Try from a different network, or wait."

### TLS cert issuer — catch Traefik's fallback cert

```bash
echo | openssl s_client -servername <host> -connect <host>:443 2>/dev/null \
  | openssl x509 -noout -issuer -subject
```

- Issuer = `R3` / `R10` / `R11` / Let's Encrypt → real cert, routing works ✅
- Issuer = `TRAEFIK DEFAULT CERT` → Traefik has no route for this host. The cert request fell through to the fallback. Means: container didn't restart after FQDN attach, or Coolify didn't regenerate labels. See `COOLIFY_API_QUIRKS.md`.
- Self-signed / expired → misconfigured TLS.

### Body content, not just status

HTTP 200 with the wrong content is a common mode:

```bash
curl -s https://<host>/ | head -c 200
```

Compare the first 200 bytes to what you expect. If it's Traefik's default 404 page or Coolify's placeholder, the server answered but the app isn't running there.

### Don't say "done" until all four pass

Every "it's working, try it" message should be preceded by:
1. SOA minimum check (warn user if ≥3600 and negative cache is likely)
2. Lookup from at least 3 external resolvers
3. openssl issuer check
4. Body snippet match

If any of these reveal mixed results, warn the user about the specific caching window up front rather than letting them find it and push back.

## Checklist for any "it's not working" thread

Before touching code:

- [ ] Env vars present in the running container?
- [ ] Container newer than the commit that added the feature?
- [ ] Direct API curl from the server works?
- [ ] Feature triggered at least once and logs show the code path ran?
- [ ] End state verified in the destination system (provider UI, DB, inbox), not just response code?
