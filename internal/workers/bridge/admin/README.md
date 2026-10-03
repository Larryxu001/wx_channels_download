# Bridge Admin Pages

This directory is a standalone Cloudflare Pages project for the Bridge administration interface. The pages run separately from the Durable Objects Worker, and the browser accesses only the Pages domain.

`public` contains only the administration interface's own HTML, CSS, and JavaScript source. Before deployment, `build.sh` creates the Git-ignored `dist` directory and copies the required Timeless runtime from `frontend/public/timeless`, avoiding a second copy of third-party assets in the repository.

## Request Flow

```text
Browser
  ├── /index.html, /app.js, /style.css ── Pages static assets
  └── /admin/api/* ── Pages Worker ── BRIDGE Service Binding ── Bridge Worker
```

`worker.js` uses Pages advanced mode and HTTP Basic Auth to protect the entire project. The username is fixed as `admin`, and the password comes from the Pages secret `BRIDGE_ADMIN_TOKEN`. The API proxy preserves the Authorization header, and the Worker validates the same administrator token again.

Each device card has a “日志” (“Logs”) button that opens the right-hand drawer and calls `/admin/api/devices/:device_id/logs` to display the last seven days of device activity. Connection, heartbeat, call, response, and system events are stored as structured logs. The drawer supports category filters, automatic refresh, pagination into older entries, and expanded details. Sensitive fields are redacted before storage, and oversized details are truncated.

The right-hand “调用 Token” (“Call Tokens”) drawer also uses `/admin/api/access-tokens` to create, immediately expire, and remove external caller credentials. Leave the token blank to generate one automatically, or specify one manually. Its purpose or user is optional. The plaintext token appears only once in the creation response. The Durable Object stores only its SHA-256 digest, optional description, expiry time, and last-used time. The device credential `BRIDGE_TOKEN` is never displayed in the administration interface and must not be distributed to external callers.

## Deployment

Run the following command from the project root to deploy the Worker, create or update the Pages project, configure secrets and the Service Binding, and publish the administration interface:

```bash
go run . deploy bridge
```

Deployment reuses `cloudflare.accountId` and `cloudflare.apiToken` directly. The token requires Workers Scripts:Edit and Pages:Edit permissions; no Wrangler login is needed. `BRIDGE_ADMIN_TOKEN` comes from `bridge.deploy.adminToken` and is never written to `wrangler.jsonc`, JavaScript, or any static file.

The Pages project name is set by `bridge.deploy.pagesProjectName`; if blank, it defaults to `<bridge.deploy.workerName>-admin`. The deployment command automatically points the `BRIDGE` Service Binding to the deployed Worker.

## Local Verification

To start both the local Worker and Pages, run this recommended command from the project root:

```bash
./internal/workers/bridge/dev.sh
```

The following approach starts only the administration interface and connects it through a Service Binding to another Worker that is already running.

For static-page checks, use any static file server. To test the Pages Worker with a remote Worker, use Wrangler with a local secret:

```bash
cd internal/workers/bridge/admin
printf 'BRIDGE_ADMIN_TOKEN="your-admin-token"\n' > .dev.vars
./build.sh
npx wrangler@latest pages dev
```

`.dev.vars*` is covered by the repository's `.gitignore`; nevertheless, avoid copying real secrets into any other version-controlled file.
