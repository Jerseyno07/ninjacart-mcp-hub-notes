# Change Log

Newest entries at the top. Each entry: what changed, why, commit hash(es).

---

## 2026-08-13 — Google Cloud Console set up, first real end-to-end login test, 3 bugs found & fixed

- **Google Cloud Console**: OAuth consent screen + Web application OAuth client created (user did this manually — needs their own console access). `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` added to the local, gitignored `.env`. Redirect URI registered: `http://localhost:3000/oauth/callback` (Railway domain callback still to be added post-deploy).
- **First real login test**, port 3000 (had to temporarily stop the PackTrack Pro PM2 process, which owns that port locally — restarted it after; production on Railway was never affected). Ran the full flow live: DCR `/register` → `/authorize` → real Google sign-in as `aranyanandi@ninjacart.com` → domain + role check passed → broker code → `/token` exchange (PKCE verified, including a deliberate mismatch test that correctly failed) → authenticated `POST /mcp` → `initialize` → `tools/list` → `tools/call query_packtrack_db`.
- **3 bugs found and fixed this way** (commit `3992cce`):
  1. `embed.js` built its OpenAI client at import time — an unset `EMBEDDING_API_KEY` crashed the *entire* server on boot, not just the knowledge tool. Made lazy.
  2. `jwt.js` signed `aud` as the bare public origin, but `tokenVerifier.js`'s `checkResourceAllowed` check compares against the `/mcp` resource path specifically — every token failed resource validation as a result. Fixed `aud` to include `/mcp`.
  3. `tokenVerifier.js` threw plain `Error` on invalid/expired tokens. `requireBearerAuth` only recognizes the SDK's own `OAuthError` subclasses, so an unrecognized error type silently became an opaque 500 instead of the "self-explanatory" 401 the plan called for. Switched to `InvalidTokenError`.
- **Verified live against the real Postgres role**: `SELECT count(*) FROM materials` → 44 (correct), `DELETE FROM materials...` → rejected before any DB round-trip, `SELECT * FROM users` → `permission denied for table users` (proves the `REVOKE` in migration 024 holds independently of the app-side guard).

**Why:** These are exactly the kind of bugs that only surface under a real OAuth round-trip — the earlier smoke test (dummy env, no real Google flow) couldn't have caught the `aud` mismatch or the error-class issue, since both only trigger once a genuinely valid or genuinely invalid *real* token exists.

**Next:** Railway deployment (steps 20–23 of the build plan) — needs the user's Railway access to create the project and set env vars, then adding the Railway domain as a second Google OAuth redirect URI, then the verification checklist against the live deployment.

---

## 2026-08-13 — Core implementation: auth, MCP server, PackTrack tools

Implemented steps 3–18 of the build plan.

- **Auth**: `roles.js` (seeded with `aranyanandi@ninjacart.com` as `ADMIN`), `jwt.js`/`tokenStore.js`, `googleOAuth.js`, `routes.js` (`/authorize`, `/oauth/callback`, `/token`, `/register`), `tokenVerifier.js`.
- **MCP server**: `mcp/mcpServer.js` + `src/server.js` — Express bootstrap, `/health`, `requireBearerAuth`-gated `/mcp` via `StreamableHTTPServerTransport`.
- **Deviation from the plan, found during implementation**: a `McpServer` can only be connected to one transport at a time (SDK throws on a second `connect()`), so the planned "one shared singleton" doesn't hold for concurrent sessions. Changed to a `createMcpServer()` factory, one fresh instance per session, still wired through the single `registerAllProjects()` path. Full explanation in [[01 - Architecture]].
- **Knowledge pipeline**: `chunk.js`, `embed.js` (OpenAI `text-embedding-3-small`, chosen as the simpler default over Voyage AI), `store.js` (pgvector), `ingest.js` CLI.
- **PackTrack project**: `schema-context.md` (adapted from `packtrack-pro/docs/db-schema.md` + the two behavioral gotchas), four seed notes (`po-upload-validations.md`, `grn-flow.md`, `indent-issuance-flow.md`, `role-model.md`, drawn from the packtrack-pro SOP doc), `db.js`, `queryGuard.js`, `index.js` (both tools + project gate).
- **Smoke-tested locally** (port 3099, dummy env): `/health` → 200, unauthenticated `POST /mcp` → 401 with a correct `WWW-Authenticate` header, DCR `/register` → `/authorize` → Google redirect verified end-to-end.
- **`packtrack-pro/db/024_mcp_readonly_role.sql` applied to the live Neon database** — `mcp_readonly` role created, verified `SELECT` works on `materials` and is blocked on `users`. Password generated locally (not committed); read-only connection string saved into `ninjacart-mcp-hub/.env` (gitignored). `packtrack-pro` commit `1033aad`.
- **`KNOWLEDGE_DATABASE_URL` uses the packtrack-pro owner credentials, not `mcp_readonly`** — ingestion needs `CREATE TABLE`/`INSERT`, which the strictly-read-only role deliberately doesn't have. Documented in [[04 - PackTrack Integration]] as a revisit-later item, not an oversight.

Commits: `ninjacart-mcp-hub` `9d9c23e`, `packtrack-pro` `1033aad`.

**Why:** Continuing straight through the build plan's implementation sequence per the go-ahead to begin implementing; the DB migration specifically was confirmed with the user first since it touches the live production database.

**Next:** Google Cloud Console OAuth client setup (manual, needs the user's own console access) and Railway deployment — steps 19–23 of the build plan. Both need credentials/access this session doesn't have.

---

## 2026-08-13 — Project scaffolded

Set up the project's infrastructure ahead of implementation, mirroring PackTrack Pro's setup:
- **New standalone Obsidian vault** (`~/Documents/ninjacart-mcp-hub/`) — separate from the PackTrack Pro vault, not a subfolder. `obsidian-git` configured for 10-min auto-commit. Notes 00–07 written from the build plan (`~/.claude/plans/robust-sparking-music.md`).
- **New GitHub repo for notes**: [`Jerseyno07/ninjacart-mcp-hub-notes`](https://github.com/Jerseyno07/ninjacart-mcp-hub-notes) (private). Initial commit `— see repo`.
- **New code repo** (`~/ninjacart-mcp-hub/`): directory skeleton per the plan's layout (`src/{server.js, mcp/, auth/, knowledge/, projects/registry.js, projects/packtrack/}`), `package.json` (ESM, dependencies listed, not yet installed), `.env.example`, `railway.toml`, `README.md`, `CLAUDE.md` (IST/UTC gotcha + logging rule carried over from PackTrack Pro). Pushed to [`Jerseyno07/ninjacart-mcp-hub`](https://github.com/Jerseyno07/ninjacart-mcp-hub) (private), commit `43801ad`.

**Why:** Wanted the same project-tracking discipline PackTrack Pro has before writing any OAuth/MCP code, so each implementation step gets logged as it lands rather than after the fact.

**Next:** Implementation sequence steps 3–23 from the build plan — `npm install`, auth modules, MCP server, PackTrack tools, knowledge ingestion pipeline, then the Neon read-only role migration in `packtrack-pro`, Google Cloud Console setup, and Railway deploy.

---
