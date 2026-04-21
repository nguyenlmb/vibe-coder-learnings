# Supabase + Google Auth Setup Guide for Vibe Coders

A practical walkthrough for setting up a production-ready Supabase project with Google OAuth, tables, RLS, and edge functions. Every gotcha here was discovered the hard way — see [lessons-learned.md](./lessons-learned.md) for the full context.

---

## Part 1: Supabase Project Setup

### 1. Create your project

1. Go to [supabase.com](https://supabase.com) and create a new project
2. Note your **Project URL** and **anon key** from **Project Settings → API**
   - The anon key is safe to include in client-side code — it's designed to be public and only grants access to what your RLS policies allow
   - Your **service role key** bypasses all security rules; treat it like a root password — never put it in frontend code or commit it to git

### 2. Install the Supabase CLI

The CLI lets you manage migrations, deploy edge functions, and push config changes from your terminal — without it you're clicking around the dashboard manually.

```bash
brew install supabase/tap/supabase
supabase login
supabase init        # in your project root
supabase link --project-ref <your-project-ref>
```

`supabase link` connects your local project to the remote one — without this, CLI commands like `db push` and `functions deploy` don't know which project to talk to.

### 3. Create your database tables

Write migrations in `supabase/migrations/` — one file per change, named with a timestamp prefix. The timestamp is how the CLI knows which migrations have already been applied and in what order; without it, you have no reliable way to track schema history.

**Schema best practices (from hard experience):**

1. Add `NOT NULL` to every column that should always have a value (especially `user_id`). Without it, a bug can silently insert a row with a missing `user_id`, which then bypasses your RLS policies — the row becomes invisible to the user but still takes up space and can cause confusing downstream failures.

2. Add `CHECK` constraints on columns with a fixed set of values:
   ```sql
   status TEXT NOT NULL CHECK (status IN ('pending', 'processing', 'completed'))
   ```
   Without this, a typo or bad write (e.g. `'complet'`) goes through silently — the row gets stranded with a status no part of your code knows how to handle.

3. Add indexes on columns you filter by frequently — especially `user_id` and `status`. Row-level security filters every query by user ID; without an index, Postgres scans every row in the table on every request. This is fine at 100 rows and invisible as a problem until you have 100,000.

4. **Always verify your schema after running migrations** — the CLI tracks migrations by filename, not by actually checking your schema. If a migration was manually marked as applied without running, `supabase db push` reports "up to date" while the table doesn't exist in production. After every migration, run a quick SQL check to confirm the table or column actually exists:
   ```sql
   SELECT column_name FROM information_schema.columns WHERE table_name = 'your_table';
   ```

### 4. Set up Row-Level Security (RLS)

Without RLS, any authenticated user can query any row in your table directly via Supabase's REST API — your app code not calling that endpoint is no protection at all. RLS moves the access check into the database itself.

Enable RLS on every table that holds user data:

```sql
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;

-- Users can only read/write their own rows
CREATE POLICY "users can access own data"
  ON your_table FOR ALL
  USING (auth.uid() = user_id);
```

**Watch out for `SECURITY DEFINER` functions** — SQL functions with this flag run with elevated privileges and bypass RLS entirely. Any authenticated user who can call such a function gets admin-level access to your data, regardless of your policies. Always add an explicit permission check at the top:

```sql
CREATE OR REPLACE FUNCTION admin_only_function()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM admins WHERE user_id = auth.uid()
  ) THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  -- ... rest of function
END;
$$;
```

---

## Part 2: Google Auth Setup

### Web App vs. Desktop App — choose the right client type

Google has two OAuth client types, and using the wrong one causes real pain. A Web App client requires a `client_secret` for every token operation — which you can't safely include in a distributed desktop app. A Desktop App client is specifically designed to work without one.

| | Web App | Desktop App |
|---|---|---|
| Use for | Next.js, React apps in a browser | Electron, native apps |
| `client_secret` required | Yes, for token exchange | No — omit it entirely |
| Redirect URIs | Your deployed domain | `127.0.0.1:PORT/callback` (see below) |
| Token refresh | Requires `client_secret` | Works without `client_secret` |

---

### Option A: Google Auth for a Web App

1. Go to [Google Cloud Console](https://console.cloud.google.com) → APIs & Services → Credentials
2. Create a **Web Application** OAuth 2.0 client
3. Add your redirect URIs (both local dev and production). Google only redirects to pre-registered URIs as a security measure — if the URL doesn't match exactly, the OAuth flow fails with a `redirect_uri_mismatch` error.
   - For local dev: `http://localhost:3000/auth/callback`
   - For production: `https://yourdomain.com/auth/callback`
4. In Supabase: go to **Authentication → Providers → Google**, enter your Client ID and Client Secret, and set the redirect URL to your Supabase auth callback URL (shown in the dashboard). This tells Supabase to act as your OAuth intermediary — it handles the code exchange, creates a session, and manages token refresh on your behalf so you don't have to wire any of that up yourself.
5. In your app, use Supabase's `signInWithOAuth`. This call opens the Google consent screen in the user's browser; Supabase handles all the redirect and callback plumbing so your code doesn't need to deal with URL parameters or token exchange:
   ```ts
   await supabase.auth.signInWithOAuth({
     provider: 'google',
     options: { redirectTo: 'https://yourdomain.com/auth/callback' },
   });
   ```
6. On the callback page, call `supabase.auth.getSession()`. Supabase detects the auth code in the URL automatically and completes the token exchange — you don't need to parse the URL or call Google's token endpoint yourself:
   ```ts
   const { data: { session } } = await supabase.auth.getSession();
   ```

---

### Option B: Google Auth for a Desktop App (Electron)

Desktop apps can't receive HTTP redirects the normal way because they're not running a web server. Instead, you spin up a tiny local server just to catch the one-time callback from Google, then shut it down.

#### 1. Create a Desktop App OAuth client

1. Go to Google Cloud Console → APIs & Services → Credentials
2. Create an **OAuth 2.0 Client ID** of type **Desktop App**
3. Note your **Client ID** — you do **not** need the Client Secret. Google explicitly doesn't require it for Desktop App clients because there's no secure way to distribute a secret in an installed app.

#### 2. Open a local callback server

When the user clicks "Sign in", your app should:

1. Generate a random CSRF token. Any process on the user's machine could send a request to your open local port — this token lets you verify the callback actually came from Google's redirect and not from something else:
   ```ts
   const csrfToken = crypto.randomUUID();
   ```

2. Start a local HTTP server on a fixed port (e.g. 54321) — **bind to `127.0.0.1`, not `localhost`**:
   - On macOS, `localhost` can silently resolve to an IPv6 address, causing the OAuth redirect to miss your server entirely with no visible error — login just hangs
   - `127.0.0.1` always resolves correctly

3. Embed the CSRF token in the callback URL you register as the redirect URI:
   ```
   http://127.0.0.1:54321/callback?state=<csrfToken>
   ```

4. Add a body size limit (64KB is generous) and a request timeout (30 seconds) to the server. Without these, a malicious local process can keep the socket open or flood it with data, leaving your app stuck waiting indefinitely.

5. Open the Google OAuth URL in the system browser. Google redirects back to your local server after the user consents:
   ```
   https://accounts.google.com/o/oauth2/v2/auth
     ?client_id=YOUR_CLIENT_ID
     &redirect_uri=http://127.0.0.1:54321/callback
     &response_type=code
     &scope=openid email profile
     &state=<csrfToken>
   ```

#### 3. Handle the callback

When Google redirects to your local server:

1. **Verify the `state` parameter matches your CSRF token** — if it doesn't match, reject the request immediately. This prevents other processes on the machine from injecting a fake session.

2. Exchange the `code` for tokens. Google sends a short-lived one-time `code` rather than tokens directly — you exchange it server-side (or locally in a desktop app) to prove you're the same client that initiated the flow. For a Desktop App client, **omit `client_secret`**:
   ```ts
   const response = await fetch('https://oauth2.googleapis.com/token', {
     method: 'POST',
     body: new URLSearchParams({
       code,
       client_id: CLIENT_ID,
       redirect_uri: 'http://127.0.0.1:54321/callback',
       grant_type: 'authorization_code',
       // No client_secret — Desktop App clients don't need it
     }),
   });
   const { access_token, refresh_token, expires_in } = await response.json();
   ```

3. Store `refresh_token` securely (e.g. in the OS keychain via `keytar` in Electron). The refresh token is long-lived and lets you get new access tokens silently without asking the user to log in again — if it's stolen, an attacker has persistent access until the user explicitly revokes it.

#### 4. Keep tokens fresh proactively

Don't wait for a 401 to know your token expired — that surfaces a visible error to the user at the worst possible moment. Google access tokens expire after 1 hour; refresh silently before that happens:

1. When you receive tokens, record the expiry time:
   ```ts
   const expiresAt = Date.now() + expires_in * 1000;
   ```

2. Set up a background timer to refresh ~2 minutes before expiry:
   ```ts
   const msUntilRefresh = expiresAt - Date.now() - 2 * 60 * 1000;
   setTimeout(() => refreshTokens(), msUntilRefresh);
   ```

3. To refresh a Desktop App token — again, **no `client_secret`**:
   ```ts
   const response = await fetch('https://oauth2.googleapis.com/token', {
     method: 'POST',
     body: new URLSearchParams({
       refresh_token: storedRefreshToken,
       client_id: CLIENT_ID,
       grant_type: 'refresh_token',
       // No client_secret
     }),
   });
   ```

---

## Part 3: Supabase Auth Integration

### 5. Connect Google tokens to Supabase sessions

After Google auth, your app has Google tokens but no Supabase session — RLS policies, database queries, and edge functions all depend on a Supabase session. `signInWithIdToken` bridges the two: it hands Google's `id_token` to Supabase, which verifies it and creates a proper session:

```ts
const { data, error } = await supabase.auth.signInWithIdToken({
  provider: 'google',
  token: id_token,
});
```

### 6. Always call `refreshSession()` before using a token

The Supabase JS client caches your session in memory and doesn't guarantee it's still valid when you call `getSession()`. If the cached token has expired, you'll pass a dead token to your edge function and get a silent 401. Before passing a token anywhere sensitive:

```ts
const { data: { session } } = await supabase.auth.refreshSession();
const token = session?.access_token;
```

Or skip the SDK wrapper entirely and use raw `fetch()` with the token directly — `supabase.functions.invoke()` uses `getSession()` under the hood and has the same staleness problem.

### 7. Handle the ES256 JWT format

Newer Supabase projects sign JWTs with ES256 (asymmetric keys). Edge functions with `verify_jwt = true` (the default) expect the older HS256 format and will reject every valid user token with a 401 — meaning all authenticated calls to your functions will fail out of the box. Set `verify_jwt = false` and verify the token yourself:

```toml
[functions.your-function]
verify_jwt = false
```

```ts
// Inside your edge function
const token = req.headers.get('Authorization')?.replace('Bearer ', '');
const { data: { user }, error } = await supabase.auth.getUser(token);
if (error || !user) return new Response('Unauthorized', { status: 401 });
```

This gives you the same security with an approach that actually works.

---

## Part 4: Edge Functions

### 8. Authenticate every edge function call

Edge functions are publicly reachable HTTP endpoints — anyone who knows the URL can call them. Every function must:

1. **Extract the token** from the `Authorization: Bearer` header
2. **Verify it's a valid session** via `supabase.auth.getUser(token)` — not `getSession()`, which reads from cache and doesn't hit the auth server
3. **Confirm the authenticated user owns the resource** they're asking about (e.g. the `user_id` in the request path matches `user.id`)

Without steps 1–2, anyone with your function URL can call it. Without step 3, any logged-in user can access another user's data.

### 9. Fail closed — never fail open

If your edge function checks a secret or env var and that value isn't set (e.g. right after deploying to a new environment), the function must **refuse the request**. A missing secret should be treated as a misconfiguration, not as "skip the check":

```ts
// BAD — when secret is missing, no verification happens and everyone gets in
if (secret) { verifySecret(secret); }

// GOOD — when secret is missing, return an error
if (!secret) return new Response('Service unavailable', { status: 503 });
verifySecret(secret);
```

### 10. Use constant-time comparison for secrets

When comparing a webhook secret or API key, don't use `===`. Regular string comparison stops at the first mismatch, which leaks timing information — an attacker making thousands of requests can measure response times to figure out the correct secret one character at a time. Use `timingSafeEqual` instead, which always takes the same time regardless of where the mismatch is:

```ts
import { timingSafeEqual } from 'https://deno.land/std/crypto/timing_safe_equal.ts';

const expected = new TextEncoder().encode(storedSecret);
const provided = new TextEncoder().encode(requestSecret);
if (expected.length !== provided.length || !timingSafeEqual(expected, provided)) {
  return new Response('Unauthorized', { status: 401 });
}
```

### 11. Restrict CORS to specific origins on admin functions

CORS determines which websites are allowed to make requests to your function from a browser. `Access-Control-Allow-Origin: *` means any website can call your admin function — including a malicious site the user happens to visit while logged into your admin portal. Restrict it to your exact admin origin:

```ts
const ALLOWED_ORIGIN = Deno.env.get('ADMIN_PORTAL_URL');
const origin = req.headers.get('Origin');

const corsHeaders = {
  'Access-Control-Allow-Origin': origin === ALLOWED_ORIGIN ? ALLOWED_ORIGIN : '',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
};
```

### 12. Never log any part of a user's token

Even `token.slice(0, 20)` leaks information. Logs are often stored long-term, shared across teams, and occasionally exposed in breaches — a partial token narrows down the search space for an attacker and confirms they're on the right track. Log only the error message:

```ts
// BAD
console.error('Auth failed, token prefix:', token.slice(0, 20));

// GOOD
console.error('Auth failed:', error.message);
```

---

## Part 5: Secrets & Config

### 13. Don't ship a `.env` file in your app bundle

In a packaged desktop app (e.g. Electron), the app bundle is just a zip file — anyone who installs it can extract its contents and read any files you included. Distinguish between what's safe to hard-code and what isn't:

- **Safe to hard-code:** Supabase URL, anon key, Google Client ID — these are visible in network requests anyway
- **Never in the bundle:** service role key, API secrets, anything that grants elevated access
- Only load `.env` in local development

### 14. Add secret scanning as a pre-commit hook

Accidentally committing a secret to git is permanent — it lives in history, forks, and CI caches even after deletion. Add [gitleaks](https://github.com/gitleaks/gitleaks) to catch it before it ever reaches the remote:

```bash
brew install gitleaks
# Add to .pre-commit-config.yaml or as a git hook in .husky/
```

Add an allowlist for values that are intentionally public (like a Supabase anon key) so developers don't start ignoring the warnings.

---

## Quick Checklist

**Supabase tables:**
- [ ] `NOT NULL` on all required columns
- [ ] `CHECK` constraints on status/enum columns
- [ ] Indexes on `user_id` and frequently-filtered columns
- [ ] RLS enabled with user-scoped policies
- [ ] Verified tables actually exist after migrations (not just "tracked as applied")

**Edge functions:**
- [ ] `verify_jwt = false` in `config.toml` + manual `getUser()` verification
- [ ] Ownership assertion (user ID from token matches resource in request)
- [ ] Fail-closed secret/env checks
- [ ] Constant-time secret comparison
- [ ] CORS restricted to specific origin
- [ ] No token logging

**Google Auth — Desktop App:**
- [ ] Desktop App client type (not Web App)
- [ ] Callback server bound to `127.0.0.1` (not `localhost`)
- [ ] CSRF token in callback URL, verified on receipt
- [ ] Body size limit + request timeout on callback server
- [ ] No `client_secret` in token exchange or refresh calls
- [ ] Proactive token refresh ~2 minutes before expiry

**Google Auth — Web App:**
- [ ] Web App client type with correct redirect URIs
- [ ] Use Supabase's `signInWithOAuth` + callback handler
- [ ] Call `refreshSession()` before passing tokens to edge functions
