# Cloudflare Durable Objects Deployment

This package provides generic Durable Objects build and deployment capabilities. It contains no application-specific Worker source, class names, bindings, secrets, or product configuration.

The caller is responsible for providing:

- A JavaScript module ready for upload;
- The Worker name, compatibility date, and entry module name;
- Durable Object bindings, exported classes, and storage types;
- Secrets to configure;
- Whether to enable a `workers.dev` URL.

`Deploy` uploads the caller-provided JavaScript, Durable Object declarations, and secrets through the Cloudflare REST API. It does not compile application-specific source code.

The current Bridge forwarding service's native JavaScript source and Worker + Pages deployment orchestration are in `internal/workers/bridge/index.js` and `internal/workers/bridge/deploy.go`, respectively. `cmd/deploy.go` only reads the CLI configuration, calls `bridge.Deploy`, and displays the result.
