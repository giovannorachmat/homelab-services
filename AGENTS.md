# AGENTS.md

## Repo shape
- This is a Docker Compose homelab, not an app monorepo: `services/*/docker-compose.yml` defines containers; `runtime/` holds persistent/private state.
- Treat compose files as the source of truth when README status/ports/resource numbers disagree; README service table is stale relative to `services/`.
- Most services join the external Docker network `private-net`; missing `networks: private-net: external: true` creates an isolated network and breaks container-name routing.
- `services/homepage/` exists but has no compose file yet.
- Open WebUI expects an `ollama` container on `private-net`, but there is no `services/ollama` compose; Ollama runs outside this repo (only `runtime/ollama/.env` is here).

## Commands
- Start one service: `docker compose -f services/<service>/docker-compose.yml up -d`
- Stop one service: `docker compose -f services/<service>/docker-compose.yml down`
- Follow logs: `docker compose -f services/<service>/docker-compose.yml logs -f`
- Verify running containers/resources: `docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"` and `docker stats --no-stream`
- After editing any `runtime/**/.env`, run `./scripts/generate-env-examples.sh` to refresh committed `.env.example` files.
- Validate compose changes without deploying: `docker compose -f services/<service>/docker-compose.yml config`

## Secrets and state
- Never commit real `.env` files, `runtime/**/data/`, `runtime/**/conf/`, `runtime/**/work/`, `runtime/**/letsencrypt/`, or `runtime/certs/`; `.gitignore` is designed around this split.
- Only `runtime/**/.env.example` is meant for git; the generator replaces values with placeholders but preserves comments/blank lines.

## Compose conventions and gotchas
- NPM is the production reverse proxy on host ports 80/443/81; proxy hosts must forward to container names and container ports, not host-mapped ports.
- `services/9router` builds from sibling repo `../../../9router/`; `services/crawl4ai` builds `crawl4ai-proxy` from `../../../../repo/crawl4ai-proxy` (paths relative to the compose file; both live outside this repo).
- Every service defines `deploy.resources` limits/reservations; follow this pattern for new services (32GB RAM mini PC host).
- Docker `env_file` reads quotes literally; avoid quoting simple values in `.env`, especially semicolon-separated provider lists.
- Monitoring has both tracked config under `services/monitoring/config/` and runtime config under `runtime/monitoring/config/`; check the compose mounts before editing the wrong copy.

## Focused validation
- There is no repo-wide test/lint/typecheck setup; `docker compose config` (above) is the verification step.
- For port changes, check conflicts with `ss -tlnp | grep -E ':(80|443|9080|9443|<port>) '`.
- README.md has a Troubleshooting section covering known incidents (NPM 502, `.env` quotes, private-net isolation); check it before debugging those symptoms.
