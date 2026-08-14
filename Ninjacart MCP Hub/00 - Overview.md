# Ninjacart MCP Hub

**MCP connection link**: `https://ninjacart-mcp-hub-production.up.railway.app/mcp` — add this as a remote MCP server in Claude Desktop/claude.ai (Google sign-in required, see [[03 - Roles & Access]] for who's allowlisted).

A personal/team MCP server that lets you ask plain-English questions and get answers from Ninjacart project data — the way ad-hoc terminal sessions already do it — but as a proper remote MCP server other MCP clients (Claude Desktop, claude.ai connectors) can connect to. Built to grow into a hub for **multiple** projects behind one shared Google login, starting with PackTrack Pro.

## Why
Turn one-off scripted DB queries into a reusable, access-controlled tool other people/clients can call — without hand-building a bespoke tool per business question.

## What it does
- Exposes a generic **read-only SQL tool** (`query_packtrack_db`) — free-form SQL over PackTrack Pro's Neon Postgres, not a curated set of business-specific tools.
- Exposes a **knowledge search tool** (`search_packtrack_knowledge`) — answers conceptual/how-it-works questions ("what validations exist on PO upload?") from embedded project notes, not the live DB.
- Gates every connection behind Google OAuth restricted to `@ninjacart.com`, **plus** a per-email role allowlist (domain membership alone isn't enough).
- Designed so a second project can be added later by dropping in its own `index.js`/`db.js`/`notes/` — no changes to the auth/MCP core.

## Stack
| Layer | Tech |
|---|---|
| Server | Node.js + Express (ESM) |
| MCP | `@modelcontextprotocol/sdk` |
| Auth | Hand-rolled Google OAuth broker + self-issued JWT (not the SDK's `ProxyOAuthServerProvider` — open bug against Google, see [[02 - Auth & OAuth Flow]]) |
| DB | Neon Postgres (same DB as PackTrack Pro), raw `pg`, read-only role |
| Knowledge base | pgvector, embedding provider TBD (Voyage AI or OpenAI) |
| Hosting | Railway (separate project from PackTrack Pro) |

## Repos
- Code: `github.com/Jerseyno07/ninjacart-mcp-hub` (private) — local: `~/ninjacart-mcp-hub/`
- Notes (this vault): `github.com/Jerseyno07/ninjacart-mcp-hub-notes` (private) — local: `~/Documents/ninjacart-mcp-hub/`

## Origin
Full build plan: `~/.claude/plans/robust-sparking-music.md` (23-step implementation sequence, verification checklist, open risks). This vault tracks progress against that plan as it's built.

## Related Notes
- [[01 - Architecture]]
- [[02 - Auth & OAuth Flow]]
- [[03 - Roles & Access]]
- [[04 - PackTrack Integration]]
- [[05 - Change Log]]
- [[06 - Deployment]]
- [[07 - Open Risks]]
