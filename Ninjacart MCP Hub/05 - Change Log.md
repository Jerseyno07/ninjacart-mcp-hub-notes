# Change Log

Newest entries at the top. Each entry: what changed, why, commit hash(es).

---

## 2026-08-14 — Embeddings switched to Voyage AI, knowledge search live in production

- **Discussed OpenAI vs Voyage AI** with the user: Voyage AI has a 200M-token one-time free grant (vs OpenAI's pay-from-token-one, though both are trivially cheap at this project's scale) and is generally rated better for retrieval — user chose Voyage.
- **Swapped `embed.js`** from OpenAI's `text-embedding-3-small` (1536 dims) to Voyage's `voyage-4` (1024 dims, `EMBEDDING_DIMENSIONS` updated to match). Also now passes Voyage's `inputType` (`'document'` at ingest time, `'query'` at search time) — a retrieval-quality knob OpenAI's API doesn't expose, wired through `ingest.js` and `packtrack/index.js`. Removed the now-unused `openai` npm dependency.
- **Real bug found while running actual ingestion** (not caught by reading docs): Voyage's free tier caps requests at 3/minute until a payment method is on file — a 4-file ingestion run hit a 429 partway through. Added retry-with-backoff in `embed.js` (up to 4 retries, ~21s apart) rather than requiring a manual re-run; confirmed the retry logic recovers automatically.
- **Full ingestion run against the shared production Neon DB**: 17 chunks across all 4 notes files (`grn-flow.md`, `indent-issuance-flow.md`, `po-upload-validations.md`, `role-model.md`). Verified with a direct semantic-search sanity check (`"what validations exist on PO upload"` → correctly top-matched `po-upload-validations.md`).
- **Pushed `EMBEDDING_API_KEY` to Railway, deployed the code swap** (commit `a08e4e6`) — the first redeploy attempt only picked up the new env var against the *old* OpenAI code (forgot to push the code changes first), caught by re-testing rather than assuming it worked. Second deploy had the real fix.
- **Full live verification of `search_packtrack_knowledge` in production**: real Google login → token → `tools/call` → correct, well-ranked results returned (top hit: "Force Complete — one concept, three screens" for a force-complete/short-delivery question).

**Why:** Closing out the one remaining gap from the Railway deployment — `search_packtrack_knowledge` was deployed but non-functional without an embedding key. Chose to actually run the real ingestion and a real production query rather than just deploying and assuming it works, which is what caught both the rate-limit issue and the forgot-to-push mistake.

**Status: both `query_packtrack_db` and `search_packtrack_knowledge` are fully live and verified in production.** No remaining blockers from the original build plan — only the items already tracked in [[07 - Open Risks]].

---

## 2026-08-13 — Deployed to Railway, live end-to-end verified

- **Railway CLI installed and used for everything** (`npm install -g @railway/cli`) — project `ninjacart-mcp-hub` created under the `jerseyno07` workspace, service linked directly to the `Jerseyno07/ninjacart-mcp-hub` GitHub repo on `main` (auto-deploys on push, same pattern as PackTrack Pro), domain generated: `ninjacart-mcp-hub-production.up.railway.app`.
- **All env vars set via `railway variable set`** from the local `.env`, with `MCP_PUBLIC_URL`/`GOOGLE_OAUTH_REDIRECT_URI` correctly pointed at the real Railway domain (not localhost). `EMBEDDING_API_KEY` intentionally left unset for now — `query_packtrack_db` works without it; `search_packtrack_knowledge` will error until it's added.
- **One deploy-time snag**: the domain was first created with `--port 3000` (a guess, before the container had actually started), but the app listens on whatever `PORT` Railway injects at runtime (came out to 8080) — caused a 502 until the domain's target port was corrected with `railway domain update --port 8080`. Domain→port binding is set once and should stay stable across redeploys.
- **Google Console redirect URI**: user added `https://ninjacart-mcp-hub-production.up.railway.app/oauth/callback` as a second Authorized redirect URI (alongside the localhost one). First login attempt hit `redirect_uri_mismatch` — not a config error, just Google's normal propagation delay (roughly 1–2 minutes here); retried successfully once it cleared.
- **Full live verification, exactly as done locally**: DCR `/register` → `/authorize` → real Google sign-in → domain+role check → token exchange → authenticated `POST /mcp` → `initialize` → `tools/call query_packtrack_db` (`SELECT count(*) FROM materials` → 44), all confirmed working against the actual production deployment, not just localhost.

**Why:** Finishing the last steps of the build plan (20–23) — Railway deployment plus the post-deploy Google Console update — and re-verifying live rather than assuming the local smoke test generalizes, since the domain-port and redirect-URI issues above are exactly the kind of thing that only shows up once real infrastructure is involved.

**Status: the Ninjacart MCP Hub is live and fully functional in production.** Remaining open items are all in [[07 - Open Risks]] (no refresh-token flow, in-memory tokenStore, no row-level scoping, etc.) — none block real use, all are known tradeoffs.

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
