# Open Risks

Carried over verbatim from the build plan so they stay tracked, not buried in a planning doc.

1. **In-memory `tokenStore`** resets on process restart (brief in-flight-login impact) and does not work if ever scaled to >1 Railway replica. Keep single-replica, or swap the backing store later.
2. **`@modelcontextprotocol/sdk` moves fast** — re-verify exact export paths and the `extra.authInfo` shape against the installed version before trusting the plan's code sketches literally.
3. **Neon role-creation mechanics** unverified — check the Neon console before assuming plain `CREATE ROLE ... PASSWORD` SQL works on the current plan.
4. **No refresh-token flow** — access tokens expire after 1h, full re-auth required. Deliberate simplicity tradeoff for personal/team scale.
5. **Google Workspace "Internal" consent screen** requires org-admin rights on the Cloud project — confirm availability; if absent, the app-level domain check alone is still sound, just one layer thinner.
6. **`roles.js` is hand-edited, not an admin UI** — fine at current scale, but every grant/revoke needs a code change + redeploy. Revisit (e.g. move to a DB table) if the team grows past a handful of people or grants become frequent.
7. **No row-level scoping within a project** — a role gates *which projects'* tools someone can call, not *which rows* of data they see. The free-form SQL tool can return any non-`users`/`sessions` row to anyone with `packtrack` in their `projects` list. Acceptable while everyone granted access should see all operational data; needs real Postgres RLS before onboarding roles below `ADMIN`.
8. **Knowledge base staleness** — `notes/` and `knowledge_chunks` only reflect reality as of the last manual `ingest.js` run, no auto re-index. Consider a pre-deploy hook later.
9. **`pgvector` availability on Neon** — verify the extension is enabled on the current plan before committing to it; fall back to `sqlite-vec` alongside the app if unavailable.

## Related Notes
- [[00 - Overview]]
- [[03 - Roles & Access]]
