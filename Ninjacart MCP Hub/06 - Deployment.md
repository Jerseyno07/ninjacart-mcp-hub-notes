# Deployment

New, **separate** Railway project (not a service inside PackTrack Pro's), deployed from `ninjacart-mcp-hub`.

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
**Done 2026-08-13** — project + consent screen + Web application OAuth client created by the user (needs their own console access, not something done from this session). `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` are in the local, gitignored `.env`. Still to do: add the Railway domain's `/oauth/callback` as a second authorized redirect URI once deployed (see step below).

1. console.cloud.google.com → select/create a project.
2. **OAuth consent screen**: User Type = **Internal** if org admin rights exist on the Cloud project (extra layer beyond the app's own `hd`/domain check). Otherwise External + rely on the app-level check.
3. **Credentials → Create OAuth client ID** → Web application, name "Ninjacart MCP Hub". Authorized redirect URIs: `https://<railway-domain>/oauth/callback` + `http://localhost:3000/oauth/callback` for local testing.
4. Copy Client ID/Secret into Railway env vars.
5. Scopes: `openid email profile` — no extra console config needed.

## Deploy sequence
1. Run `packtrack-pro/db/024_mcp_readonly_role.sql` against Neon (via console if plain SQL role creation isn't available), enable `pgvector` if `knowledge_chunks` shares the DB.
2. Set all env vars in Railway, deploy from GitHub.
3. Run `node src/knowledge/ingest.js --project packtrack` once notes exist and the DB is reachable.
4. If redirect URI/public URL weren't known before first deploy, update both Google Cloud Console and Railway env vars once the real domain exists, redeploy.

## Related Notes
- [[00 - Overview]]
- [[04 - PackTrack Integration]]
