# PackTrack Integration (first project on the hub)

## Two kinds of question, two tools
- **Live data** ("what indents are pending right now") → `query_packtrack_db`, hits the DB.
- **Static/how-it-works knowledge** ("what validations exist on PO upload") → `search_packtrack_knowledge`, answered from embedded notes, never the live DB.

## `query_packtrack_db` — defense in depth (`queryGuard.js` + `db.js`)
1. **Keyword rejection** before any DB round-trip — reject `INSERT`/`UPDATE`/`DELETE`/`DROP`/`ALTER`/`TRUNCATE`/`GRANT`/`REVOKE`/`CREATE` as standalone keywords (word-boundary regex, case-insensitive).
2. **Statement timeout**: 10s, enforced DB-side (`ALTER ROLE mcp_readonly SET statement_timeout = '10s'`) — holds regardless of client behavior.
3. **Row cap**: wrap as `SELECT * FROM (<query>) AS q LIMIT 501`; trim to 500 with a truncation note if hit.
4. **Logging**: every attempt → `{ event: 'packtrack_query', email, sql, rowCount, durationMs, timestamp }` — this *is* the audit trail.
5. **Project gate first**: if `'packtrack'` isn't in `extra.authInfo.projects`, reject before reaching the guard or DB.
6. `schema-context.md` (adapted `docs/db-schema.md` + gotchas below) injected into the tool description for grounding.

## The read-only Postgres role — lives in `packtrack-pro`, not this repo
**Applied 2026-08-13** — `mcp_readonly` role created and verified (`has_table_privilege('mcp_readonly','materials','SELECT')` → true, same check on `users` → false). Migration `packtrack-pro/db/024_mcp_readonly_role.sql`, commit `1033aad`:
```sql
CREATE ROLE mcp_readonly WITH LOGIN PASSWORD '<generate strong random password>';
GRANT CONNECT ON DATABASE <db> TO mcp_readonly;
GRANT USAGE ON SCHEMA public TO mcp_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO mcp_readonly;
REVOKE SELECT ON users, sessions FROM mcp_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO mcp_readonly;
ALTER ROLE mcp_readonly SET statement_timeout = '10s';
```
`users`/`sessions` excluded entirely (password hashes, session tokens). Neon may require the console/CLI for role creation rather than plain SQL — check the Neon console's "Roles" tab first.

## Two behavioral gotchas (not yet in packtrack-pro's own schema doc — must ride along in `schema-context.md`)
1. `FORCE_COMPLETED` is a shared status string across `indent_lines.status`, `stock_issues.status`, and `purchase_orders.status` — same string, different meaning per table, disambiguated only by context/`force_complete_reason`.
2. IST/UTC: server runs UTC; `new Date()` is UTC; IST = UTC+5:30. Correct pattern: `new Date(Date.now() + 5.5*60*60*1000)` before slicing a date string. (Same gotcha documented in packtrack-pro's own `CLAUDE.md`.)

## `search_packtrack_knowledge` — knowledge base
- Same Neon Postgres, `pgvector` extension (`CREATE EXTENSION IF NOT EXISTS vector;`) — no second database.
- `knowledge_chunks` table: `id, project, source_file, heading, content, embedding vector(1536), updated_at` — `store.js`'s `ensureSchema()` creates both on first ingest run.
- **`KNOWLEDGE_DATABASE_URL` uses the full `packtrack-pro` owner connection string, not `mcp_readonly`** — decided during implementation. `mcp_readonly` is intentionally SELECT-only (that's the whole point of the query-tool defense-in-depth), but ingestion needs `CREATE TABLE`/`INSERT` into `knowledge_chunks`. Reusing the strictly-read-only role for that would either require weakening it (defeating its purpose) or granting narrow extra privileges on just one table — for a single-owner setup, using the existing owner credentials for this one write path was simpler and is what's in the local `.env`. Revisit with a dedicated `mcp_knowledge` role if this ever needs tighter separation.
- Embedding provider: **OpenAI `text-embedding-3-small`** (1536 dims) — picked over Voyage AI as "the simpler fallback" per the original plan's own wording, avoiding a new vendor account for a single-owner tool. `src/knowledge/embed.js` isolates this choice to one file.
- Ingestion (`src/knowledge/ingest.js --project packtrack`): walk `notes/*.md`, header-aware chunk, embed, upsert. **Manual, not automatic** — re-run whenever `notes/` changes. Staleness risk tracked in [[07 - Open Risks]].
- Query time: embed query, cosine similarity (`<=>`) filtered `WHERE project = 'packtrack'`, top 6 chunks returned with `source_file`/`heading` metadata. Model should say plainly if nothing relevant comes back, not fall back to general knowledge.
- Seed `notes/` content: PO upload validation rules, GRN flow, role model, indent→PO→GRN→issuance stage semantics — written fresh, not just a copy of `schema-context.md`.

## Related Notes
- [[00 - Overview]]
- [[03 - Roles & Access]]
- [[07 - Open Risks]]
