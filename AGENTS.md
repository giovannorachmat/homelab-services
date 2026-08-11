# AGENTS.md

## Repo shape
- This is a Docker Compose homelab, not an app monorepo: `services/*/docker-compose.yml` defines containers; `runtime/` holds persistent/private state.
- Treat compose files as the source of truth when README status/ports/resource numbers disagree; README service counts are stale relative to `services/`.
- Most services join the external Docker network `private-net`; missing `networks: private-net: external: true` creates an isolated network and breaks container-name routing.

## Commands
- Start one service: `docker compose -f services/<service>/docker-compose.yml up -d`
- Stop one service: `docker compose -f services/<service>/docker-compose.yml down`
- Follow logs: `docker compose -f services/<service>/docker-compose.yml logs -f`
- Verify running containers/resources: `docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"` and `docker stats --no-stream`
- After editing any `runtime/**/.env`, run `./scripts/generate-env-examples.sh` to refresh committed `.env.example` files.

## Secrets and state
- Never commit real `.env` files, `runtime/**/data/`, `runtime/**/conf/`, `runtime/**/work/`, `runtime/**/letsencrypt/`, or `runtime/certs/`; `.gitignore` is designed around this split.
- Only `runtime/**/.env.example` is meant for git; the generator replaces values with placeholders but preserves comments/blank lines.

## Compose conventions and gotchas
- NPM is the production reverse proxy on host ports 80/443/81; proxy hosts must forward to container names and container ports, not host-mapped ports.
- `services/9router` builds from sibling repo `../../../9router/`; `services/crawl4ai` builds `crawl4ai-proxy` from `../../../../repo/crawl4ai-proxy`.
- `services/firecrawl` is unusually heavy and has an internal `backend` bridge plus `private-net`; do not start it casually on the 16GB host.
- Keep resource limits conservative; this runs on a 16GB RAM mini PC and not all defined services can run at once.
- Docker `env_file` reads quotes literally; avoid quoting simple values in `.env`, especially semicolon-separated provider lists.
- Monitoring has both tracked config under `services/monitoring/config/` and runtime config under `runtime/monitoring/config/`; check mounts before editing the wrong copy.

## Focused validation
- There is no repo-wide test/lint/typecheck setup; validate compose changes with `docker compose -f services/<service>/docker-compose.yml config` before suggesting deploy.
- For port changes, check conflicts with `ss -tlnp | grep -E ':(80|443|9080|9443|<port>) '`. 
