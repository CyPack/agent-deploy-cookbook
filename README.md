# Agent & Service Deployment Cookbook

Battle-tested, **generic** deployment guides for self-hosting AI agents and Docker
services on a [Dokploy](https://dokploy.com) (Docker + Traefik + Swarm) PaaS.

These notes were distilled from real deployments and rewritten to be reusable by
anyone. They contain **no credentials, no private hosts, no personal data** — every
secret-shaped value is a placeholder you fill in yourself.

## What's inside

| Guide | What it covers |
|-------|----------------|
| [`dokploy-deployment-guide.md`](./dokploy-deployment-guide.md) | The general Dokploy **compose deployment golden path** — the workflow, the API/MCP gotchas that silently bite you, the REST API pattern that actually works, a production-grade compose checklist, an error table, and Playwright-based UI debugging. Start here. |
| [`agent-zero-deployment.md`](./agent-zero-deployment.md) | Deploying **Agent Zero** (`agent0ai/agent-zero`) — a Docker-only, self-improving agentic framework with a full Linux + desktop in one container. |
| [`hermes-agent-deployment.md`](./hermes-agent-deployment.md) | Installing **Hermes Agent** (`NousResearch/hermes-agent`) — a host-installed CLI/TUI agent (NOT a container by default). |

## Placeholder convention

Replace these with your own values everywhere they appear:

| Placeholder | Meaning |
|-------------|---------|
| `<DOKPLOY_URL>` | Your Dokploy base URL (default `http://localhost:3000`) |
| `<DOKPLOY_API_KEY>` | API token from Dokploy → Settings → API |
| `<HOST_LAN_IP>` | Your server's LAN IP (e.g. `192.168.x.x`) |
| `<SERVER_PUBLIC_IP>` | Your server's public IP |
| `<APP_DOMAIN>` | The domain you point at the service (e.g. `app.example.com`) |
| `<COMPOSE_ID>` / `<PROJECT_ID>` / `<ENV_ID>` | IDs returned by Dokploy when you create resources |
| `<APP_NAME>` | The app name (Dokploy appends a random suffix — read it back, don't hardcode) |

## Prerequisites

- A running Dokploy instance (Docker Swarm mode)
- `docker` CLI access on the host
- Either the Dokploy web UI, or the `dokploy-mcp` MCP server / REST API for automation

## Disclaimer

Provided as-is for educational/operational reference. Always review a compose file
and set strong, unique secrets before exposing any service to a network.
