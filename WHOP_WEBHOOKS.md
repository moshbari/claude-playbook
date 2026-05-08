# Whop webhooks — what actually works

_Last updated: 2026-05-08 (GoNoGo $7 credit pack — first session that hit every Whop webhook trap in one go)_

## Why this file exists

I built a credit-purchase flow on Whop for GoNoGo. Every assumption I made about Whop webhooks was wrong. This file is the corrected mental model so I never lose another half-day to it.

## The four bugs I hit (in order)

1. **Wrong webhook URL configured in the dashboard.** I created the webhook with URL `https://gonogo-api.99dfy.com` (no path). Whop happily POSTed to the root, which my Express app didn't handle → 404 every time. The provider showed 25 "delivered" attempts because the TCP connection succeeded — the 404 was a separate signal I had to dig for.
2. **Wrong signature scheme.** I implemented plain HMAC-SHA256 of the raw body, looking for a `whop-signature` header. Whop actually uses **Svix Standard Webhooks**, which is a totally different scheme.
3. **Metadata URL params silently dropped.** I appended `?metadata[user_id]=<uuid>` to the checkout URL expecting Whop to round-trip it back in the webhook payload. The webhook arrived with `metadata: {}`. The buyer's email was in `data.user.email` though — that's the only reliable handle.
4. **Product-page URL bounced logged-in users.** I used `whop.com/<company>/<product-slug>/?metadata=…` as the checkout URL. For any Whop user who has a relationship with the seller's company, that URL 307-redirects to `/joined/<company>/` (the company hub) instead of showing a buy button. Real customers got "Whop not found" instead of a checkout.

## The right way (locked in)

### Webhook URL

In Whop dashboard → Developer → Webhooks → Create webhook:

- **URL must include the path**: `https://your-api.example.com/whop/webhook` (not just the host)
- **API version**: pick **v1**, not v2. v2 is labeled "deprecated" — Whop will turn it off.
- **Events**: at minimum check `payment.succeeded` and `membership.activated`. Whop fires both for every one-time purchase.
- **Save** → Whop reveals a signing secret of the form `ws_<64 hex chars>`. Copy it into the server's env as `WHOP_WEBHOOK_SECRET`.
- The Whop API key cannot create or list webhooks. This is dashboard-only.

### Signature verification (Svix Standard Webhooks)

Headers Whop sends on every webhook:

| Header | What it is |
|---|---|
| `svix-id` | Unique message id (use this for retry idempotency) |
| `svix-timestamp` | Unix seconds when Whop signed |
| `svix-signature` | `v1,<base64sig>` — possibly multiple space-separated when secret is rotating |

**The signed payload is `${svix-id}.${svix-timestamp}.${raw_body}`** — three pieces joined by literal dots. Not just the body.

**Algorithm:** HMAC-SHA256, output base64-encoded (not hex).

**Whop secret format:** `ws_<hex>`. Strip the `ws_` prefix and **hex-decode** to get the 32 raw key bytes. Don't use the literal string as the key — it'll never verify.

Reference verifier in Node:

```js
import crypto from 'node:crypto';

function verify(rawBody, headers, secret) {
  const id  = headers['svix-id'];
  const ts  = headers['svix-timestamp'];
  const sig = headers['svix-signature'];
  if (!id || !ts || !sig) return false;

  const key = Buffer.from(secret.replace(/^ws_/, ''), 'hex');
  const payload = `${id}.${ts}.${rawBody.toString('utf8')}`;
  const expected = crypto.createHmac('sha256', key).update(payload).digest('base64');

  // svix-signature: "v1,<sig1> v1,<sig2>"
  return sig.split(/\s+/).some(s => {
    const got = s.includes(',') ? s.split(',')[1] : s;
    return got.length === expected.length &&
           crypto.timingSafeEqual(Buffer.from(got), Buffer.from(expected));
  });
}
```

The Express route MUST use `express.raw()` (not `express.json()`) so the body is the byte-exact buffer Whop signed. Mount the raw parser **before** `app.use(express.json())` so it wins for `/whop/webhook`.

### Matching the buyer to a Supabase / app user

`metadata[...]` URL params are dropped. The fields you can rely on in the webhook payload:

- `data.user.email` — the email the buyer used at Whop checkout. Match this against your `user_profiles.email`.
- `data.user.id` — Whop's user id (`user_xxx`). Useful for analytics, not for matching.
- `data.id` — payment id (`pay_xxx`) for `payment.succeeded` events, membership id (`mem_xxx`) for `membership.activated`.
- `data.membership.id` — same membership id as above; **use this as the idempotency key** because both events share it for one purchase.
- `data.plan.id` — which pack was purchased (`plan_xxx`). Map this → credits in your code.

**Critical UX consequence:** users must check out with the same email they used to sign up to your app. Add a banner on the checkout-trigger page: "Credits will be added to: **user@example.com**." If the Whop checkout autofills a different email, credits land on no account (handler should return 200 with `{error: "no_user_match"}` and log the buyer email so admin can manually reconcile).

### Checkout URL — direct-plan, not product-page

| Pattern | Result |
|---|---|
| `whop.com/<company>/<product-slug>/` | 307-redirects logged-in users to `/joined/<company>/` → "Whop not found" for non-members |
| `whop.com/checkout/<plan_id>/` | Standalone checkout form, works regardless of login state ✅ |

So always use `https://whop.com/checkout/${planId}` for the buy button. The plan id is `plan_xxx`, visible in the Whop API response for the product or in the dashboard.

### Idempotency

Whop fires multiple events for one purchase: `payment.succeeded` then one or more retries of `membership.activated`. They share the same `data.membership.id`. Use that as your ledger primary key:

```sql
create table whop_credit_grants (
  payment_id   text primary key,   -- store membership.id here
  user_id      uuid not null references auth.users(id) on delete cascade,
  credits      integer not null check (credits > 0),
  plan_id      text,
  granted_at   timestamptz not null default now()
);
```

In the handler: look up `payment_id` first; if it exists, return 200 with `{credited: false, reason: "already_processed"}`. Don't try to be clever with `INSERT … ON CONFLICT` — explicit idempotency check makes the log trail readable.

## Diagnostic playbook

When a real purchase happens but credits don't land, in order:

1. **Check the Whop dashboard "Recent deliveries" panel** for the webhook. Each delivery shows the request body, your response code, and a Redeliver button. This is the single most useful debug surface — the body shows whether `metadata` came through, which event names Whop is sending, and what the user object looks like.
2. **Check your server logs** for `whop signature verification failed`. If you see it but the dashboard shows 401 responses, your verifier is wrong (most likely the wrong header name or wrong key derivation).
3. **Send a synthetic Svix-signed POST** to `/whop/webhook` with a hand-built payload. If that succeeds and a real Whop delivery still fails, the difference is in the headers — look at `headerNames` your handler logs.
4. **For the already-paid-but-uncredited case**, manually reconcile by inserting a row into the ledger keyed on the real `membership.id` from the dashboard payload, then bumping `runs_limit`. Using the real id (not a placeholder like `manual_recon_xxx`) means future redeliveries get deduplicated, not double-credited.

## Whop platform quirks worth knowing

- **You can't buy your own product.** When you're logged in as the company owner, the "Complete order" button is dead — Whop / Stripe enforce this. Test with an unrelated account or in incognito.
- **Whop creates webhooks at v2 (deprecated) by default.** When you go to add a v1 webhook, the v2 one may already exist and quietly intercept events. List all webhooks before creating a new one. Delete any v2 ones unless you know what they're for.
- **Two webhooks at the same URL is fine functionally** but each gets sent its own copy of every event — wasted compute and double-signature-verification load. Keep one.
- **The Whop API key has narrow default scopes.** Listing/creating/editing webhooks, listing payments, listing memberships, retrieving plans — all require permissions that aren't on by default. Enable in dashboard → API key permissions, then **regenerate the key** for new permissions to take effect.
