# Dokploy Compose Deployment — Golden Path & Gotchas

A practical, reusable guide for deploying Docker Compose stacks to a
[Dokploy](https://dokploy.com) PaaS (Docker + Traefik, running in **Docker Swarm mode**).
Written from real deployments — focused on the workflow that *actually works* and the
sharp edges that silently waste hours.

> All values like IPs, domains, IDs, and keys below are **placeholders**. See the
> placeholder table in the README.

---

## 1. Concept hierarchy

```
Server (physical / virtual host)
└── Project (logical grouping)
    └── Environment (production / staging / dev — one is created automatically)
        ├── Application (single container; git or docker-image source)
        └── Compose (a full docker-compose stack — multiple services)
            ├── Domain (custom domain + automatic Let's Encrypt SSL)
            ├── Deployment (deploy history)
            └── Backup (scheduled / manual)
```

Most operations need an ID chain: `project → environment → compose/application`.
When automating, **read IDs back from the create response** — never hardcode them.

---

## 2. The Compose deployment golden path

Use this for anything multi-service or anything that needs environment variables.

```
Step 1  Create project        → returns projectId + auto environmentId
Step 2  Create compose        → returns composeId + appName (with random suffix!)
Step 3  Set composeFile + sourceType=raw   ← via REST API (see §3, the MCP path is buggy)
Step 4  Deploy                → "Deployment queued"
Step 5  Wait, poll status     → composeStatus: running → done
Step 6  Verify container      → docker ps shows "Up" + port mapping
Step 7  Health/HTTP test      → curl returns expected code
```

### Why `sourceType` matters

When you create a compose resource, it defaults to `sourceType: "github"`. If you deploy
a **raw YAML** compose without changing this, you get a *"no repository configured"*-style
failure. You **must** set `sourceType: "raw"` (together with `composeFile`) before deploying.

---

## 3. The REST API pattern that works (and the MCP bugs that don't)

Several automation paths *look* like they succeed but silently do nothing. Know these:

| Operation | The trap | The fix |
|-----------|----------|---------|
| Updating `composeFile` via the compose-update MCP tool | The tool updates `name`/`appName`/`description` but **not** `composeFile` | Set it via REST API (below) |
| Setting env vars via a "save environment" tool | Env vars don't reach the Swarm container | Put them in the compose YAML `environment:` block |
| Importing compose as base64 | Parser errors ("not valid JSON" / "Cannot read properties of undefined") | **Don't use import.** Use the REST API `compose.update` |

**Authoritative REST recipe** (set `composeFile` + `sourceType` in one call):

```bash
DOKPLOY_KEY="<DOKPLOY_API_KEY>"        # from Dokploy → Settings → API

python3 - <<'PY'
import json, urllib.request, os
key = os.environ.get("DOKPLOY_KEY", "<DOKPLOY_API_KEY>")
compose = open("docker-compose.yml").read()
body = {
    "composeId":  "<COMPOSE_ID>",
    "name":       "<APP_NAME>",
    "appName":    "<APP_NAME>",   # the suffixed one from create response
    "composeFile": compose,
    "sourceType":  "raw",
}
def post(path, payload):
    req = urllib.request.Request(
        "<DOKPLOY_URL>" + path,
        data=json.dumps(payload).encode(),
        headers={"x-api-key": key, "Content-Type": "application/json"},
        method="POST")
    try:
        r = urllib.request.urlopen(req, timeout=30)
        return r.status, r.read().decode()
    except urllib.error.HTTPError as e:
        return e.code, e.read().decode()

# Plain path is simplest; tRPC path is the fallback (older versions)
s, t = post("/api/compose.update", body)
if s != 200:
    s, t = post("/api/trpc/compose.update", {"json": body})
print(s, t[:200])
PY
```

Hard-won rules, all verified on real Dokploy versions:

- **Auth header:** `x-api-key: <KEY>` — `Authorization: Bearer` is **rejected** (401).
- **Path:** `/api/compose.update` works on recent versions (plain JSON body). Older
  versions need `/api/trpc/compose.update` with the body wrapped as `{"json": {...}}`.
  Try plain first, fall back to tRPC.
- **Verify:** after setting, re-read `compose.one?composeId=<COMPOSE_ID>` and confirm
  `composeFile` is non-empty and `sourceType` is `raw`. Don't trust the write blindly.
- **API key source:** Dokploy → Settings → API. Cookie/session auth does not work from
  plain curl.

---

## 4. Production compose checklist

Apply this to every stack. Each item exists because skipping it caused a real failure.

| # | Rule | Why |
|---|------|-----|
| 1 | **Do NOT set `container_name`** | Dokploy manages naming (`{appName}-{service}-1`). A fixed name causes *"container name already in use"* on redeploy. |
| 2 | `restart: unless-stopped` | Survives crashes, still lets Dokploy stop it. |
| 3 | **Named volumes, not bind mounts** | Swarm + SELinux makes bind mounts permission-prone. |
| 4 | `healthcheck` per service | Enables monitoring + `depends_on: service_healthy`. |
| 5 | Generous `start_period` | DB ~10s, app ~15–30s (migrations need time). |
| 6 | `logging` driver + size limits | Prevents disk from filling (`max-size`, `max-file`). |
| 7 | Set `TZ` | Correct log timestamps. |
| 8 | `extra_hosts: ["host.docker.internal:host-gateway"]` | Reach host services from the container (mandatory on Linux). |
| 9 | **Don't expose DB ports** to the host | Internal-only is safer; avoids multi-DB port clashes. |
| 10 | `depends_on: { condition: service_healthy }` | App shouldn't start before its DB is ready. |
| 11 | Drop the obsolete `version:` key | It's deprecated and triggers warnings. |
| 12 | Real auth ON in production | `AUTH=none` + `CORS=*` = wide-open API. |

**Reference skeleton:**

```yaml
services:
  app:
    image: example/app:1.2.3          # pin a version, avoid bare :latest
    ports:
      - "<HOST_PORT>:80"
    volumes:
      - app_data:/data                # named volume
    environment:
      - TZ=Europe/Amsterdam
      - DATABASE_URL=postgres://app:<DB_PASSWORD>@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 2G                  # cap RAM so one service can't starve the host
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:80/health"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"

  db:
    image: postgres:16-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=<DB_PASSWORD>
      - POSTGRES_DB=app
      - TZ=Europe/Amsterdam
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

volumes:
  app_data:
  db_data:
```

---

## 5. Pre-built vs locally-built images

- **Pre-built (GHCR / Docker Hub):** simplest. `docker pull <registry>/<image>:<tag>` to
  confirm access, then reference `image:` directly in compose. Pin a specific tag.
- **Locally built image:** Dokploy uses `--pull always`, so it tries the registry and
  **won't find a purely local image**. Run a local registry and push to it:
  ```bash
  docker run -d -p 5000:5000 --name registry registry:2
  docker build -t localhost:5000/myapp:latest .
  docker push localhost:5000/myapp:latest
  # compose: image: localhost:5000/myapp:latest
  ```

### Volume naming

Dokploy creates volumes as `{appName}_{volumeName}`. If `appName` changes (new
project/compose), volume names change too and **old data won't auto-attach**. To reuse
data, either copy it:
```bash
docker run --rm -v old_vol:/from:ro -v new_vol:/to alpine cp -a /from/. /to/
```
or declare the old volume as external in the new compose.

---

## 6. Vite / SPA front-end gotchas

1. **`VITE_*` env vars are baked at BUILD time**, not runtime. Setting them in the compose
   `environment:` block does nothing. Pass them as build args:
   ```dockerfile
   ARG VITE_API_URL
   ENV VITE_API_URL=$VITE_API_URL
   RUN npm run build
   ```
   ```bash
   docker build --build-arg VITE_API_URL="https://<APP_DOMAIN>/api" -t localhost:5000/myapp .
   ```
2. **SPA routing needs an nginx catch-all**, or deep links 404:
   ```nginx
   location / { try_files $uri $uri/ /index.html; }
   ```
   When writing this inside a Dockerfile via `printf`, use **single quotes** so the shell
   doesn't expand `$uri`.

---

## 7. The "server is fine but the UI says it's broken" pattern

A very common class of bug: the **server** is healthy (curl returns 200, container is Up),
but the dashboard shows "connection failed". The fault is almost always in **layer 4 — the
browser**, where many apps store worker config (URLs, API keys) in `localStorage`, not on
the server.

```
Layer 4: Browser / dashboard UI  (localStorage config, JS errors)  ← check here when curl is OK
Layer 3: Application API          (auth, endpoint, port)
Layer 2: Docker container         (crash, missing env, port mapping)
Layer 1: System                   (port clash, disk, network)
```

Diagnosis order is usually `2 → 3 → 4 → 1`. When the server checks pass, **go look at the
browser** with a tool like Playwright instead of assuming it's fine:

```
1. Navigate to the URL (basic auth → http://user:pass@host:port/path)
2. Screenshot — what's actually on screen?
3. Read console messages — a "401 @ /api/version" means the dashboard's stored key is wrong
4. Inspect network requests — wrong port? missing auth header? 4xx/5xx?
5. Fix via the UI (correct the stored API key, save), re-check console
```

This is the single highest-value debugging habit for dashboard-style apps.

---

## 8. Common error table

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Unauthorized` (curl) | Wrong/missing key, or `Bearer` header | Use `x-api-key: <KEY>` |
| `404 Not Found` | Stale/wrong ID | Re-resolve IDs (list projects again) |
| `Compose not found` | Bad `composeId` | Re-read the environment contents |
| `Deploy failed` | Build error / image not found | Read the deploy log; check the image name |
| `Port conflict` | Port already bound | `docker ps` to find the owner; change the host port |
| `Volume permission denied` | Swarm + SELinux on a bind mount | Use a named volume |
| `version is obsolete` warning | Deprecated compose key | Remove the `version:` line |
| Container `Exited(1)`, empty log | Missing env vars | `docker inspect <c>` → check the `Env` section |
| Healthcheck fails but app works | `start_period` too short | Increase it (≥60s for heavy apps) |
| Deploy status "error" but containers run | Transient status desync | Re-set status to `done` via REST, no redeploy needed |
| Env change didn't reach container after redeploy | Known env-propagation quirk | `docker stop/rm` the service + `docker compose up -d --no-deps <service>` |
| `Unexpected end of JSON input` after a deploy/stop call | Response race | Re-read state with a `*.one` call |

---

## 9. Post-deploy verification (don't claim success without evidence)

```bash
# 1. Container up + ports mapped?
docker ps --filter "name=<APP_NAME>" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 2. HTTP health
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:<HOST_PORT>/

# 3. App logs sane?
docker logs <CONTAINER_NAME> --tail 30

# 4. For dashboard apps: open it in a browser and check the console for errors
```

Success criterion: `docker ps` → **Up**, and the HTTP endpoint returns the expected code.
"Should work" is not verified — run the command, read the output, then claim.

---

## 10. Quick reference — operation → steps

| Operation | Steps |
|-----------|-------|
| Compose deploy | create project → create compose → REST `compose.update` (raw + composeFile) → deploy → verify |
| Image-only app + env vars | use a compose stack (env via YAML, not a "save environment" call) |
| Redeploy | `compose-redeploy` / `application-redeploy` |
| Stop / start | `*-stop` / `*-start` |
| Fix status desync | REST `compose.update` with `composeStatus: "done"` |
| Custom domain + SSL | `domain-create` (host + https + letsencrypt) — then point DNS A record at `<SERVER_PUBLIC_IP>` |
| Inspect containers | `docker ps`, `docker logs`, `docker inspect` |
| Restart a Swarm service container | `docker-restartContainer` (not bare `docker restart`) |
