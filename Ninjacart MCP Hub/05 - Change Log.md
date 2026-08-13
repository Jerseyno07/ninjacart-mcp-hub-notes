# Change Log

Newest entries at the top. Each entry: what changed, why, commit hash(es).

---

## 2026-08-13 — Project scaffolded

Set up the project's infrastructure ahead of implementation, mirroring PackTrack Pro's setup:
- **New standalone Obsidian vault** (`~/Documents/ninjacart-mcp-hub/`) — separate from the PackTrack Pro vault, not a subfolder. `obsidian-git` configured for 10-min auto-commit. Notes 00–07 written from the build plan (`~/.claude/plans/robust-sparking-music.md`).
- **New GitHub repo for notes**: [`Jerseyno07/ninjacart-mcp-hub-notes`](https://github.com/Jerseyno07/ninjacart-mcp-hub-notes) (private). Initial commit `— see repo`.
- **New code repo** (`~/ninjacart-mcp-hub/`): directory skeleton per the plan's layout (`src/{server.js, mcp/, auth/, knowledge/, projects/registry.js, projects/packtrack/}`), `package.json` (ESM, dependencies listed, not yet installed), `.env.example`, `railway.toml`, `README.md`, `CLAUDE.md` (IST/UTC gotcha + logging rule carried over from PackTrack Pro). Pushed to [`Jerseyno07/ninjacart-mcp-hub`](https://github.com/Jerseyno07/ninjacart-mcp-hub) (private), commit `43801ad`.

**Why:** Wanted the same project-tracking discipline PackTrack Pro has before writing any OAuth/MCP code, so each implementation step gets logged as it lands rather than after the fact.

**Next:** Implementation sequence steps 3–23 from the build plan — `npm install`, auth modules, MCP server, PackTrack tools, knowledge ingestion pipeline, then the Neon read-only role migration in `packtrack-pro`, Google Cloud Console setup, and Railway deploy.

---
