# Deployment

**Live since 2026-08-13** — `https://ninjacart-mcp-hub-production.up.railway.app`, fully verified (real Google login → token → authenticated tool call, see [[05 - Change Log]]).

**MCP connection link (what clients actually connect to)**: `https://ninjacart-mcp-hub-production.up.railway.app/mcp`

New, **separate** Railway project (not a service inside PackTrack Pro's), deployed from `ninjacart-mcp-hub`. Railway CLI (`@railway/cli`, `npm install -g`) was used for the whole setup — project creation, service linked directly to the GitHub repo on `main` (auto-deploys on push, same as PackTrack Pro), env vars, and domain, all via `railway` commands rather than the dashboard.

**Gotcha hit during setup**: the service domain must target the actual port the app binds to — Railway injects its own `PORT` value at container runtime (came out to 8080 here), not necessarily the port guessed when the domain was first created. If `/health` 502s right after a fresh domain, check `railway domain update --port <actual-port>`.

## `railway.toml`
```toml
[build]
builder = "NIXPACKS"
buildCommand = "npm install"

[deploy]
startCommand = "node src/server.js"
restartPolicyType = "ON_FAILURE"
```

## Env vars (`.env.example`)
```
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_OAUTH_REDIRECT_URI=https://<your-railway-domain>/oauth/callback
ALLOWED_GOOGLE_DOMAIN=ninjacart.com
MCP_JWT_SECRET=                 # openssl rand -hex 32, do not reuse from another project
MCP_PUBLIC_URL=https://<your-railway-domain>
PACKTRACK_READONLY_DATABASE_URL=postgresql://mcp_readonly:password@host/dbname?sslmode=require
KNOWLEDGE_DATABASE_URL=postgresql://mcp_knowledge:password@host/dbname?sslmode=require
EMBEDDING_API_KEY=
PORT=3000
```

## Google Cloud Console setup (manual)
**Done 2026-08-13** — project + consent screen + Web application OAuth client created by the user, plus the Railway domain's `/oauth/callback` added as a second Authorized redirect URI alongside localhost. `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` are set both in the local gitignored `.env` and as Railway service variables.

**Gotcha**: adding a redirect URI in the console isn't instant — a first login attempt right after saving hit `redirect_uri_mismatch` even though the URI was correctly saved; it started working within a couple of minutes. If this happens, just wait and retry before assuming the config is wrong.

1. console.cloud.google.com → select/create a project.
2. **OAuth consent screen**: User Type = **Internal** if org admin rights exist on the Cloud project (extra layer beyond the app's own `hd`/domain check). Otherwise External + rely on the app-level check.
3. **Credentials → Create OAuth client ID** → Web application, name "Ninjacart MCP Hub". Authorized redirect URIs: `https://<railway-domain>/oauth/callback` + `http://localhost:3000/oauth/callback` for local testing.
4. Copy Client ID/Secret into Railway env vars.
5. Scopes: `openid email profile` — no extra console config needed.

## Deploy sequence (as actually run)
1. ~~Run `packtrack-pro/db/024_mcp_readonly_role.sql` against Neon~~ — done earlier, see [[04 - PackTrack Integration]].
2. ~~Set all env vars in Railway, deploy from GitHub~~ — done via Railway CLI.
3. ~~Run `node src/knowledge/ingest.js --project packtrack`~~ — done 2026-08-14 (17 chunks, 4 files), see [[05 - Change Log]].
4. ~~Update redirect URI/public URL once the real domain exists~~ — done.

## Still open
- Add the MCP server as a connector in Claude Desktop or claude.ai to confirm the full client-side experience (DCR → authorize → connected), not just raw HTTP calls.

## Related Notes
- [[00 - Overview]]
- [[04 - PackTrack Integration]]
