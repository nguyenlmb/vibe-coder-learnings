# Building an MCP Server on Cloudflare Workers

A step-by-step guide to building a production MCP server that connects to Claude.ai, based on real implementations (Gong MCP and Copilot MCP).

---

## What is MCP?

MCP (Model Context Protocol) lets Claude.ai call tools on an external server — like searching your database, fetching meetings, or querying an API. Claude talks to your server over JSON-RPC, and your server decides what tools to expose and how to handle them.

```
Claude.ai  ──────────────────►  Your MCP Worker  ──►  Your Data (Supabase, Gong, etc.)
           JSON-RPC over HTTPS
```

---

## Stack

- **Cloudflare Workers** — serverless runtime, deploys globally in seconds
- **Cloudflare KV** — key-value store for sessions
- **Google OAuth** — identify users by email
- **Your data source** — Supabase, REST API, anything

---

## Step 1 — Project Structure

A single JS file is all you need. No frameworks, no dependencies.

```
my-mcp/
└── worker.js      ← everything lives here
```

Deploy via the Cloudflare dashboard (paste and click Deploy) or `wrangler deploy worker.js`.

**Environment variables to set in Cloudflare dashboard → Worker Settings → Variables:**
```
GOOGLE_CLIENT_ID          (plain)
GOOGLE_CLIENT_SECRET      (secret)
DEBUG_SECRET              (secret)  ← for /debug endpoint
# plus whatever your data source needs
```

**KV namespace to create and bind:**
```
KV binding name: SESSIONS
```

---

## Step 2 — Worker Skeleton

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // CORS preflight
    if (request.method === "OPTIONS") {
      return new Response(null, { headers: corsHeaders() });
    }

    // Route handlers go here...

    return new Response("Not found", { status: 404 });
  }
};

function corsHeaders() {
  return {
    "Access-Control-Allow-Origin": "https://claude.ai",
    "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
    "Access-Control-Allow-Headers": "Authorization, Content-Type, Mcp-Session-Id",
  };
}
```

---

## Step 3 — OAuth Discovery Endpoints

Claude.ai fetches these two endpoints before starting the OAuth flow. **Both are required** — missing either one causes an immediate "Authorization failed" error.

```javascript
// Required by MCP protocol 2025-11-25
if (url.pathname === "/.well-known/oauth-protected-resource") {
  return Response.json({
    resource: url.origin,
    authorization_servers: [url.origin],
  });
}

// Authorization server metadata
if (url.pathname === "/.well-known/oauth-authorization-server") {
  return Response.json({
    issuer: url.origin,
    authorization_endpoint: url.origin + "/oauth/authorize",
    token_endpoint: url.origin + "/oauth/token",
    response_types_supported: ["code"],
    grant_types_supported: ["authorization_code"],
  });
}
```

---

## Step 4 — OAuth Flow

### 4a. Authorize endpoint

Claude.ai sends the user here to log in. You redirect to Google (or any identity provider).

```javascript
if (url.pathname === "/oauth/authorize") {
  const state = url.searchParams.get("state") ?? crypto.randomUUID();

  // Store PKCE challenge keyed by its own value
  // (Claude.ai doesn't send state to /oauth/token, so we can't use state as the key)
  const codeChallenge = url.searchParams.get("code_challenge");
  if (codeChallenge) {
    await env.SESSIONS.put(`pkce_challenge:${codeChallenge}`, "1", { expirationTtl: 600 });
  }

  const params = new URLSearchParams({
    response_type: "code",
    client_id: env.GOOGLE_CLIENT_ID,
    redirect_uri: "https://claude.ai/api/mcp/auth_callback",
    scope: "openid email",
    state,
    access_type: "online",
  });

  return Response.redirect("https://accounts.google.com/o/oauth2/v2/auth?" + params, 302);
}
```

### 4b. Token endpoint

Claude.ai calls this after the user logs in. Exchange the code for an email, verify the user, create a session.

```javascript
if (url.pathname === "/oauth/token" && request.method === "POST") {
  const params = new URLSearchParams(await request.text());
  const code = params.get("code");
  const codeVerifier = params.get("code_verifier");

  if (!code) return Response.json({ error: "missing_code" }, { status: 400 });

  // Validate PKCE — compute SHA-256 of verifier, check it matches a stored challenge
  if (codeVerifier) {
    const hash = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(codeVerifier));
    const challenge = btoa(String.fromCharCode(...new Uint8Array(hash)))
      .replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");
    const valid = await env.SESSIONS.get(`pkce_challenge:${challenge}`);
    if (!valid) return Response.json({ error: "invalid_pkce" }, { status: 400 });
    await env.SESSIONS.delete(`pkce_challenge:${challenge}`);
  }

  // Exchange code with Google
  const tokenRes = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      code,
      redirect_uri: "https://claude.ai/api/mcp/auth_callback",
      client_id: env.GOOGLE_CLIENT_ID,
      client_secret: env.GOOGLE_CLIENT_SECRET,
    }),
  });
  const tokens = await tokenRes.json();
  if (!tokenRes.ok) {
    console.error("Google token exchange failed:", JSON.stringify(tokens));
    return Response.json({ error: "authentication_failed" }, { status: 500 });
  }

  // Get user's email
  const userRes = await fetch("https://www.googleapis.com/oauth2/v3/userinfo", {
    headers: { Authorization: "Bearer " + tokens.access_token },
  });
  const { email } = await userRes.json();
  if (!email) return Response.json({ error: "no_email" }, { status: 401 });

  // Verify user is allowed (check your DB, allowlist, etc.)
  const user = await isAllowedUser(email, env);
  if (!user) return Response.json({ error: "not_authorized" }, { status: 403 });

  // Create session
  const sessionToken = crypto.randomUUID();
  await env.SESSIONS.put(
    `session:${sessionToken}`,
    JSON.stringify({ email, userId: user.id }),
    { expirationTtl: 60 * 60 * 24 * 30 }  // 30 days
  );

  return Response.json({
    access_token: sessionToken,
    token_type: "bearer",
    expires_in: 60 * 60 * 24 * 30,
  }, { headers: corsHeaders() });
}
```

> **Key gotcha:** Claude.ai MCP protocol 2025-11-25 does **not** send `state` to the token endpoint. Do not require it — use PKCE instead.

---

## Step 5 — Define Your Tools

Tools are what Claude can call. Define them as JSON schema.

```javascript
const TOOLS = [
  {
    name: "list_meetings",
    description: "List the user's recent meetings. Call this first to discover available meetings.",
    inputSchema: {
      type: "object",
      properties: {
        limit:     { type: "number", description: "Max results (default 20)" },
        date_from: { type: "string", description: "Filter from this date. Format: YYYY-MM-DD" },
        date_to:   { type: "string", description: "Filter to this date. Format: YYYY-MM-DD" },
      },
      required: [],
    },
  },
  {
    name: "get_meeting_summary",
    description: "Get the summary and action items for a specific meeting.",
    inputSchema: {
      type: "object",
      properties: {
        meeting_id: { type: "string", description: "The meeting UUID from list_meetings" },
      },
      required: ["meeting_id"],
    },
  },
];
```

**Tips for good tool descriptions:**
- Tell Claude **when** to use it and **when not to**
- Describe what the output looks like
- If tools have a natural order (list first, then get), say so explicitly

---

## Step 6 — MCP Endpoint (JSON-RPC Handler)

This is the main endpoint Claude.ai talks to after authentication.

```javascript
if (url.pathname === "/mcp" || url.pathname === "/") {
  // 1. Check auth header
  const authHeader = request.headers.get("Authorization");
  if (!authHeader?.startsWith("Bearer ")) {
    return new Response("Unauthorized", {
      status: 401,
      headers: {
        ...corsHeaders(),
        "WWW-Authenticate": `Bearer realm="${url.origin}", resource_metadata="${url.origin}/.well-known/oauth-protected-resource"`,
      },
    });
  }

  // 2. Validate session (with retry for KV consistency)
  const token = authHeader.slice(7);
  const sessionRaw = await getSessionWithRetry(env.SESSIONS, token);
  if (!sessionRaw) {
    return Response.json(
      { jsonrpc: "2.0", id: null, error: { code: 401, message: "Session expired. Please reconnect." } },
      { status: 401, headers: corsHeaders() }
    );
  }

  // Non-POST requests just confirm the endpoint is alive
  if (request.method !== "POST") {
    return new Response("MCP ready", { status: 200, headers: corsHeaders() });
  }

  const session = JSON.parse(sessionRaw);
  const { method, params, id } = await request.json();

  // 3. Route JSON-RPC methods
  if (method === "initialize") {
    return Response.json({
      jsonrpc: "2.0", id,
      result: {
        protocolVersion: "2024-11-05",
        capabilities: { tools: {}, prompts: {} },
        serverInfo: { name: "my-mcp", version: "1.0.0" },
      },
    }, { headers: corsHeaders() });
  }

  if (method === "tools/list") {
    return Response.json({
      jsonrpc: "2.0", id,
      result: { tools: TOOLS },
    }, { headers: corsHeaders() });
  }

  if (method === "tools/call") {
    const { name, arguments: args } = params;
    console.log(JSON.stringify({ event: "tool_call", user: session.email, tool: name, ts: new Date().toISOString() }));
    try {
      const result = await handleToolCall(name, args ?? {}, session, env);
      return Response.json({
        jsonrpc: "2.0", id,
        result: { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] },
      }, { headers: corsHeaders() });
    } catch (e) {
      return Response.json({
        jsonrpc: "2.0", id,
        result: { content: [{ type: "text", text: "Error: " + e.message }], isError: true },
      }, { headers: corsHeaders() });
    }
  }

  if (method === "notifications/initialized") {
    return Response.json({ jsonrpc: "2.0", id, result: {} }, { headers: corsHeaders() });
  }

  return Response.json({
    jsonrpc: "2.0", id,
    error: { code: -32601, message: "Method not found: " + method },
  }, { headers: corsHeaders() });
}
```

---

## Step 7 — Tool Handlers

Implement the actual logic for each tool.

```javascript
async function handleToolCall(name, args, session, env) {
  if (name === "list_meetings") {
    const data = await fetchFromYourDB(session.userId, args);
    return { meetings: data };
  }

  if (name === "get_meeting_summary") {
    if (!args.meeting_id) throw new Error("meeting_id is required");
    const data = await fetchMeeting(session.userId, args.meeting_id, env);
    if (!data) throw new Error("Meeting not found");
    return data;
  }

  throw new Error("Unknown tool: " + name);
}
```

Always scope queries to the authenticated user's ID — never return data from other users.

---

## Step 8 — Helper Utilities

### KV session retry

Cloudflare KV is eventually consistent. A session written at `/oauth/token` may not be visible on the next request milliseconds later. Retry with backoff:

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

### Debug endpoint

Essential for production debugging. Gate it with a secret.

```javascript
if (url.pathname === "/debug") {
  if (!env.DEBUG_SECRET || url.searchParams.get("key") !== env.DEBUG_SECRET) {
    return new Response("Forbidden", { status: 403 });
  }

  const email = url.searchParams.get("email");
  let sessionInfo = "pass ?email=user@domain.com to look up a session";

  if (email) {
    const keys = await env.SESSIONS.list({ prefix: "session:" });
    for (const key of keys.keys) {
      const raw = await env.SESSIONS.get(key.name);
      if (!raw) continue;
      const s = JSON.parse(raw);
      if (s.email?.toLowerCase() === email.toLowerCase()) {
        sessionInfo = {
          found: true,
          userId: s.userId,
          expires: key.expiration ? new Date(key.expiration * 1000).toISOString() : "no TTL",
        };
        break;
      }
    }
  }

  return Response.json({ server: "my-mcp", session: sessionInfo });
}
```

Access: `https://your-worker.workers.dev/debug?key=YOUR_SECRET&email=user@company.com`

---

## Step 9 — MCP Prompts (Optional but Useful)

Prompts are pre-written instructions that guide Claude through multi-step workflows. They show up in Claude.ai's UI as slash commands.

```javascript
const PROMPTS = [
  {
    name: "meeting_summary",
    description: "Get the summary and action items for a specific meeting.",
    arguments: [
      { name: "meeting_id", description: "Meeting UUID from list_meetings", required: true },
    ],
  },
];

// Add to your MCP handler:
if (method === "prompts/list") {
  return Response.json({ jsonrpc: "2.0", id, result: { prompts: PROMPTS } }, { headers: corsHeaders() });
}

if (method === "prompts/get") {
  const { name, arguments: args } = params;
  return Response.json({
    jsonrpc: "2.0", id,
    result: {
      description: "Get meeting summary",
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Use get_meeting_summary with meeting_id="${args?.meeting_id}". Present results in sections: Summary, Action Items.`
        }
      }]
    }
  }, { headers: corsHeaders() });
}
```

---

## Common Pitfalls

| Problem | Symptom | Fix |
|---|---|---|
| Missing `oauth-protected-resource` | "Authorization failed" immediately, no Google popup | Add the `/.well-known/oauth-protected-resource` endpoint |
| Requiring `state` at token exchange | Google popup shows, then "couldn't connect" | Make `state` optional — MCP 2025-11-25 doesn't send it |
| PKCE key uses `state` as lookup key | PKCE validation always fails when state is absent | Key PKCE challenge by the challenge value itself |
| No KV retry on session lookup | Intermittent 401 immediately after connecting | Add retry with 200ms/400ms backoff |
| Validating session only on `tools/call` | `tools/list` returns data for any bearer string | Validate session at the top, before any method routing |
| Wildcard CORS (`*`) | Any site can read auth responses | Restrict to `https://claude.ai` only |
| Returning raw API errors | Internal details leak to Claude | Log server-side, return generic messages |

---

## Full Request Flow Reference

```
1. Claude.ai → GET /.well-known/oauth-protected-resource       → 200
2. Claude.ai → GET /.well-known/oauth-authorization-server     → 200
3. Claude.ai → GET /oauth/authorize?state=&code_challenge=     → 302 to Google
4. User logs in with Google
5. Google    → redirect to https://claude.ai/api/mcp/auth_callback?code=&state=
6. Claude.ai → POST /oauth/token  { code, code_verifier }      → { access_token }
7. Claude.ai → POST /  { "method": "initialize" }              → capabilities
8. Claude.ai → POST /  { "method": "tools/list" }              → tool definitions
9. Claude.ai → POST /  { "method": "tools/call", ... }         → tool result
```

---

*Built April 2026 — MCP protocol version 2025-11-25, Cloudflare Workers.*
