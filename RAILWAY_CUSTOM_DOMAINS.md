# Railway — custom domains, the SSL trap, and the project-token model

_Last updated: 2026-05-23_

## Why this exists

Railway's custom-domain flow has TWO undocumented traps that cost us most of a day on the UOM AI Coach project. Both are easy to avoid once you know about them. This file is the next-project shortcut.

## Trap 1 — `CERTIFICATE_STATUS_TYPE_VALIDATING_OWNERSHIP` is not a "wait longer" state

Railway changed their custom-domain flow some time in late 2025 / early 2026. They now require a **TXT record for verification** before Let's Encrypt is even attempted. The cert sits in `VALIDATING_OWNERSHIP` forever until that TXT exists.

The new requirement is barely mentioned in their UI. The first time you hit it you'll think it's a Let's Encrypt rate-limit, a DNS issue, or a Railway bug. None of those. It's the missing TXT.

**Detect via the GraphQL API:**

```graphql
query {
  domains(projectId: "...", environmentId: "...", serviceId: "...") {
    customDomains {
      domain
      status {
        verificationDnsHost      # e.g. "_railway-verify.aicoach"
        verificationToken        # e.g. "railway-verify=cff02859ba..."
        certificateStatus        # "CERTIFICATE_STATUS_TYPE_VALIDATING_OWNERSHIP" when waiting
      }
    }
  }
}
```

If `verificationToken` is non-null, the user must add a TXT record at `_railway-verify.<subdomain>` with the token value as-is (including the `railway-verify=` prefix).

**Tell the user:**

```
Type: TXT
Host: _railway-verify.<their-subdomain>          ← starts with underscore, no domain suffix
Value: railway-verify=<long-token-from-API>       ← include the prefix
TTL: Automatic
```

For Namecheap users specifically: the Host field is relative — they enter `_railway-verify.aicoach` (NOT `_railway-verify.aicoach.ultimateonlinemastery.org`). Namecheap appends the domain automatically.

Cert issuance completes within 5-15 minutes after the TXT propagates.

## Trap 2 — `customDomainUpdate(targetPort: ...)` breaks routing if you guess wrong

If you call the API and set `targetPort` to a port the app isn't actually listening on, traffic to the custom domain returns 502 Bad Gateway from Railway's edge ("Application failed to respond"). The Railway-supplied URL keeps working fine because it uses Railway's auto-detection.

The fix is to set `targetPort` to the port Railway actually exposes for your service. For most Node services on Railway that's **8080** (Railway sets `PORT=8080` in the container by default — your code does `const PORT = process.env.PORT || 3000` and ends up on 8080 in prod even though you THINK it's on 3000).

Recovery call:

```graphql
mutation {
  customDomainUpdate(id: "...", environmentId: "...", targetPort: 8080)
}
```

Better default: **don't pass `targetPort` at all** when creating or updating. Let Railway auto-detect. Only set it if you intentionally bind to a non-default port.

## Trap 3 — Project tokens cannot connect GitHub repos

There are two Railway token types:

| Token | Header | What it can do | What it cannot do |
|---|---|---|---|
| **Personal Access Token** | `Authorization: Bearer <token>` | Everything in the account | — |
| **Project Token** | `Project-Access-Token: <token>` | Deploys (CLI + API), env vars, custom domains, basic mutations within ONE project | GitHub repo connection, project creation, anything cross-project |

For "Mosh gives me one token, I do everything in his project" the right choice is a **Project Token** scoped to the specific project. He generates it at https://railway.com/account/tokens. Auth header is `Project-Access-Token`, NOT `Authorization: Bearer`.

If you try `Authorization: Bearer <project-token>` you get `"Not Authorized"` errors. Easy to misdiagnose as a bad token.

## Useful Railway GraphQL endpoints (project-token authorized)

```bash
ENDPOINT="https://backboard.railway.com/graphql/v2"
HDR="Project-Access-Token: $RAILWAY_TOKEN"

# Verify the token works
curl -s -X POST $ENDPOINT -H "Content-Type: application/json" -H "$HDR" \
  -d '{"query":"query { projectToken { projectId environmentId } }"}'

# List services in the project
curl -s -X POST $ENDPOINT -H "Content-Type: application/json" -H "$HDR" \
  -d '{"query":"query { project(id: \"PROJ\") { services { edges { node { id name } } } } }"}'

# Read all env vars on a service
curl -s -X POST $ENDPOINT -H "Content-Type: application/json" -H "$HDR" \
  -d '{"query":"query { variables(projectId: \"PROJ\", environmentId: \"ENV\", serviceId: \"SVC\") }"}'

# Add a service-level public domain (returns the generated *.up.railway.app)
curl -s -X POST $ENDPOINT -H "Content-Type: application/json" -H "$HDR" \
  -d '{"query":"mutation { serviceDomainCreate(input: { serviceId: \"SVC\", environmentId: \"ENV\" }) { domain } }"}'

# Add a custom domain (returns the CNAME target the user must point to)
curl -s -X POST $ENDPOINT -H "Content-Type: application/json" -H "$HDR" \
  -d '{"query":"mutation { customDomainCreate(input: { projectId: \"PROJ\", serviceId: \"SVC\", environmentId: \"ENV\", domain: \"app.example.com\" }) { id domain status { dnsRecords { requiredValue } verificationToken verificationDnsHost } } }"}'

# Force-refresh a custom domain (won't fix VALIDATING_OWNERSHIP — only the TXT will)
curl -s -X POST $ENDPOINT -H "Content-Type: application/json" -H "$HDR" \
  -d '{"query":"mutation { customDomainUpdate(id: \"DOM\", environmentId: \"ENV\") }"}'
```

## CLI authentication with a project token

```bash
export RAILWAY_TOKEN=<project-token>
railway up --service <service-name> --detach    # works
```

Note `railway login` will say "Not logged in" even when the env-var auth works for the actual commands. Ignore that — `railway up` and `railway variables` work fine with the env var alone.

## Custom-domain creation flow (full sequence, no clicks for the user)

Assuming the user gives you (1) a project token and (2) a domain they want to use:

1. `customDomainCreate` — Railway returns a CNAME target like `xxxxxxxx.up.railway.app` AND a `verificationToken` / `verificationDnsHost`.
2. Tell the user to add BOTH at their DNS provider:
   - CNAME at `<subdomain>` → the value Railway returned
   - TXT at `_railway-verify.<subdomain>` → the verification token (include the `railway-verify=` prefix)
3. Wait for both to propagate (5-30 min). Verify with `dig` from multiple resolvers.
4. Railway auto-issues Let's Encrypt cert within minutes of both records being live.
5. Test: `curl -sSI https://<domain>/healthz` should return 200 and the cert subject should match the domain (not `*.up.railway.app`).

**Symptom if you skip step 2-TXT:** `CERTIFICATE_STATUS_TYPE_VALIDATING_OWNERSHIP` stuck forever. Browser shows "Not Secure" with the Railway wildcard cert. Adding the TXT and waiting fixes it.

**Symptom if you set wrong targetPort:** SSL works (green padlock) but every request returns 502 "Application failed to respond." Set targetPort=8080 or remove it entirely.

## Quick checklist for any new Railway custom domain

- [ ] Create domain via `customDomainCreate` API
- [ ] Capture BOTH `dnsRecords[].requiredValue` (CNAME) AND `verificationToken`/`verificationDnsHost` (TXT) from response
- [ ] Give user the two records to add at DNS
- [ ] DO NOT set `targetPort` unless needed
- [ ] Poll `certificateStatus` until it leaves `VALIDATING_OWNERSHIP`
- [ ] Test cert with `openssl s_client` to confirm subject matches the domain
- [ ] Test live URL returns 200

## Trap 4 — `serviceCreate` from a GitHub repo does NOT arm push-to-deploy

Creating a service with `serviceCreate(input:{ projectId, name, source:{ repo:"owner/name" } })` builds once from the repo, but it does **not** create a deploy trigger. `service(id){ repoTriggers }` comes back empty, and future `git push` does nothing. You must explicitly create the trigger:

```graphql
mutation { deploymentTriggerCreate(input:{
  projectId:"...", environmentId:"...", serviceId:"...",
  provider:"github", repository:"owner/name", branch:"main"
}){ id } }
```

Verify with `service(id){ repoTriggers { edges { node { repository branch } } } }`, then prove it end-to-end: push a trivial change (e.g. add `/healthz`) and confirm a NEW deployment appears that you didn't trigger, and the new route responds live. (FindMyBuyers, Jun 2026.)

## Trap 5 — variable write via the account Bearer token can silently no-op; use the CLI

The Railway CLI stores a live account access token at `~/.railway/config.json` → `user.accessToken` (Bearer, ~1h TTL, refreshable). It works for `me`, `projectCreate` (needs `workspaceId` from `me { workspaces { id } }`), `serviceCreate`, `volumeCreate`, `serviceDomainCreate`, `deploymentTriggerCreate`. BUT writing env vars over it is flaky: `variableCollectionUpsert` returned `true` yet persisted nothing, and `variableUpsert` returned `"Problem processing request"`. What worked reliably: the CLI —

```bash
railway link --project <id> --environment production --service <name>
railway variables --set "KEY=VALUE" --set "K2=V2" --service <name>
```

Then `railway variables --service <name>` to confirm, and redeploy. Gotcha that wastes time: your app reads `process.env.DATA_DIR`, but Railway only auto-sets `RAILWAY_VOLUME_MOUNT_PATH`. You must set `DATA_DIR=/data` yourself or the JSON DB lands on the ephemeral disk and the admin seeds with default creds. A bare one-word `<name>.up.railway.app` rename is rejected — Railway gives `<service>-production.up.railway.app`. (FindMyBuyers, Jun 2026.)
