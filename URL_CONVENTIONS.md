# URL conventions

_Last updated: 2026-04-20_

## Default: flat subdomains, not nested

For any multi-tenant app where users get their own URL (link-in-bio pages, hosted guides, child apps, tenant dashboards), put tenants at the **first subdomain level of the parent brand domain**, not nested under the app.

Good:

- `heychatmate.com` (marketing) + `app.heychatmate.com` (dashboard) + `<guide>.heychatmate.com` (user-published content)
- `bizapp.club` (marketing / apps.bizapp.club dashboard) + `<name>.bizapp.club` (child apps — note: flat, not `<name>.apps.bizapp.club`)
- `everylink.click` + `<user>.everylink.click` (bio pages)

Avoid:

- `<name>.apps.bizapp.club` — three dots feel suspicious to non-technical users ("this URL has too many parts, is it a phishing site?")
- `<name>.dashboard.brand.com` — same problem

## Why

A 2026 user test on the apps.bizapp.club rewrite: Mosh's direct feedback was _"this kind of URL is not regular and a lot of users will feel risky clicking on this kind of link. I mean, there are literally three dots in the URL."_ He pointed to `heychatmate.com`'s pattern (`<guide>.heychatmate.com`) as the gold standard and asked for a full refactor.

This isn't just aesthetic — nested subdomains trigger the "too complex to be real" heuristic that phishing-aware users rely on. If your user base includes non-developers (and it usually does for SaaS), default flat.

## How to execute

At the infrastructure layer:

1. **Wildcard DNS at the flat level.** `*.brand.com → <server-ip>`, not `*.app.brand.com`. Otherwise Traefik / nginx / whatever can't route the subdomain to your app.
2. **App-side Host-header routing reads the first label before the parent domain.** If the parent is multi-word (e.g. `brand.com` has 1 label to the left of the TLD? No, it's 2: `brand` + `com`). Count labels from the right of the FQDN and pull everything left of the parent's labels.
3. **Reserve infra subdomains.** `www`, `mail`, `api`, `admin`, `ftp`, `smtp`, `ns1`, `ns2`, plus any dashboard sub (`app`, `apps`). Match these first and 404 / route-elsewhere before the tenant logic runs.
4. **Separate `PARENT_DOMAIN` (dashboard host) from `APP_DOMAIN` (tenant suffix)** as env vars. They often differ — `apps.bizapp.club` is the dashboard, `bizapp.club` is the tenant suffix. One env var can't represent both.

## When nested IS the right call

Only two cases I've found worth the three-dot cost:

1. **B2B white-label where customers bring their own domain anyway.** The `<customer>.yourapp.com` default is disposable because serious customers move to their own domain; ugliness is acceptable for the default.
2. **Internal / staff-only tools.** `logs.observability.company.com` is fine because the users are engineers who read URLs like DNS records.

Consumer-facing, public-linkable tenants → always flat.

## Decision rule

Before shipping any URL structure: ask yourself "would my grandma trust clicking this link in an email?" (See `UX_CLICK_ONLY.md` for the general grandma test.) Three+ dots usually fail that test. One dot → `.brand.com` passes.
