# Building OAuth-Protected MCP Servers on Cloudflare Workers

A practical guide based on building and debugging two production MCP servers (Gong MCP and Copilot MCP) for Claude.ai. Covers architecture, OAuth flow, security, and hard-won lessons from real failures.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [OAuth Flow](#oauth-flow)
3. [Required Endpoints](#required-endpoints)
4. [Security Checklist](#security-checklist)
5. [Hard-Won Lessons](#hard-won-lessons)
6. [Cloudflare-Specific Gotchas](#cloudflare-specific-gotchas)
7. [Debugging Playbook](#debugging-playbook)

---

## Architecture Overview

```
User (Claude.ai)
    │
    ▼
Claude.ai frontend
    │  1. Probe MCP endpoint (no auth) → 401
    │  2. Fetch /.well-known/oauth-protected-resource
    │  3. Fetch /.well-known/oauth-authorization-server
    │  4. Redirect user to /oauth/authorize
    │
    ▼
Cloudflare Worker (MCP Server)
    │  5. Redirect to Google (or Gong) OAuth
    │
    ▼
Identity Provider (Google / Gong)
    │  6. User authenticates, redirects back to claude.ai callback
    │
    ▼
Claude.ai backend
    │  7. POST /oauth/token with code + code_verifier
    │
    ▼
Cloudflare Worker
    │  8. Exchange code → get email → verify user → create session → return access_token
    │
    ▼
Claude.ai (now has access_token)
    │  9. POST / with Authorization: Bearer <token>  →  MCP JSON-RPC
```

**Stack:**
- Cloudflare Workers (runtime)
- Cloudflare KV (session storage)
- Google OAuth (identity — email only)
- Supabase / external API (data)

---

## OAuth Flow

### Step 1 — Discovery

Claude.ai fetches two metadata endpoints before starting OAuth:

```
GET /.well-known/oauth-protected-resource
→ { "resource": "https://your-worker.workers.dev", "authorization_servers": ["https://your-worker.workers.dev"] }

GET /.well-known/oauth-authorization-server
→ { "issuer": "...", "authorization_endpoint": ".../oauth/authorize", "token_endpoint": ".../oauth/token", ... }
```

**Both are required.** Missing `oauth-protected-resource` causes an immediate "Authorization failed" error — Claude.ai never starts the OAuth flow.

### Step 2 — Authorization

Claude.ai redirects the user to your `/oauth/authorize` with:
- `state` — CSRF token (Claude.ai generates this)
- `code_challenge` — PKCE challenge (SHA-256 of `code_verifier`)
- `code_challenge_method=S256`
- `resource` — your worker URL (MCP 2025-11-25+)

Your handler stores the PKCE challenge and redirects to the identity provider.

### Step 3 — Token Exchange

After the user authenticates, Claude.ai calls your `/oauth/token` with:
- `code` — the auth code from the identity provider
- `code_verifier` — the PKCE verifier
- **NOT `state`** — Claude.ai MCP protocol 2025-11-25 does not send state here

Your handler:
1. Parses the code and verifier
2. Validates PKCE (compute SHA-256 of verifier, match against stored challenge)
3. Exchanges the code with the identity provider
4. Verifies the user exists in your system
5. Creates a session token, stores in KV
6. Returns `{ access_token, token_type: "bearer", expires_in }`

### Step 4 — MCP Requests

All subsequent MCP requests arrive as:
```
POST /
Authorization: Bearer <session_token>
Content-Type: application/json

{ "jsonrpc": "2.0", "method": "initialize", "id": 1 }
```

---

## Required Endpoints

### `GET /.well-known/oauth-protected-resource`
```javascript
return Response.json({
  resource: url.origin,
  authorization_servers: [url.origin]
});
```

### `GET /.well-known/oauth-authorization-server`
```javascript
return Response.json({
  issuer: url.origin,
  authorization_endpoint: url.origin + "/oauth/authorize",
  token_endpoint: url.origin + "/oauth/token",
  response_types_supported: ["code"],
  grant_types_supported: ["authorization_code"]
});
```

### `GET /oauth/authorize`
```javascript
// Store PKCE challenge keyed by challenge value (not state)
const codeChallenge = url.searchParams.get("code_challenge");
if (codeChallenge) {
  await env.KV.put(`pkce_challenge:${codeChallenge}`, "1", { expirationTtl: 600 });
}
// Redirect to identity provider
```

### `POST /oauth/token`
```javascript
const code = params.get("code");
const state = params.get("state");         // may be absent — do not require
const codeVerifier = params.get("code_verifier");

// Validate PKCE (primary security mechanism)
if (codeVerifier) {
  const hash = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(codeVerifier));
  const challenge = btoa(String.fromCharCode(...new Uint8Array(hash)))
    .replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");
  const valid = await env.KV.get(`pkce_challenge:${challenge}`);
  if (!valid) return Response.json({ error: "invalid_pkce" }, { status: 400 });
  await env.KV.delete(`pkce_challenge:${challenge}`);
}
```

### `POST /` (MCP endpoint)
```javascript
// Always validate session before checking method
const token = request.headers.get("Authorization")?.slice(7);
const session = await getSessionWithRetry(env.KV, token); // see KV retry below
if (!session) return Response.json({ error: "unauthorized" }, { status: 401 });
```

---

## Security Checklist

### ✅ Authentication
- [ ] Session validated before **every** MCP method — not just `tools/call`
- [ ] `GET /mcp` with a fake token must return 401, not 200
- [ ] Session lookup uses retry (KV eventual consistency — see below)

### ✅ OAuth
- [ ] `/.well-known/oauth-protected-resource` endpoint exists
- [ ] `state` is **optional** at `/oauth/token` (MCP 2025-11-25 doesn't send it)
- [ ] PKCE challenge stored by challenge value, not by state
- [ ] State consumed immediately after reading (prevent replay)
- [ ] Google scopes minimal: `"openid email"` only — no `profile`

### ✅ CORS
- [ ] Restricted to `https://claude.ai` only — not `*`
- [ ] `WWW-Authenticate` includes `resource_metadata` URL

### ✅ Error handling
- [ ] Internal errors return generic messages — no stack traces, no API details
- [ ] Google/Gong token exchange errors logged server-side, not returned to client

### ✅ Data safety
- [ ] Every user-supplied field sanitized before returning to Claude
- [ ] Prompt injection patterns filtered (e.g. "ignore previous instructions")
- [ ] All Supabase queries scoped to `user_id` — never return other users' data
- [ ] No `...spread` of raw API responses — enumerate fields explicitly

### ✅ Tooling
- [ ] `/debug` endpoint gated by `DEBUG_SECRET` env var
- [ ] Tool calls logged with user email + tool name + timestamp
- [ ] `TOOL_HEADER` injected into every tool response as defense-in-depth

---

## Hard-Won Lessons

### 1. `state` is not sent to `/oauth/token` in MCP 2025-11-25

Claude.ai's current MCP protocol does **not** forward `state` when calling your token endpoint. If you require it, every token exchange returns 400 and no user can ever connect. Make `state` optional and use PKCE as the primary security mechanism instead.

**Symptom:** Google login popup appears, user selects account, then "couldn't connect."  
**Log signature:** `token_exchange_start` fires but no subsequent `state_check` log.

### 2. `/.well-known/oauth-protected-resource` is required

Claude.ai MCP 2025-11-25 fetches this before anything else. If it returns 404, Claude.ai cannot discover your OAuth server and immediately shows "Authorization failed" — the Google popup never appears.

**Symptom:** "Authorization with MCP server failed" immediately on connect, no Google popup.  
**Log signature:** `GET /.well-known/oauth-protected-resource` → 404.

### 3. PKCE key must not depend on `state`

The natural key for the PKCE challenge is `pkce:${state}`. But since `state` isn't sent to the token endpoint, the lookup always misses. Key it by the challenge value itself instead:

```javascript
// At authorize time:
await env.KV.put(`pkce_challenge:${codeChallenge}`, "1", { expirationTtl: 600 });

// At token time — derive challenge from verifier:
const computed = sha256base64url(codeVerifier);
const valid = await env.KV.get(`pkce_challenge:${computed}`);
```

### 4. KV eventual consistency breaks fresh sessions

Cloudflare KV is eventually consistent. A session token written at `/oauth/token` may not be visible on the edge node handling the next MCP request (milliseconds later). Add retry:

```javascript
async function getSessionWithRetry(kv, token) {
  for (const delay of [0, 200, 400]) {
    if (delay) await new Promise(r => setTimeout(r, delay));
    const raw = await kv.get(`session:${token}`);
    if (raw) return raw;
  }
  return null;
}
```

**Symptom:** Intermittent 401 immediately after connecting, works on retry.  
**Log signature:** `session_not_found` with `cpuTimeMs: 0, wallTimeMs: 0`.

### 5. The initial unauthenticated `POST /` is expected

Claude.ai's first request to your MCP endpoint has **no Authorization header**. This is normal — it's a probe to trigger the OAuth discovery flow. Your server should return 401 with `WWW-Authenticate`. Do not treat this as an error.

---

## Cloudflare-Specific Gotchas

**KV writes are eventually consistent globally** — reads on a different edge node may miss a write for up to a few seconds. Use retry for any KV read that immediately follows a write in a different request.

**`cpuTimeMs: 0` in logs** means the response was returned before any async operations. If you see this on a 401, the request has no Authorization header (synchronous path) — not a KV miss (which takes wall time).

**Secrets via Cloudflare dashboard** — never hardcode secrets in worker code. Use the Cloudflare dashboard → Worker Settings → Variables & Secrets, or `wrangler secret put`.

**KV TTL is in seconds** — `expirationTtl: 600` = 10 minutes, `expirationTtl: 86400 * 30` = 30 days.

**Workers are stateless** — each request is a fresh invocation. Do not rely on in-memory state between requests.

---

## Debugging Playbook

### "Authorization with MCP server failed" immediately
1. Check if `GET /.well-known/oauth-protected-resource` returns 200
2. Check if `GET /.well-known/oauth-authorization-server` returns 200
3. If either is 404 → add the missing endpoint

### Google popup appears but then "couldn't connect"
1. Check logs for `POST /oauth/token`
2. Look for `token_exchange_start` — if present, check `has_code` and `has_state`
3. If `has_state: false` and you require state → make state optional
4. If no `token_exchange_start` at all → token endpoint isn't being hit (check URL routing)

### Connected but 401 on tool calls
1. Check if session exists in KV via `/debug?key=SECRET&email=user@domain.com`
2. If session exists → KV consistency issue, add retry
3. If session missing → token exchange may have silently failed

### Debug endpoint
Add a gated `/debug` endpoint to check live state without touching the database:

```javascript
if (url.pathname === "/debug") {
  const key = url.searchParams.get("key");
  if (!env.DEBUG_SECRET || key !== env.DEBUG_SECRET) {
    return new Response("Forbidden", { status: 403 });
  }
  // Check KV sessions, external API connectivity, etc.
}
```

Access: `https://your-worker.workers.dev/debug?key=YOUR_SECRET&email=user@company.com`

### Useful log patterns to add
```javascript
// On auth failure — tells you if header is missing vs. session not found
console.warn(JSON.stringify({ event: "auth_failed", reason: "no_auth_header" | "session_not_found" }));

// On token exchange — tells you which parameters arrived
console.warn(JSON.stringify({ event: "token_exchange_start", reason: `code=${!!code},state=${!!state},verifier=${!!verifier}` }));

// On tool call — audit trail
console.log(JSON.stringify({ event: "tool_call", user: session.email, tool: name }));
```

---

## MCP JSON-RPC Methods to Handle

```javascript
"initialize"              // Return server capabilities
"tools/list"              // Return tool definitions
"tools/call"              // Execute a tool
"prompts/list"            // Return prompt templates
"prompts/get"             // Return a specific prompt
"notifications/initialized" // Acknowledge (return empty result)
```

All other methods should return `{ code: -32601, message: "Method not found" }`.

---

*Built and battle-tested April 2026. MCP protocol version: 2025-11-25.*
