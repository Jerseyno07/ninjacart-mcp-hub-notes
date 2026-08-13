# Architecture

## Repository layout
```
ninjacart-mcp-hub/
├── package.json                  # "type": "module" (ESM)
├── .env.example
├── railway.toml
├── README.md
├── src/
│   ├── server.js                 # Express bootstrap: helmet, cors, auth routes,
│   │                              # MCP metadata routes, requireBearerAuth-gated /mcp, /health
│   ├── mcp/mcpServer.js          # The ONE shared McpServer instance every project registers onto
│   ├── auth/
│   │   ├── googleOAuth.js        # OAuth2Client wrapper: buildAuthUrl / exchangeCodeForTokens / verifyIdTokenAndDomain
│   │   ├── roles.js              # THE ONE FILE touched to grant/revoke a person's access
│   │   ├── tokenStore.js         # In-memory Map + TTL sweep — PKCE challenges, DCR clients
│   │   ├── jwt.js                # signAccessToken / verifyAccessToken (jsonwebtoken)
│   │   ├── tokenVerifier.js      # Passed to requireBearerAuth
│   │   └── routes.js             # /authorize, /oauth/callback, /token, /register
│   ├── knowledge/
│   │   ├── ingest.js             # CLI: chunk notes/, embed, upsert into knowledge_chunks
│   │   ├── embed.js              # embedText() wrapper
│   │   ├── store.js              # pgvector upsert/search
│   │   └── chunk.js              # header-aware markdown chunking
│   ├── projects/
│   │   ├── registry.js           # THE ONE FILE touched to plug in a new project's tools
│   │   └── packtrack/
│   │       ├── index.js          # register(mcpServer): defines both tools, enforces project gate
│   │       ├── db.js             # dedicated read-only Pool (PACKTRACK_READONLY_DATABASE_URL)
│   │       ├── queryGuard.js     # keyword rejection + row cap + logging
│   │       ├── schema-context.md # adapted from packtrack-pro/docs/db-schema.md + gotchas
│   │       └── notes/            # source markdown, chunked/embedded by ingest.js
│   └── util/logger.js
└── public/                       # optional login/rejection page CSS
```

## Extensibility model
Adding a **second project** later needs only: its own `index.js` exporting `register(mcpServer)`, its own DB/API client with its own credential env var, its own query-guard-equivalent, its own `schema-context.md`, its own `notes/` folder, and one new import line in `registry.js`. Nothing in `auth/`, `mcp/mcpServer.js`, `knowledge/`, or `server.js` changes to add a project — that's what keeps the core project-agnostic.

Granting a *person* access to a project is a separate, equally small change: extend their entry in `roles.js` — see [[03 - Roles & Access]].

## Deliberate deviations from PackTrack Pro's style
- **ESM, not CommonJS** — `@modelcontextprotocol/sdk` and its docs are ESM-first; avoids interop friction. (PackTrack Pro is CommonJS.)
- **Same Neon Postgres**, but via a dedicated read-only role (`mcp_readonly`) rather than the app's own credentials — see [[04 - PackTrack Integration]].

## Related Notes
- [[00 - Overview]]
- [[02 - Auth & OAuth Flow]]
- [[04 - PackTrack Integration]]
