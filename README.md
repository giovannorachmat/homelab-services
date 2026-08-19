# Homelab

This is a personal homelab running on a mini PC at home.

I use it to learn Docker, networking, reverse proxies, monitoring, and everything else a good DevOps engineer should know (though at the time of this writing, I'm not a DevOps engineer).

Everything runs on a single machine behind a Cloudflare DNS, accessible from anywhere via a subdomain.

## Hardware

| Spec        | Value                                            |
| ----------- | ------------------------------------------------ |
| **Machine** | Lenovo M715q (mini PC)                           |
| **CPU**     | AMD Ryzen 5 2400GE, 3.2 GHz, 4 cores / 8 threads |
| **RAM**     | 32 GB                                            |
| **Storage** | 1 TB SSD                                         |
| **OS**      | Ubuntu 24.04.4 LTS (Noble Numbat)                |

## Services

| Stack              | Image / Build                                                         | Host Ports                   | CPU / RAM Limits | Purpose                  | Current state |
| ------------------ | --------------------------------------------------------------------- | ---------------------------- | ---------------- | ------------------------ | ------------- |
| **9router**        | build `../../../9router/`                                             | 20128                        | 0.75 / 500M      | API routing & proxy      | ✅ Running    |
| **AdGuardHome**    | `adguard/adguardhome:latest`                                          | 53 TCP/UDP, 8080→80          | 0.5 / 250M       | DNS ad-blocker           | ✅ Running    |
| **Arcane**         | `ghcr.io/getarcaneapp/manager:latest`                                 | 3552                         | 0.5 / 1G         | Docker management UI     | ✅ Running    |
| **crawl4ai**       | `unclecode/crawl4ai:latest` + build `../../../../repo/crawl4ai-proxy` | internal only                | 1.5 / 2.25G      | Web crawling + proxy     | ✅ Running    |
| **Drawio**         | `jgraph/drawio`                                                       | internal only                | 0.5 / 256M       | Diagramming tool         | Defined       |
| **Floci**          | `floci/floci:latest`                                                  | 4566                         | 0.5 / 256M       | Local AWS emulator       | Defined       |
| **Headroom Proxy** | `ghcr.io/chopratejas/headroom:latest`                                 | 8787                         | 0.5 / 250M       | Headroom optimizer proxy | Defined       |
| **MCPO**           | `ghcr.io/open-webui/mcpo:main`                                        | internal only                | 0.5 / 250M       | MCP-to-OpenAPI proxy     | ✅ Running    |
| **Monitoring**     | Prometheus, Loki, Promtail, Grafana, Node Exporter                    | 3030, 9090, 9091, 9100, 3100 | 2 / 2.5G         | Metrics + logs           | Defined       |
| **NPM**            | `jc21/nginx-proxy-manager:2.15.1`                                     | 80, 81, 443                  | 0.5 / 250M       | Reverse proxy + SSL      | ✅ Running    |
| **Open Terminal**  | `ghcr.io/open-webui/open-terminal:latest`                             | 8000                         | 2 / 4G           | Browser terminal backend | ✅ Running    |
| **Open WebUI**     | `ghcr.io/open-webui/open-webui:main-slim`                             | internal only                | 1 / 2G           | Chat interface for LLMs  | ✅ Running    |
| **Tika**           | `apache/tika:latest`                                                  | 9998                         | 1 / 2G           | Content extraction       | ✅ Running    |

## Networking

Most containers share a single Docker bridge network called `private-net`. This lets them talk to each other by container name (e.g., NPM forwards to `open-webui:8080`).

```
                                  private-net
                    ┌─────────────────────────────────────┐
                    │--> adguardhome:80                   │
  Client -> NPM --> ┤--> open-webui:8080                  │
(Internet)          │--> crawl4ai-proxy:8000              │
                    │--> mcpo                             │
                    └─────────────────────────────────────┘
```

### Compose Template

Copy and fill in the blanks:

```yaml
services:
    <service-name>:
        container_name: <service-name>
        image: <image:tag>
        restart: unless-stopped

        env_file:
            - ../../runtime/<service>/.env

        environment:
            TZ: "Asia/Jakarta"
            # Add non-sensitive env vars here

        ports:
            - "<host_port>:<container_port>"

        volumes:
            - ../../runtime/<service>/data:/path/inside/container

        healthcheck:
            test: ["CMD-SHELL", "curl -f http://localhost:<port> || exit 1"]
            interval: 1m30s
            timeout: 10s
            retries: 5
            start_period: 10s

        depends_on:
            - <other-service>

        networks:
            - private-net

        deploy:
            resources:
                limits:
                    cpus: "0.5"
                    memory: 256M
                reservations:
                    cpus: "0.25"
                    memory: 128M

networks:
    private-net:
        external: true
```

**Field order rationale:**

| #   | Field                 | Why                                                                                                        |
| --- | --------------------- | ---------------------------------------------------------------------------------------------------------- |
| 1   | `container_name`      | Predictable name for `docker exec` and NPM                                                                 |
| 2   | `image`               | What to pull from Docker Hub                                                                               |
| 3   | `restart`             | `unless-stopped` = auto-restart on crash/reboot, but not if manually stopped                               |
| 4   | `env_file`            | Path to `.env` with secrets                                                                                |
| 5   | `environment`         | Non-sensitive vars (safe in compose)                                                                       |
| 6   | `ports`               | Host-to-container port mapping                                                                             |
| 7   | `volumes`             | Bind mounts for persistent data                                                                            |
| 8   | `healthcheck`         | How Docker knows the service is actually working                                                           |
| 9   | `depends_on`          | Start order                                                                                                |
| 10  | `networks`            | Which Docker network to join                                                                               |
| 11  | `deploy.resources`    | CPU and RAM limits                                                                                         |
| 12  | Top-level `networks:` | Declares the external network (critical — without `external: true`, Docker creates a new isolated network) |

---

## Troubleshooting

### 1. AdGuardHome admin panel direct access

**Symptom:** `http://YOUR.IP:8080` does not match NPM behavior or bypasses HTTPS.

**Cause:** The compose file maps host `8080` to container port `80`, but NPM should still forward to `adguardhome:80`.

**Fix:** Use NPM for HTTPS access. Remove `8080:80/tcp` from `services/adguardhome/docker-compose.yml` if you want to disable direct HTTP admin access.

---

### 2. AdGuardHome setup wizard vs admin panel

**Symptom:** Port 8080 doesn't load after starting AdGuardHome, but port 3000 works.

**Cause:** AdGuardHome uses port 3000 for the first-launch setup wizard. After setup completes, it switches to port 80.

**Fix:** Complete the setup wizard at `http://YOUR.IP:3000` first. Then the admin panel runs on port 80 (accessible via NPM).

---

### 3. NPM 502 Bad Gateway

**Symptom:** Proxy host returns "502 Bad Gateway".

**Cause:** Wrong forward port — used the host-mapped port instead of the container port.

**Fix:** In NPM, set Forward Host to the container name and Forward Port to the container's internal port. Verify with `docker inspect <container>` under "Ports".

---

### 4. .env file quotes breaking parsing

**Symptom:** Service starts but can't connect to providers. `.env` values have quotes like `HF_TOKEN="hf_abc..."` but the container reads them literally with quotes included.

**Cause:** Docker's `env_file` reads values literally. Quotes become part of the value. Worse, quotes around a value in a semicolon-separated list (like `$HF_TOKEN` in `OPENAI_API_KEYS`) break the entire chain.

**Fix:** Remove all quotes from `.env` values unless the value itself contains spaces.

---

### 5. Open WebUI "Missing Authentication header"

**Symptom:** Open WebUI loads but can't connect to AI providers. "Missing Authentication header" errors.

**Cause:** Wrong environment variable name (e.g., `OPEN_API_KEYS` instead of `OPENAI_API_KEYS`).

**Fix:** Double-check env var names against the official Open WebUI docs. If the DB already stored bad config, either delete `runtime/open-webui/data/webui.db` and re-setup, or manually fix it with SQLite:

```bash
cp runtime/open-webui/data/webui.db runtime/open-webui/data/webui.db.bak
sqlite3 runtime/open-webui/data/webui.db "SELECT * FROM config WHERE key LIKE 'api%';"
sqlite3 runtime/open-webui/data/webui.db "UPDATE config SET value = 'correct_url' WHERE key = 'api_base_url';"
docker compose -f services/open-webui/docker-compose.yml restart
```

---

### 6. Port conflict between services

**Symptom:** One of two services fails to start — can't bind to a port.

**Cause:** Both services try to map the same host port (e.g., port 9080).

**Fix:** Remove port mappings from one service and access it through NPM. Check existing port usage with: `ss -tlnp | grep :<port>`

**Note:** Host port 8080 is currently used by AdGuardHome. Open WebUI and Drawio also listen on container port 8080 internally, so expose them through NPM or use different host ports.

---

### 7. Container not connecting to private-net

**Symptom:** Container starts but can't reach other services by name. `docker exec -it <container> ping npm` fails.

**Cause:** Missing the top-level `networks:` declaration with `external: true` in the compose file. Without it, Docker creates a new isolated network with the same name.

**Fix:** Every compose file needs:

```yaml
# Inside the service:
networks:
    - private-net

# At the bottom of the file:
networks:
    private-net:
        external: true
```

---

_Last updated: 2026-08-09_
