# AGENTS.md

## Repo shape
- Docker Compose homelab, not an app monorepo: `services/*/docker-compose.yml` defines containers; `runtime/` holds persistent/private state.
- Treat compose files as the source of truth when README status/ports/resource numbers disagree; README service table is stale relative to `services/`.
- Most services join the external Docker network `private-net`; missing the top-level `networks: private-net: external: true` creates an isolated network and breaks container-name routing.
- Compose service keys differ from container names in a few places: npm's service key is `app` (container `npm`), and monitoring containers carry `-c` suffixes (`prometheus-c`, `grafana-c`, ...). `docker compose` commands use the service key; NPM proxy hosts use container names.
- Open WebUI expects an `ollama` container on `private-net` (`OLLAMA_BASE_URL=http://ollama:11434`), but there is no `services/ollama` compose; only `runtime/ollama/.env` exists here.

## Commands
- Start/stop/logs one service: `docker compose -f services/<service>/docker-compose.yml up -d` / `down` / `logs -f`
- Verify running containers/resources: `docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"` and `docker stats --no-stream`
- After editing any `runtime/**/.env`, run `./scripts/generate-env-examples.sh` to refresh committed `.env.example` files.
- Validate compose changes without deploying: `docker compose -f services/<service>/docker-compose.yml config`

## Secrets and state
- Never commit real `.env` files, `runtime/**/data/`, `runtime/**/conf/`, `runtime/**/work/`, `runtime/**/letsencrypt/`, `runtime/syncthing/config/`, or `runtime/certs/`; `.gitignore` encodes this split.
- Only `runtime/**/.env.example` is meant for git; the generator replaces values with placeholders but preserves comments/blank lines.
- `runtime/syncthing/config/config.xml` holds secrets (API key, device IDs) plus TLS private keys (`key.pem`, `https-key.pem`). The API key there drives Syncthing's `/rest/config` API, which is how to fix config or GUI auth without restarting the container.

## Compose conventions and gotchas
- NPM is the production reverse proxy on host ports 80/443/81; proxy hosts must forward to container names and container ports, not host-mapped ports.
- `services/9router` builds from sibling repo `../../../9router/`; `services/crawl4ai` builds `crawl4ai-proxy` from `../../../../repo/crawl4ai-proxy` (paths relative to the compose file; both live outside this repo).
- Every service defines `deploy.resources` limits/reservations; follow this pattern for new services (32GB RAM mini PC host).
- Docker `env_file` reads quotes literally; avoid quoting simple values in `.env`, especially semicolon-separated provider lists.
- Monitoring has both tracked config under `services/monitoring/config/` and runtime config under `runtime/monitoring/config/`; check the compose mounts before editing the wrong copy. Its promtail still mounts `runtime/airflow-docker/logs` and `runtime/vault/logs`, which no longer exist (leftovers from removed services).
- Syncthing is the one exception to the `runtime/` data convention: its data mount is the absolute host path `/home/giografi/syncthing` → `/data`. Folder paths set in the Syncthing GUI must be container paths (`/data/<name>`) to land in `~/syncthing`; host paths like `/home/giografi/...` fail inside the container. Adding a folder never requires editing the compose file.
- Syncthing replaced the native Debian package install (user systemd unit `syncthing.service` stopped + disabled); ports 8384/22000/21027 belong to the compose service. If they conflict, check `systemctl --user status syncthing`.

## Focused validation
- There is no repo-wide test/lint/typecheck setup; `docker compose config` (above) is the verification step.
- For port changes, check conflicts with `ss -tlnp | grep -E ':(80|443|9080|9443|<port>) '`.
- README.md has a Troubleshooting section covering known incidents (NPM 502, `.env` quotes, private-net isolation); check it before debugging those symptoms.
