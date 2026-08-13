# Change Log

Newest entries at the top. Each entry: what changed, why, commit hash(es).

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
