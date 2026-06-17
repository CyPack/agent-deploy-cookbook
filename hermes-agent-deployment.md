# Installing Hermes Agent

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (`NousResearch/hermes-agent`)
is an open-source, self-improving AI agent: a built-in learning loop that creates skills
from experience, persistent cross-session memory, an 18-platform messaging gateway,
multiple terminal backends, cron scheduling, and MCP support. It is model-agnostic (works
with many providers via OpenRouter and others — bring your own keys).

## Deployment model — host CLI, NOT a container (by default)

This is the key difference from container-only agents: **the default install is a native
CLI/TUI on the host**, not a Docker deployment.

- The one-line installer sets up a native app under `~/.hermes` (data/config/code) and
  links a `hermes` command into `~/.local/bin`.
- It bootstraps its own dependencies (uv, Python 3.11, Node.js, ripgrep, ffmpeg, a
  portable Git Bash on Windows) — no manual prereqs.
- The repo *also* ships an optional `Dockerfile` + `docker-compose.yml`, so you **can**
  containerize the whole gateway+agent stack if you prefer — but it isn't required.

### "Terminal backends" ≠ where Hermes is deployed

Hermes lists several terminal backends (local, Docker, SSH, Singularity, Modal, Daytona).
These are **where the agent executes commands in isolation**, not where Hermes itself
runs. Hermes lives on the host; it can *run a task* inside a Docker sandbox. Don't confuse
the two.

## Install (host CLI)

```bash
# Inspect first (good hygiene for any curl|bash installer):
curl -fsSL https://hermes-agent.nousresearch.com/install.sh -o /tmp/hermes-install.sh
less /tmp/hermes-install.sh        # skim: install dir, sudo use, downloaded domains

# Run it. --skip-setup installs without launching the interactive model/key wizard,
# so you can configure the provider yourself afterward.
bash /tmp/hermes-install.sh --skip-setup
```

What the installer does (so there are no surprises):
- Installs into `~/.hermes` (data) + `~/.hermes/hermes-agent` (code) and links
  `~/.local/bin/hermes`. Stays in your home dir; doesn't touch system-wide paths.
- `sudo` is used **only** for optional system packages (e.g. `git`) via your package
  manager, and it asks first.
- Pulls from reputable hosts only (the project's domain, `astral.sh/uv`, `nodejs.org`,
  `pypi.org`). No telemetry.

## Verify

```bash
~/.local/bin/hermes --version          # e.g. "Hermes Agent vX.Y.Z ... Up to date"
ls ~/.hermes/                          # config.yaml, hermes-agent/, skills/, sessions/...
```

If `hermes` isn't found in a new shell, ensure `~/.local/bin` is on your `PATH`
(bash/zsh: add to your rc file; fish: `fish_add_path ~/.local/bin`).

## Configure & run

```bash
hermes setup          # configure API key(s) + model provider
hermes                # start an interactive session
hermes gateway install  # (optional) install the messaging + cron gateway service
hermes update         # update to the latest version
```

When choosing a model provider, pick one you trust for the data you'll route through it —
Hermes is persistent and builds memory across sessions.

## Uninstall / roll back

```bash
rm -rf ~/.hermes ~/.local/bin/hermes
```

## When to containerize instead

If you want Hermes managed alongside other services (e.g. on Dokploy), use the repo's
`docker-compose.yml` and follow the [Dokploy golden path](./dokploy-deployment-guide.md).
For a single personal machine, the host CLI install is simpler and lighter.
