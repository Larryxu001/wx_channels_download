# WeChat Official Accounts Cloudflare Worker

This directory contains the Worker source, D1 migrations, and complete deployment orchestration used by `deploy mp`.

- `index.js`: WeChat Official Accounts RSS/API Worker entry point;
- `migrations/`: D1 migrations embedded in the Go binary and executed by `mp.Deploy`;
- `deploy.go`: Find or create D1, validate connectivity, run migrations, upload the Worker, and resolve its workers.dev URL;
- `wrangler.toml`, `package.json`: Optional Wrangler configuration for local development.

The CLI entry point only reads `cloudflare.*` and `mp.remoteServer.hostname` settings and calls `mp.Deploy`:

```bash
go run . deploy mp
```

The API token requires Workers Scripts:Edit and D1:Edit permissions. When `cloudflare.d1Name` is configured, deployment looks up the database by name and creates it if necessary; otherwise, it uses `cloudflare.d1Id` directly.
