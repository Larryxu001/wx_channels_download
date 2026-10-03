# WeChat Channels Query Worker

This directory contains the Cloudflare Worker source, query page, and deployment orchestration used by `deploy sph`.

- `worker.js`: WeChat Channels query Worker entry point;
- `index.html`: Query page served at the Worker's root path;
- `deploy.go`: Read the shared icon, upload the Worker, configure bindings, and resolve its workers.dev URL.

The CLI entry point only reads `cloudflare.sph*` settings and calls `sph.Deploy`:

```bash
go run . deploy sph
```
