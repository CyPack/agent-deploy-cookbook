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

## Troubleshooting: model configuration

### Symptom — raw `model code cannot be empty` / `model=/` traceback in chat
You set a model in **Configure Models**, the UI shows it saved, but every chat fails with a raw
litellm/Python traceback such as:

```
litellm ... OpenAIException - model:The model code cannot be empty.   # z.ai code 1214
# or, with OpenAI:  BadRequestError ... model=/
```

and background memory calls throw an auth error for a provider you thought you had replaced.

### Root cause — profile scope shadows global (no merge)
Agent Zero resolves plugin config (including `_model_config`) by **first match across scopes**, and
the **active agent profile scope is checked before global**, with **no deep-merge**. So if
`settings.json` has `agent_profile: "developer"` and a profile config exists at:

```
usr/agents/<profile>/plugins/_model_config/config.json
```

that file **completely shadows** the global `usr/plugins/_model_config/config.json` that the UI
writes to. If the profile file has an empty `chat_model.name` (or a stale `utility_model`
provider), runtime builds `f"{provider}/{model}"` = `"<provider>/"` and the provider rejects the
empty model. **The UI shows global; runtime reads the profile** — that mismatch is the trap, and
the user only sees a raw provider traceback with no pointer to the cause. (Upstream discussion:
`agent0ai/agent-zero#1392`.)

### Diagnose
```bash
C=$(docker ps --format '{{.Names}}' | grep agent-zero | head -1)
# which profile is active?
docker exec "$C" python3 -c "import json;print(json.load(open('/a0/usr/settings.json'))['agent_profile'])"
# what does the ACTIVE profile resolve to? (the file runtime actually reads)
docker exec "$C" cat /a0/usr/agents/<profile>/plugins/_model_config/config.json
# time-bounded logs are the real evidence (the UI keeps stale error messages in the DOM):
docker logs --since 100s "$C" 2>&1 | grep -iE "1214|model code|openrouter|glm|error"
```

### Fix
Edit the **active profile** config (not just global), setting both slots correctly — e.g. for a
z.ai GLM Coding Plan:

```json
{ "chat_model":    { "provider": "zai_coding", "name": "glm-5.2" },
  "utility_model": { "provider": "zai_coding", "name": "glm-5.2" } }
```

…or delete the profile `config.json` so resolution falls back to global. Model resolution is
call-time, so a restart is usually unnecessary.

### z.ai GLM specifics (worth knowing)
- Use provider **`Z.AI Coding`** (endpoint `https://api.z.ai/api/coding/paas/v4`) for a **Coding
  Plan** key; plain **`Z.AI`** (`/api/paas/v4`) is for pay-as-you-go. Wrong endpoint → error
  `1113`, not `1214`.
- The model name is **bare** (`glm-5.2`). Agent Zero is litellm/OpenAI-compatible and prepends the
  `openai/` prefix itself — writing `openai/glm-5.2` or `zai/glm-5.2` double-prefixes it.
- Agent Zero talks the **OpenAI-compatible** endpoint (litellm speaks OpenAI). Don't point it at
  z.ai's Anthropic endpoint (`/api/anthropic`) — that one is for Claude-Code-style tools.
- An API key entered once on a `Z.AI Coding` slot is **shared** across every slot using that provider.

### Verifying it actually works
Don't trust "what model are you?" — the system prompt can inject a model identity, so the model may
parrot the wrong name. Trust the **time-bounded log**: send one chat message, then
`docker logs --since 100s` should show the call succeed with **zero** `1214`/`openrouter` lines.
Old error messages linger in the chat DOM, so always scope the log by time.

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
