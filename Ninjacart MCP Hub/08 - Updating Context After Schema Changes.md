# Updating Context After Schema Changes

Checklist for whenever PackTrack Pro gets new tables, columns, or workflow changes — nothing here is automatic, all three steps are manual.

## 1. New/changed tables → grant read access

Since [[07 - Open Risks]] #10 (`026_mcp_readonly_deny_future_tables.sql`), `mcp_readonly` is **deny-by-default** on new tables. A new table is invisible to `query_packtrack_db` until explicitly granted:

```sql
GRANT SELECT ON <new_table> TO mcp_readonly;
```

Run this directly against the live Neon DB (owner credentials) after creating the table. Without it, the SQL tool returns `permission denied for table <new_table>`.

## 2. Schema description → update `schema-context.md`, then redeploy

`src/projects/packtrack/schema-context.md` is hand-written and `readFileSync`'d **once at server startup** into `query_packtrack_db`'s tool description — it does not reflect the live DB automatically.

- Not required for the tool to *function* — it can run SQL against anything it's granted regardless.
- But without an update, the model won't know a new table/column exists unless it discovers it itself (e.g. querying `information_schema`).
- Edit the file, commit, push to `main` — Railway auto-deploys and the new process re-reads it on startup.

## 3. Workflow/conceptual changes → notes + re-ingest

`search_packtrack_knowledge` only reflects whatever was last embedded into `knowledge_chunks`. Nothing watches `src/projects/packtrack/notes/` for changes ([[07 - Open Risks]] #8).

1. Edit or add a `.md` file under `src/projects/packtrack/notes/`.
2. Run:
   ```
   node src/knowledge/ingest.js --project packtrack
   ```
3. This is safe to re-run — `replaceFileChunks` deletes and re-inserts chunks scoped to `(project, source_file)` inside one transaction, so no duplicates or stale versions pile up for an edited file.

**Caveat**: this is scoped per file. Renaming or deleting a notes file does *not* automatically clean up its old chunks in `knowledge_chunks` — there's no tooling for that today, would need a manual `DELETE` if a file is removed/renamed.

## Quick reference

| Change | Action | Needs redeploy? |
|---|---|---|
| New table | `GRANT SELECT ... TO mcp_readonly` | No |
| New/changed columns worth surfacing to the model | Edit `schema-context.md` | Yes |
| New/changed workflow, business rule, concept | Edit/add `notes/*.md` → `ingest.js` | No (ingest writes straight to Neon) |
| Removed/renamed notes file | Manual `DELETE FROM knowledge_chunks WHERE ...` | No |

## Related Notes
- [[04 - PackTrack Integration]]
- [[07 - Open Risks]]
