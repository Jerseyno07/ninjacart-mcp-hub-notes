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

3. **`POST /token`**, `grant_type=authorization_code` — PKCE check (S256 only). Sign our JWT access token (`sub=email`, `role`, `projects`, 1h expiry) **and** issue a refresh token (see below). Returns `{ access_token, refresh_token, token_type: "Bearer", expires_in: 3600 }`.

4. **`POST /token`, `grant_type=refresh_token`** (added 2026-08-14) — client sends its refresh token; we hash it, look it up in Postgres (`src/auth/db.js`), reject if missing/expired/revoked. Role/projects are **re-derived live** via `getRole(email)`, not trusted from the old token — a `roles.js` revocation takes effect on the account's next refresh. Issues a new access token + a **rotated** refresh token (old one immediately revoked). Same response shape as above. This is what lets an MCP client stay connected indefinitely without the user re-clicking "Sign in with Google" every hour.

5. **`POST /register`** — Dynamic Client Registration for MCP clients; public PKCE client, no secret. Registered clients are persisted in Postgres (`oauth_clients` table), not in-memory — survives redeploys. Only registers *client apps*, not users — real access control is the Google-domain + role check above.

6. **SDK metadata routes** — `mcpAuthMetadataRouter` exposes `/.well-known/oauth-protected-resource`. `oauthMetadata.grant_types_supported` advertises both `authorization_code` and `refresh_token`.

## Storage
`tokenStore.js`'s in-memory `Map` + TTL sweep is now only used for **minutes-lived, low-consequence** state — pending Google logins (~10min) and one-time authorization codes (~2min). Fine to lose on a restart; user just retries.

**Longer-lived state moved to Postgres on 2026-08-14** (`src/auth/db.js`, same Neon DB as `knowledge_chunks`): registered MCP clients (`oauth_clients`) and refresh tokens (`refresh_tokens`, opaque tokens stored as sha256 hashes, 90-day sliding TTL, rotated on every use). This is what actually fixed the "disconnects on every redeploy" problem — verified end-to-end including surviving a real Railway redeploy, see [[05 - Change Log]]. Still single-Railway-instance-only for the remaining in-memory pieces — tracked in [[07 - Open Risks]].

## Self-explanatory error copy (exact requirement)
- 401 JSON: `{ "error": "unauthorized", "error_description": "Access to this MCP server requires signing in with an @ninjacart.com Google account. Start the login flow at: https://<server>/authorize" }`
- Domain rejection page: "Access denied — you signed in as `<email>`, but this server only allows @ninjacart.com Google Workspace accounts." No raw OAuth error codes or stack traces shown.

## Related Notes
- [[00 - Overview]]
- [[03 - Roles & Access]]
