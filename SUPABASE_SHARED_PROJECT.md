# Supabase — when one project hosts multiple apps

_Last updated: 2026-05-23_

## Why this exists

Mosh's Supabase Pro account has ONE project (`yuqeqricenygpsrdfswx`) shared across several apps (GoNoGo, FreedomClock, UOM AI Coach, etc.). That sharing is fine — but several Supabase settings are **project-level, not app-level**, and quietly screwing those settings to make one app work will break the others. This file is the avoid-list.

## What's shared at the project level (and will collide across apps)

- **Custom SMTP** — one sender configured per project. Change it for UOM, GoNoGo's emails start coming from UOM's sender.
- **Site URL** — one global default redirect for ALL auth emails. Change it, every app's default redirect changes.
- **Email templates** — magic link / confirmation / recovery templates are shared.
- **Auth providers** — Google/GitHub/etc OAuth config is shared.
- **Database / `public` schema** — every app's tables sit here together.

## What's safe to change per app

- **Redirect URLs allow-list** — ADD entries here freely. Removing affects the app you removed. Adding doesn't break anyone else.
- **Tables with a unique prefix** — `uom_profiles`, `gonogo_users`, etc. Don't dump everything into `users` / `profiles` / `accounts` in a shared project. Collision is guaranteed.
- **RLS policies on YOUR tables** — scoped to the table.
- **Triggers / functions on YOUR tables** — scoped.

## Rules for any app added to a shared Supabase project

### 1. Prefix EVERY table you create

```sql
-- Bad in a shared project:
create table profiles (...);

-- Good:
create table uom_profiles (...);
```

Same goes for functions, triggers, and policies — prefix all of them. The cost is tiny. The collision risk is enormous if two apps both add a `users` or `profiles` table.

### 2. Always add an explicit INSERT policy on your profile-equivalent table

The `auth.users` → `public.X_profiles` trigger pattern looks clean but FAILS SILENTLY in production. Common reasons:
- Supabase locked down `auth.users` trigger creation on newer accounts
- `security definer` function fails for non-obvious permission reasons
- Race condition: client tries to create profile before trigger fires

**Always add an INSERT RLS policy as a safety net so the client-side fallback works:**

```sql
create policy "uom p i" on public.uom_profiles 
  for insert with check (auth.uid() = id);
```

Without this, sign-in loops silently — user authenticates, profile insert blocked by RLS, loadProfile throws, app shows "loading..." forever.

### 3. Don't touch Custom SMTP. Bypass it entirely.

Trying to "use this project's SMTP for the new app" steals the sender from whatever app was using it before. Don't do it.

**Bypass pattern — generate magic links yourself, send emails yourself via Resend (or any provider):**

```js
// server.js — runs with SUPABASE_SERVICE_ROLE_KEY env var
const sbAdmin = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
  auth: { autoRefreshToken: false, persistSession: false }
});

app.post('/api/auth/magic-link', async (req, res) => {
  const { email } = req.body;
  // Supabase generates the link but does NOT send the email
  const { data } = await sbAdmin.auth.admin.generateLink({
    type: 'magiclink',
    email,
    options: { redirectTo: 'https://yourapp.com/' }   // MUST end with / — see rule 4
  });
  const link = data.properties.action_link;
  // We send the email ourselves with our own branding
  await sendEmail({ to: email, subject: 'Sign in', html: ourTemplate(link) });
  res.json({ ok: true });
});
```

Same pattern for `type: 'recovery'` (password reset) and `type: 'signup'`. Each app sends its own branded emails. Supabase's project SMTP stays untouched.

### 4. `redirectTo` MUST end with `/` to match `/**` in the allow-list

The Supabase Redirect URLs allow-list patterns like `https://app.example.com/**` require the URL to have a path component. A bare origin without trailing slash does NOT match:

```
allow-list:  https://app.example.com/**
redirectTo:  https://app.example.com         ← does NOT match → falls back to Site URL
redirectTo:  https://app.example.com/        ← matches ✓
```

In server code, always normalize:

```js
function originFromReq(req) {
  const o = req.headers.origin;
  if (o) return o.replace(/\/+$/, '') + '/';   // strip then add one trailing slash
  return APP_URL.replace(/\/+$/, '') + '/';
}
```

**Symptom if you skip this:** Magic link verification works, but user is redirected to the WRONG app's URL (whichever has Site URL set in the shared project). Looks like "auth doesn't work" — it actually does, just lands at the wrong app.

### 5. Site URL goes to the most-active app, but never depend on it

If you can change it without breaking the older apps (often the case if older apps' Site URLs already point to dead domains), point Site URL at the newest app. But the right long-term pattern is: every app passes `redirectTo` explicitly. Don't depend on Site URL at all.

### 6. URL-confusion trap when editing in the Supabase dashboard

Supabase remembers the LAST project you opened and lands there. The dashboard top-bar shows the project NAME (which may be a legacy name from the original app), not the ref. Mosh edited the wrong project SQL editor twice in one session because of this.

**Always use direct links with the project ref:**

```
https://supabase.com/dashboard/project/<PROJECT_REF>/sql/new
https://supabase.com/dashboard/project/<PROJECT_REF>/auth/url-configuration
https://supabase.com/dashboard/project/<PROJECT_REF>/auth/templates
```

When you tell Mosh to go fix something in the dashboard, paste the full URL with the ref. Never say "go to your SMTP settings."

### 7. URL-hash auth error handling

Magic links can expire or be prefetched by Gmail's URL-scanning. When that happens, Supabase redirects back to the app with `#error=access_denied&error_code=otp_expired&...`. Without handling this, the user sees a normal login screen with no explanation — feels like the magic link "did nothing."

In your app's init, parse the hash for errors and show them:

```js
function checkUrlError() {
  const h = window.location.hash || '';
  if (!h.includes('error=')) return null;
  const params = new URLSearchParams(h.replace(/^#/, ''));
  history.replaceState(null, '', window.location.pathname);
  return {
    code: params.get('error_code') || params.get('error'),
    desc: decodeURIComponent(params.get('error_description') || '').replace(/\+/g, ' ')
  };
}
```

Then on the login screen show: "That magic link expired or was already used. Tap Send magic link again."

## Quick checklist before adding a new app to a shared Supabase project

- [ ] Picked a unique table prefix (e.g. `uom_`, `gonogo_`)
- [ ] Schema includes INSERT RLS policy on the profile-equivalent table
- [ ] Server has `SUPABASE_SERVICE_ROLE_KEY` env var (server-only, never client)
- [ ] App sends its own branded emails via Resend/Postmark/etc — NOT Supabase SMTP
- [ ] All `redirectTo` URLs include trailing slash
- [ ] Added app's URL pattern to Redirect URLs allow-list — did NOT touch Site URL
- [ ] URL hash error handling in client init
- [ ] Used direct project-ref URLs when asking Mosh to make dashboard changes
