# SaaS launch checklist (billing + auth)

_Last updated: 2026-05-08_ (GoNoGo Whop credit-pack lessons)

## Why this exists

Every item on this list is a mistake I've actually made. First real test of Printables: a $1 test purchase didn't flip the user to PRO. Root cause: the webhook handler was in the code, but the webhook was never **registered** in the Whop dashboard. App looked fine, checkout worked, tier stayed FREE. Cost hours to debug.

The rule: don't trust "code is deployed." Prove it with a real, discounted, end-to-end purchase.

## Billing webhook (Whop, Stripe, LemonSqueezy, Paddle)

1. **Register the webhook in the provider's dashboard.** Code being deployed is not enough. The provider has to know to POST to your URL. Verify the URL matches production exactly — no staging, no localhost, no trailing slash mismatch.
2. **Copy the webhook signing secret into env vars.** Every time the webhook is (re)created, the provider generates a new secret. Paste that exact secret into the server's env (e.g. `WHOP_WEBHOOK_SECRET`).
3. **Match event names exactly.** Providers differ: Whop uses underscores (`membership_activated`, `invoice_paid`, `membership_deactivated`). Stripe uses dots (`customer.subscription.created`). Confirm which strings the provider actually sends before writing the handler.
4. **Match users by metadata.userId AND email fallback.** Stamp `userId` into checkout metadata at session creation. In the webhook, try metadata first, then fall back to email. Email fallback saves users who buy in a different browser or before metadata wiring lands.
5. **Do a real end-to-end test with a real charge.** Use a $1 test price. Watch the webhook fire in the provider's dashboard, watch the server log the receipt, watch the user's tier flip in the DB. If any of those three doesn't light up, it's not shipped.

## Coolify deploys

6. **Env var changes require Redeploy, not Restart.** Restart reuses the old container image with old envs baked in. Redeploy triggers a Docker rebuild that picks up the new envs.
7. **Build-time vs runtime env vars.** `NEXT_PUBLIC_*` and any value inlined at build time won't update on Redeploy unless the build actually runs again. Triple-check the Coolify build logs show the new value.
8. **GitHub → Coolify auto-deploy is not on by default.** After a `git push`, Coolify may not redeploy unless you wire the webhook in Coolify → Git Source settings. Until that's set up, you have to click Redeploy in the Coolify UI manually.

## Account UX (grandma test)

9. **Always show which account is logged in.** Email visible somewhere persistent — header avatar tooltip, dashboard, settings page. Users need to verify they're in the right account without digging.
10. **Always include a Sign-out button.** Visible, reachable from every page. Don't assume users never want to switch accounts.
11. **If using magic-link auth (passwordless), say so up front.** Users look for "Password" and "Change password" fields and panic when they're missing. Put a short line on the sign-in page: "No password needed. We'll email you a link."

## Session & tier changes

12. **JWT sessions are frozen until re-login.** When you flip a user's tier in the DB manually, their existing session still reports the old tier until they sign out and back in. After any manual DB tier change, tell the user to re-login.
13. **Webhook-driven tier changes also need session refresh awareness.** If a user is looking at the page when the webhook fires, they may need to refresh or re-login to see the new tier. Either set short JWT TTL or re-fetch tier server-side on dashboard.

## Observability

14. **Log every webhook receipt.** At minimum: event type, event id, matched user id (or "no match"), tier change applied. Makes debugging trivial when a customer says "I paid but don't have access."
15. **Keep a provider-dashboard bookmark.** Whop / Stripe / Paddle all have a "recent webhook deliveries" view with response codes. First stop when anyone reports billing issues.



## Whop specifically

This list grew out of Printables (where the webhook wasn't registered). GoNoGo added four more lessons in one session — see `WHOP_WEBHOOKS.md` for the full story. Quick summary:

16. **Whop uses Svix Standard Webhooks, not custom HMAC.** Headers are `svix-id`, `svix-timestamp`, `svix-signature`. Signed payload is `${id}.${timestamp}.${body}`. Secret format `ws_<hex>` — hex-decode to 32 raw key bytes. Don't write a custom `whop-signature` verifier; copy the working one from `WHOP_WEBHOOKS.md`.
17. **Webhook URL must include the full path.** Whop dashboard accepts `https://api.example.com` (no path) and reports "delivered" because the TCP connection succeeds — but your app sees 404. Always set the URL to `https://api.example.com/whop/webhook` or whatever the route is.
18. **`?metadata[user_id]=...` URL params are dropped silently.** They never reach the webhook. Match buyers by `data.user.email` instead, and tell users on the checkout-trigger page which email the credits will land on.
19. **Use `whop.com/checkout/<plan_id>` as the buy URL, not the product page.** The product page redirects existing-relationship users to the company hub and shows "Whop not found" instead of a buy button.
20. **Whop fires both `payment.succeeded` and `membership.activated` for the same purchase.** Use `data.membership.id` as the idempotency key — it's shared across both events.
21. **Whop creates webhooks at v2 (deprecated) by default.** Pick v1 in the dashboard. List all webhooks before creating a new one — there's likely already one pointing at a wrong URL intercepting events.
22. **Whop API key permissions matter and don't auto-apply.** Listing payments, listing webhooks, retrieving plans — each needs a separate permission. After enabling in the dashboard, **regenerate the key** for the new scopes to take effect.

## The new diagnostic rule

23. **First stop when a webhook isn't crediting users: the provider's "recent deliveries" panel.** Whop / Stripe / Paddle all have one. It shows the actual request body, your response code, and a Redeliver button. The provider knows what it sent — your logs only know what arrived. Disagreements between those two are the bug.
24. **Send a synthetic signed POST before declaring the integration broken.** If your handler accepts a hand-built valid signature but rejects the real Whop deliveries, the bug is in header parsing or secret derivation, not in the verification math. If your handler rejects the synthetic too, the bug is in the math.

## The 5-minute smoke test before any launch

- [ ] Sign up with a throwaway email.
- [ ] See my email in the header or dashboard.
- [ ] Click Sign out. Confirm I'm back to the homepage.
- [ ] Sign in again with the same email via magic link.
- [ ] Buy the $1 test tier.
- [ ] Provider dashboard shows a 200 OK webhook delivery.
- [ ] Server logs show the webhook received and user upgraded.
- [ ] DB shows the user is PRO.
- [ ] Refresh or re-login on the app — see PRO badge.
- [ ] Cancel the subscription in provider dashboard.
- [ ] DB shows user back to FREE.

If any box is unchecked, the SaaS isn't ready.
