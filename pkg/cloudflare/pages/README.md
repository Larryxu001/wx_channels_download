# Cloudflare Pages Deployment

This package supports the complete deployment flow for Direct Upload Pages projects through the Cloudflare REST API:

- Look up, create, or update Pages projects;
- Configure production/preview secrets and Service Bindings;
- Validate, hash, group into buckets, and concurrently upload static assets;
- Upload advanced-mode `_worker.js` and `_routes.json`;
- Create a production deployment and return the project, deployment ID, and URL.

The caller must explicitly provide an Account ID and API token. The package does not store credentials. The API token requires Pages:Edit permissions.
