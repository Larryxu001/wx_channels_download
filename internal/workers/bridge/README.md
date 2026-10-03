# Personal CGI Bridge

A Bridge is a forwarding service managed by one person that connects external callers to multiple operating-system devices. It forwards uniform `method + args` calls to an appropriate online device and stores only call arguments, results, and download metadata. It does not proxy or store video files.

## Concepts

- One Worker deployment is one Bridge; it is no longer subdivided into multiple `bridge.id` values within a Worker.
- A device is an operating-system instance: a macOS host is one device; Linux in Docker and Windows/Linux in virtual machines are separate devices.
- Multiple processes on the same operating system still belong to one device and use the same stable `deviceId`; a new connection replaces that device's previous connection.
- Each device registers the `methods` it supports. External callers use a single CGI-style Bridge interface, express calls as `method + args`, and let the Bridge select the device that executes them.
- Discovery, authorization, search, and routing across multiple Bridges belong to a higher-level Bridge marketplace or collection, outside this Worker.

```text
External caller ─HTTPS─┐
                    │
Admin browser ─HTTPS──┼── Personal Bridge Worker ── Durable Object + SQLite
                    │             │
macOS device ────WSS──┤             ├── method: wxchannels.fetch
Linux container ─WSS──┘             └── method: download.create
```

Tasks use at-least-once delivery. The Bridge creates a 120-second lease when assigning a task, and the executing device renews it periodically. After a connection drops, tasks whose leases expire are requeued. If the publisher misses the WebSocket completion notification, it can still retrieve the persisted result through the task query API.

## Deploying a Bridge

Configure the Cloudflare deployment settings in `config.yaml`:

```yaml
cloudflare:
  accountId: "<ACCOUNT_ID>"
  apiToken: "<API_TOKEN>" # Workers Scripts:Edit and Pages:Edit

bridge:
  deploy:
    workerName: "dm-bridge"
    pagesProjectName: "" # Defaults to dm-bridge-admin when blank
    token: "<strong-random-device-only-secret>"
    adminToken: "<admin-password-different-from-device-token>"
```

Run:

```bash
go run . deploy bridge
```

The command deploys a single Durable Object Bridge Worker and a separate Cloudflare Pages administration project. Repeated deployments update the code while preserving the Bridge's device registrations and task data.

- `BRIDGE_TOKEN` is only for device connections and calls published by devices themselves. Do not distribute it to external callers.
- `BRIDGE_ADMIN_TOKEN` is only for the administration interface, administration API, and call-token management.
- External calls use separate call tokens created dynamically by the administrator in the Bridge.
- `/health` requires no authentication.
- `/v1/connect` requires the device's `BRIDGE_TOKEN` and device-identity headers.
- Other `/v1/*` endpoints accept dynamic call tokens; device programs can also use their device secret.
- `/admin/api/*` requires administrator authentication.

## Registering Operating-System Devices

Each operating system uses a configuration for a single Bridge:

```yaml
bridge:
  enabled: true
  url: "https://dm-bridge.<account>.workers.dev"
  deviceId: "mayfair-macbook"
  deviceName: "Mayfair MacBook"
  token: "<BRIDGE_TOKEN>"
  httpTimeoutSeconds: 30
  methods: "auto"
```

`deviceId` must be unique and stable within the Bridge. If left blank, the program uses the system hostname; set it explicitly in production. `deviceName` is the name displayed in the administration interface and also defaults to the hostname. The program automatically reports the operating-system type and the methods it has registered.

`methods` is a general method allowlist rather than separate boolean settings for each capability. `auto` exposes all methods registered by the current program; `none` lets the device publish calls without executing remote calls. You can also provide a comma-separated list such as `wxchannels.fetch,wxchannels.contact.feed.list,download.create`.

The WeChat Channels adapter currently registers these Bridge methods:

| method | args | Corresponding scraper method |
| --- | --- | --- |
| `wxchannels.contact.search` | `keyword`, `next_marker` | `SearchChannelsContact` |
| `wxchannels.contact.feed.list` | `username`, `next_marker` | `FetchChannelsFeedListOfContact` |
| `wxchannels.live.replay.list` | `username`, `next_marker` | `FetchChannelsLiveReplayList` |
| `wxchannels.feed.profile` | `oid`, `nid`, `url`, `eid` | `FetchChannelsFeedProfile` |
| `wxchannels.feed.comment.list` | `oid`, `nid`, `comment_id`, `next_marker` | `FetchChannelsFeedCommentList` |
| `wxchannels.feed.share_url` | `oid` | `FetchChannelsFeedShareUrl` |

These methods depend on the WebSocket connection from the WeChat Channels page on the device. Calls fail or time out if the device is connected to the Bridge but the Channels page is not connected.

The WeChat Official Accounts adapter currently registers these Bridge methods:

| method | args | Corresponding scraper method |
| --- | --- | --- |
| `wxmp.biz.msg.list` | `username`, `offset` | `FetchBizMsgList` |

This method depends on the WebSocket connection from the WeChat Official Accounts page on the device. `username` is required; `offset` may be omitted or set to the offset returned by the previous page.

Linux running in Docker uses a separate identity:

```yaml
bridge:
  enabled: true
  url: "https://dm-bridge.<account>.workers.dev"
  deviceId: "downloader-linux"
  deviceName: "Downloader Linux"
  token: "<BRIDGE_TOKEN>"
  httpTimeoutSeconds: 30
  methods: "download.create"
```

Local status endpoint:

```http
GET /api/bridge/status
```

Returns the status of the current operating-system device's single connection to its personal Bridge. The `?bridge=` selection parameter is no longer accepted.

## Administration Interface

The administration interface source is in `internal/workers/bridge/admin`:

- `public/index.html`, `style.css`, and `app.js` are the static source files;
- `build.sh` generates the Git-ignored `dist` directory and copies the shared runtime from `frontend/public/timeless`;
- `worker.js` handles HTTP Basic Auth and Service Binding forwarding for `/admin/api/*`.

Browser login:

- Username: `admin`
- Password: `bridge.deploy.adminToken`

The administration interface refreshes every five seconds and displays operating-system devices. Each device card has a “日志” (“Logs”) button that opens a right-hand drawer showing the last seven days of events, including connections, disconnections, connection heartbeats, call dispatch, device receipt, task heartbeats, successful responses, retries after failure, lease expiry, and administrator resets. Logs support category filters, automatic refresh, pagination into older entries, and expandable call arguments or response data. Sensitive fields such as Token, Cookie, Authorization, Password, and Secret are automatically redacted before storage, and oversized entry details are truncated.

Call tokens are also added and managed through the right-hand drawer. Administrators can create separate tokens, set initial credits, add credits, choose validity periods of 1/7/30/90 days or no expiry, expire tokens immediately, and remove tokens. Leave the token blank for the Bridge to generate it automatically, or specify it manually. The purpose or user is optional. The plaintext token is displayed only once after successful creation; the Bridge stores only its SHA-256 digest.

Each call token is a separate publisher. It can list and query only calls it created, and cannot read tasks published by other tokens or devices. Once a token expires or is removed, new requests immediately return `401`; calls already assigned to devices continue executing.

Administration API:

```bash
curl -H 'Authorization: Bearer <BRIDGE_ADMIN_TOKEN>' \
  'https://dm-bridge.<account>.workers.dev/admin/api/overview'
```

The overview response contains:

```json
{
  "generated_at": 0,
  "devices": [],
  "methods": [],
  "task_counts": [],
  "access_tokens": []
}
```

The administration API also supports direct call-token management:

```http
GET /admin/api/access-tokens
POST /admin/api/access-tokens
DELETE /admin/api/access-tokens/:id
POST /admin/api/access-tokens/:id/expire
POST /admin/api/access-tokens/:id/credits
GET /admin/api/credit-transactions?access_token_id=&limit=
GET /admin/api/devices/:device_id/logs?category=&before_id=&limit=
Authorization: Bearer <BRIDGE_ADMIN_TOKEN>
```

Device-log `category` can be `connection`, `heartbeat`, `call`, `response`, or `system`. `limit` ranges from 1–500; use `next_before_id` from the response to retrieve older logs.

Creation request:

```json
{
  "name": "合作方 A",
  "token": "custom-token-at-least-16-characters",
  "expires_in_seconds": 604800,
  "credits": 1000
}
```

The Bridge generates `token` automatically when it is blank or omitted. Custom values must be 16–256 characters long and contain only letters, numbers, and `._~+/=-`. `name` may be blank; `expires_in_seconds` set to `null` means no expiry; `credits` is a non-negative integer defaulting to `0`. The creation response is the only time `token` is returned in plaintext.

`amount` in a credit-adjustment request is the change in credits. The administration API permits negative adjustments to correct the ledger, but the resulting balance cannot fall below `0`. The administration interface only supports adding positive amounts:

```json
{
  "amount": 500,
  "reason": "购买 500 积分"
}
```

Every change is recorded in a permanent credit ledger with the token, associated task, adjustment, resulting balance, method, reason, and timestamp. Removing a token uses soft revocation, preserving historical records.

When a device registers `wxchannels.fetch`, its card offers a targeted method-call test. After a successful fetch, the result can be submitted as `args` to any online device that has registered `download.create`. Both operations use the same `/admin/api/call` endpoint.

## Local Development

```bash
./internal/workers/bridge/dev.sh
```

The Worker listens on `http://127.0.0.1:8787` by default, and Pages on `http://127.0.0.1:8788`. The administration username is `admin`, and the default local password is `local-bridge-admin-token`.

You can override the ports and tokens:

```bash
BRIDGE_WORKER_PORT=8797 \
BRIDGE_PAGES_PORT=8798 \
BRIDGE_TOKEN=my-local-token \
BRIDGE_ADMIN_TOKEN=my-local-admin-token \
WRANGLER_VERSION=latest \
./internal/workers/bridge/dev.sh
```

## Calling Methods Between Devices

The local API provides a unified endpoint:

```http
POST /api/bridge/call
Content-Type: application/json

{
  "method": "wxchannels.fetch",
  "target_device_id": "mayfair-macbook",
  "idempotency_key": "wx-feed-123",
  "args": {
    "url": "https://channels.weixin.qq.com/web/pages/feed?..."
  }
}
```

`target_device_id` is optional. If omitted, the Bridge selects an online, idle device that has registered the requested `method`. Adding a method only requires registering its handler on the device; the Bridge's task-type definitions do not need to change.

Existing application endpoints are convenient adapters for the unified call interface. For example, device A can delegate WeChat Channels parsing to device B:

Device A can delegate WeChat Channels parsing to device B:

```http
POST /api/bridge/tasks/wxchannels
Content-Type: application/json

{
  "url": "https://channels.weixin.qq.com/web/pages/feed?...",
  "target_device_id": "mayfair-macbook",
  "idempotency_key": "wx-feed-123",
  "download": {
    "download_dir": "/path/on/publisher",
    "auto_start": true,
    "config": {}
  }
}
```

When `target_device_id` is omitted, the Bridge automatically selects an online, idle device that has registered `wxchannels.fetch`.

Create a download task on a specific device:

```http
POST /api/bridge/tasks/download
Content-Type: application/json

{
  "target_device_id": "downloader-linux",
  "idempotency_key": "download-file-123",
  "url_request": {
    "url": "https://example.com/video.mp4",
    "download_dir": "/downloads",
    "filename": "video.mp4",
    "auto_start": true,
    "config": {}
  }
}
```

## External CGI Interface

For the complete user-facing integration flow, task-status descriptions, and code examples in multiple languages, see [`docs/feature/bridge.md`](../../../docs/feature/bridge.md).

External systems can treat the whole Bridge as a CGI node. Query online devices and methods:

```http
GET /v1
Authorization: Bearer <CALL_TOKEN>
```

Query the current token's credits:

```http
GET /v1/credits
Authorization: Bearer <CALL_TOKEN>
```

Each synchronous or asynchronous call created with an external token costs `1` credit. Credit deduction and task creation occur in the same SQLite transaction. Insufficient balance returns `402 Payment Required`; invalid requests are not charged, and asynchronous replays with the same `idempotency_key` are not charged again. Once a task is created, credits are not refunded for device execution failures, internal retries, or `/v1/invoke` timeouts. Calls using device secrets or the administrator console are not charged.

Make a synchronous call and receive the method result directly:

```http
POST /v1/invoke
Authorization: Bearer <CALL_TOKEN>
Content-Type: application/json

{
  "method": "wxchannels.contact.feed.list",
  "target_device_id": "mayfair-macbook",
  "args": {
    "username": "example@finder",
    "next_marker": ""
  }
}
```

`/v1/invoke` requires no `idempotency_key`, returns no task information, and waits up to 10 seconds. On success, its response body is the JSON returned by the device method. Execution failures return `502`; timeouts return `504`. A timeout does not cancel the internal task, so use the asynchronous interface for calls with side effects or calls that may take longer than 10 seconds.

Create an asynchronous task:

```http
POST /v1/call
Authorization: Bearer <CALL_TOKEN>
Content-Type: application/json

{
  "method": "wxchannels.fetch",
  "idempotency_key": "external-123",
  "args": {
    "url": "https://channels.weixin.qq.com/web/pages/feed?..."
  }
}
```

Asynchronously fetch the video list for a WeChat Channels account:

```http
POST /v1/call
Authorization: Bearer <CALL_TOKEN>
Content-Type: application/json

{
  "method": "wxchannels.contact.feed.list",
  "target_device_id": "mayfair-macbook",
  "args": {
    "username": "example@finder",
    "next_marker": ""
  }
}
```

Without `target_device_id`, external callers depend only on methods exposed by the Bridge and do not need to know its internal devices. When a target is specified, the Bridge verifies that it is online and has registered the method before scheduling the call.

Use the task ID from the submission response and the same call token to query the task:

```http
GET /v1/tasks/<task-id>
Authorization: Bearer <CALL_TOKEN>
```

`GET /v1/tasks` also returns only tasks published by the current call token. Callers cannot override their publisher identity through query parameters or headers.

## Local API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/bridge/status` | Connection status between the current device and its personal Bridge |
| `POST` | `/api/bridge/call` | Submit any method call using `method + args` |
| `POST` | `/api/bridge/tasks/wxchannels` | Submit a WeChat Channels parsing task |
| `POST` | `/api/bridge/tasks/download` | Submit a download task to a specific device |
| `GET` | `/api/bridge/tasks/:id` | Query a task, its content/result, and errors |
| `GET` | `/api/bridge/tasks?status=&limit=` | Query tasks published by the current device |

## Migrating Legacy Configurations

After upgrading, legacy external callers should stop using the shared `BRIDGE_TOKEN`. First log in to the administration interface with `BRIDGE_ADMIN_TOKEN`, create a separate call token for each caller, and replace its `Authorization` value. Leave `bridge.token` in device configurations unchanged.

When upgrading from a version without credit fields, existing dynamic call tokens are retained with an initial balance of `0`. After deploying the upgrade, add credits to these tokens in the administration interface before resuming external calls. Existing task and token identities remain unchanged.

A legacy configuration with exactly one `bridge.instances` entry can still connect temporarily. The new version uses its `url`, `clientId`, and token, while retaining compatibility with legacy paths and protocol fields. During migration, legacy `capabilities` booleans map to the corresponding methods; new configurations should use the general `bridge.methods` setting. Migrate to `bridge.url`, `bridge.deviceId`, `bridge.deviceName`, and `bridge.token`.

Legacy configurations with multiple instances fail immediately because each operating-system device now belongs to one personal Bridge. Higher-level applications handle multi-Bridge management.

The first upgrade from a legacy multi-`bridge.id` Worker activates a new singleton Durable Object. Devices re-register after reconnecting, but historical tasks from old Durable Objects are not automatically merged into the new Bridge.

## Reliability and Limitations

- Only one task is created for the same publisher and `idempotency_key`.
- Each external call task costs one credit. Insufficient balance returns `402`; queries and polling do not consume credits.
- Each device currently takes one task at a time. WeChat Channels parsing runs sequentially within the same operating system.
- Each call's args or result is limited to 1 MiB.
- Completed and failed tasks are retained for seven days.
- Tasks may execute again after lease expiry; executors must tolerate duplicate calls.
- `BRIDGE_TOKEN` is a high-privilege secret shared by all devices and belongs only in device configuration. Distribute separate call tokens to people and external systems.
- Dynamic call tokens can currently invoke all online `methods`. Finer-grained method authorization and rate limiting remain the responsibility of a future Bridge marketplace or higher-level gateway.
