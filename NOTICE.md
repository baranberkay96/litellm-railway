# Notice

## Upstream software

- **LiteLLM** — MIT for content outside `enterprise/`; `enterprise/` is under the LiteLLM enterprise license (enterprise features are not enabled by this template). Copyright (c) Berri AI. Image used: `ghcr.io/berriai/litellm-database:v1.104.2` (published by BerriAI on GHCR).
- **PostgreSQL** — PostgreSQL License. Image `ghcr.io/railwayapp-templates/postgres-ssl:18` (Railway).
- **Redis** — Redis 8 is available under RSALv2/SSPLv1/AGPLv3. Image `redis:8.2` (Docker official image).

## Verified against

| What | Source | Version / ref | Checked |
|---|---|---|---|
| Latest stable release and image tag | https://github.com/BerriAI/litellm/releases/tag/v1.104.2, GHCR manifest `berriai/litellm-database:v1.104.2` | v1.104.2 (2026-10-08) | 2026-10-10 |
| Image entrypoint / CMD | `docker/Dockerfile.database`, `docker/prod_entrypoint.sh` | v1.104.2 | 2026-10-10 |
| CLI env vars (`HOST`, `PORT`, `NUM_WORKERS`) | `litellm/proxy/proxy_cli.py` | v1.104.2 | 2026-10-10 |
| `cache_params` `os.environ/` resolution and cache `mode` | `litellm/proxy/proxy_server.py`, `litellm/caching/caching.py` | v1.104.2 | 2026-10-10 |
| Production recommendations | https://docs.litellm.ai/docs/proxy/prod | docs | 2026-10-10 |

## Adapted code

None. `services/litellm/Dockerfile` and `config.yaml` were written for this template; setting names follow LiteLLM's documented configuration keys. Postgres and Redis service definitions are copied from this repository's `_railway-base` files.

## Reference only (not copied)

- Railway marketplace templates `Lm9gxI` (LiteLLM), `litellm-proxy` and `litellm-database`: read for ideas only; no text, scripts or files copied. No license is attached to them.

## Trademarks

LiteLLM is a trademark of Berri AI; provider names belong to their respective owners. This is a community template and is not affiliated with or endorsed by Berri AI.
