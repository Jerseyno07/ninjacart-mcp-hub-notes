# Change Log

## October 30, 2026
- Added a knowledge-base entry (`src/projects/packtrack/notes/google-sso-login.md`) documenting PackTrack Pro's invite-email redesign (table-based HTML template + plain-text fallback in `server/lib/mailer.js`, commit `04d61c8` on the packtrack-pro side) — commit `dc7315b`.
- Found `KNOWLEDGE_DATABASE_URL` in the local `.env` was stale (dated 2026-08-28, predating the 2026-10-22 Neon password rotation from the Railway security incident) — ingestion (`node src/knowledge/ingest.js --project packtrack`) failed with `password authentication failed for user 'neondb_owner'` until the credential was refreshed from the current Neon connection string. Re-ran ingestion successfully afterward (all 13 knowledge files re-embedded via Voyage).
