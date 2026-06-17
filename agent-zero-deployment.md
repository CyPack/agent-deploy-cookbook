# Deploying Agent Zero on Dokploy

[Agent Zero](https://github.com/agent0ai/agent-zero) (`agent0ai/agent-zero`) is an
open-source, self-improving agentic framework. Its defining trait: **it ships as a single
Docker container that contains a full Linux system** — a web UI, a real XFCE desktop in a
browser canvas, a private search engine (SearXNG), browser automation, and a multi-agent
hierarchy (a primary agent that spawns subordinate agents). It supports MCP and A2A.

## Deployment model — Docker only

> Agent Zero is distributed **exclusively as a Docker image. There is no bare-metal
> install path.** This makes it a natural fit for a Dokploy compose deployment.

- **Image:** `agent0ai/agent-zero:latest` (also a Kali-based "Hacking Edition" variant)
- **Image size:** large (~8–9 GB) because it bundles a whole desktop OS — the first pull
  can take **10–30 minutes**. This is normal; be patient and watch the deploy log.
- **Web UI:** container port `80`, map it to a host port (e.g. `50080`).
- **Persistence:** everything (settings, memory, projects, API keys) lives under `/a0`.
  Mount a volume there so it survives container replacement.

## Compose file (Dokploy-ready)

Applies the production checklist from the main guide — **no `container_name`**, a
**named volume**, a memory cap, timezone, and log rotation:

```yaml
services:
  agent-zero:
    image: agent0ai/agent-zero:latest
    ports:
      - "50080:80"
    volumes:
      - a0_data:/a0
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam
    deploy:
      resources:
        limits:
          memory: 4G        # cap it; the desktop stack can balloon under load
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"

volumes:
  a0_data:
```

Idle footprint is surprisingly small (tens of MB); the 4G limit is a ceiling for when the
agent actually drives the desktop/browser, not a reservation.

## Deploy steps (Dokploy)

Follow the [main golden path](./dokploy-deployment-guide.md#2-the-compose-deployment-golden-path):

```
1. Create project          → projectId + environmentId
2. Create compose          → composeId + appName (read the suffixed appName back)
3. REST API compose.update → set sourceType="raw" + composeFile (the YAML above)
4. compose-deploy          → "Deployment queued"
5. Poll: composeStatus running → done   (watch the deploy log; pull takes a while)
6. Verify:
     docker ps --filter name=agent-zero          # Up, 0.0.0.0:50080->80/tcp
     curl -s -o /dev/null -w "%{http_code}" http://<HOST>:50080/   # 200
```

The deploy log will show layer extraction (`Extracting ...MB`) during the long pull, then:
```
Container ...-agent-zero-1 Started
Docker Compose Deployed: ✅
```

## First-run configuration

Deliberately **do not bake API keys into the compose file.** Configure them in the web UI
on first launch — they persist in the `/a0` volume, keep secrets out of YAML, and sidestep
env-propagation quirks:

1. Open the web UI (`http://<HOST>:50080`, or via your private overlay network).
2. **Settings → model / API keys.**
3. Choose your model provider. Pick a provider you trust for the work you'll feed it —
   Agent Zero executes code and accumulates context across sessions.

## Security notes

1. **The web UI port is exposed on `0.0.0.0` by default** — i.e. to everyone on the
   network. Agent Zero is a genuinely powerful tool that **executes real code/commands**.
   If you only need local or private-overlay (e.g. Tailscale/WireGuard) access, bind it to
   loopback instead:
   ```yaml
   ports:
     - "127.0.0.1:50080:80"
   ```
   then reach it over your private overlay, and redeploy.
2. **Set up Agent Zero's own login/auth** before exposing it anywhere.
3. Because it can run arbitrary code, treat the container as privileged-ish — don't mount
   sensitive host paths or the Docker socket unless you understand the blast radius.

## Management

```bash
docker logs agent-zero-<suffix>-agent-zero-1 -f      # follow logs
# Dokploy: compose-stop / compose-start / compose-redeploy (by composeId)
```

To remove cleanly: stop the compose in Dokploy, then delete the project (decide whether to
keep or drop the `a0_data` volume — dropping it erases all settings/memory).
