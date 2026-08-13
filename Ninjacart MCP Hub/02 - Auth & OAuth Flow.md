# Auth & OAuth Flow

## Why hand-rolled, not the SDK shortcut
The MCP TypeScript SDK's `ProxyOAuthServerProvider` has an open, unresolved bug specifically against Google (`modelcontextprotocol/typescript-sdk#1112`, filed Nov 2025 — `invalid_request: Unregistered redirect_uri`). This server instead runs the completely standard "Sign in with Google" flow itself, checks the account's domain, and only then issues its own short-lived token for MCP tool calls. The custom part is small and isolated to that translation step, not the Google login itself.

## Flow
1. **`GET /authorize`** — MCP client sends `client_id`, `redirect_uri`, `code_challenge`, `state`, etc. Validate `client_id` against registered clients (via `/register`). Render plain-language login page, link to Google's `/o/oauth2/v2/auth` with a broker-generated `state` mapped (in `tokenStore`, ~10min TTL) to the original client's PKCE challenge/redirect/state.

2. **`GET /oauth/callback`** — Google redirects back with `code` + our `state`. Exchange code, verify ID token. **Domain check**:
   ```js
   const hdOk = payload.hd === process.env.ALLOWED_GOOGLE_DOMAIN;
   const emailOk = payload.email_verified &&
     payload.email.toLowerCase().endsWith('@' + process.env.ALLOWED_GOOGLE_DOMAIN);
   if (!hdOk && !emailOk) { /* reject */ }
   ```
   Then **role check**: `getRole(payload.email)` from `roles.js` — undefined means real `@ninjacart.com` account but not on the allowlist, rejected the same as a domain failure (see [[03 - Roles & Access]]). On success: one-time broker code → `tokenStore` (email + role + client's PKCE/redirect/state, ~2min TTL) → redirect to client's `redirect_uri`.

3. **`POST /token`** — standard `grant_type=authorization_code` exchange. PKCE check (S256 only). Sign our JWT (`sub=email`, `role`, `projects`, 1h expiry), return `{ access_token, token_type: "Bearer", expires_in: 3600 }`.

4. **`POST /register`** — Dynamic Client Registration for MCP clients; public PKCE client, no secret. Only registers *client apps*, not users — real access control is the Google-domain + role check above.

5. **SDK metadata routes** — `mcpAuthMetadataRouter` exposes `/.well-known/oauth-protected-resource`.

## Storage tradeoff
`tokenStore.js` is an in-memory `Map` + TTL sweep — fine for short TTLs and a single Railway instance. A mid-flow restart drops in-flight logins (user retries); breaks if ever scaled to >1 replica. Keep single-replica, or swap backing store later. Tracked in [[07 - Open Risks]].

## Self-explanatory error copy (exact requirement)
- 401 JSON: `{ "error": "unauthorized", "error_description": "Access to this MCP server requires signing in with an @ninjacart.com Google account. Start the login flow at: https://<server>/authorize" }`
- Domain rejection page: "Access denied — you signed in as `<email>`, but this server only allows @ninjacart.com Google Workspace accounts." No raw OAuth error codes or stack traces shown.

## Related Notes
- [[00 - Overview]]
- [[03 - Roles & Access]]
