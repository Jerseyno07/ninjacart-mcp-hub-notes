# Ninjacart MCP Hub

A personal/team MCP server that lets you ask plain-English questions and get answers from internal project data via an MCP client (Claude Desktop, claude.ai connectors), instead of running one-off scripted queries by hand. Built to grow into a hub for multiple internal projects behind one shared login, starting with a packaging-material inventory tracking system.

## What it does, at a high level
- Exposes a read-only data-query tool and a knowledge-search tool over an internal project's data.
- Access is gated behind company-domain login plus a per-person allowlist — not open to the public.
- Designed so additional internal projects can be added later without changing the shared login/access layer.

## Stack
| Layer | Tech |
|---|---|
| Server | Node.js + Express |
| MCP | `@modelcontextprotocol/sdk` |
| DB | PostgreSQL, read-only role |
| Hosting | Railway |

## A note on this vault
This project's detailed engineering notes (architecture, auth flow internals, deployment specifics, access lists, and open risk tracking) are kept in a private location — not published here. This page is intentionally a high-level summary only.
