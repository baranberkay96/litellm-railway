# LiteLLM v1 AI Gateway with Spend Tracking — Railway template spec

## Listing

- **Name:** LiteLLM v1 AI Gateway with Spend Tracking (LISTINGS.md name with the pinned major version inserted; LiteLLM tags releases as `v1.x.y`)
- **Description (80 chars):** One OpenAI-compatible API for 100+ LLMs with keys, budgets and rate limits. (unchanged)
- **Category:** AI/ML
- **Tags:** litellm, llm, ai-gateway, openai-compatible, proxy, budgets, rate-limiting

## Base market template

- **Chosen base: `Lm9gxI` "LiteLLM"** (125 active / 340 total projects): pinned official image, Postgres + Redis, readiness healthcheck, generated master and salt keys, drain time. Kept: topology, `/health/readiness`, `STORE_MODEL_IN_DB`, generated `sk-` keys, draining seconds, optional provider keys.
- Rejected from `Lm9gxI`: its ~9 KB inline Python start command (pre-deploy migration + hiding the Prisma CLI). It is clever but opaque, hard to upgrade and not licensed for reuse; this template keeps LiteLLM's own startup migration and a readable baked config instead.
- `litellm-proxy` (38 active): uses floating `main-stable` tag and no healthcheck on Redis/Postgres; took nothing beyond confirming `LITELLM_MODE=PRODUCTION`.
- `litellm-database` (21 active): unpinned `litellm/litellm-database` image, empty UI credentials, no healthcheck. Nothing taken.
- `_upstream`: none for this app.

## Pinned versions

| Service | Image / source | Tag | Release date | Checked |
|---|---|---|---|---|
| LiteLLM | `ghcr.io/berriai/litellm-database` (Dockerfile in `services/litellm`) | `v1.104.2` | 2026-10-08 | 2026-10-10 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl` | `18` | Railway base | 2026-10-10 |
| Redis | `redis` | `8.2` | Railway base | 2026-10-10 |

`v1.104.2` is the release GitHub marks as latest (non-prerelease). Newer `v1.105.0-rc.*` and `v1.106.0-dev.*` tags are pre-releases and were not used.

## Architecture

```
  clients / apps (OpenAI SDK, Langfuse-instrumented apps, n8n ...)
                      |
                HTTPS (public domain)
                      |
              [LiteLLM :4000]  -- NUM_WORKERS uvicorn processes
               |            |
     Prisma    |            |  redis-py (cache + router state)
               v            v
          [Postgres]     [Redis]
```

- LiteLLM is the only public service. It serves the OpenAI-compatible API (`/v1/...`), the Admin UI (`/ui`) and health endpoints.
- Postgres stores models, virtual keys, users, teams, budgets and spend logs. Prisma migrations run automatically when the proxy starts.
- Redis holds the response cache and the router's shared state (rate-limit counters, cooldowns), so limits stay correct across worker processes and replicas.
- Other Railway services can call the gateway privately at `http://${{LiteLLM.RAILWAY_PRIVATE_DOMAIN}}:4000` (the proxy binds `::`).

## Services

### LiteLLM

Built from `services/litellm/Dockerfile`: official `litellm-database` image plus `config.yaml` (Redis cache, router state, production settings). Upstream entrypoint unchanged.

- Source: repo `baranberkay96/litellm-railway`, root directory `/services/litellm` (Dockerfile build)
- Start command: image default
- Pre-deploy command: none
- Healthcheck: `/health/readiness` (timeout 300 s)
- Public domain: yes, target port `4000`
- TCP proxy: none
- Volume: none
- Restart policy: ON_FAILURE (max 10 retries); replicas: 1; cron: none

| Variable | Value | Description |
|---|---|---|
| `PORT` | `4000` | HTTP port of the proxy and Admin UI (public domain target). |
| `HOST` | `::` | Bind address; :: accepts IPv4 and IPv6 so other Railway services can call the gateway privately. |
| `NUM_WORKERS` | `2` | Uvicorn worker processes. ~0.6-1 GB RAM each; for more throughput prefer adding replicas. |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres for models, virtual keys, teams, budgets and spend logs. Migrations run on start. |
| `LITELLM_MASTER_KEY` | `sk-${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Admin API key (must start with sk-). Use it to create virtual keys; keep private. |
| `LITELLM_SALT_KEY` | `sk-${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Encrypts provider credentials stored in Postgres. Generated once; never change it after adding models. |
| `UI_USERNAME` | `admin` | Admin UI username (/ui). |
| `UI_PASSWORD` | `${{secret(24, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Admin UI password. Generated; read it from this variable. |
| `STORE_MODEL_IN_DB` | `True` | Lets you add and edit models from the Admin UI (stored encrypted in Postgres). |
| `REDIS_HOST` | `${{Redis.REDISHOST}}` | Redis host for response caching and shared router state (rate limits, cooldowns) across workers/replicas. |
| `REDIS_PORT` | `${{Redis.REDISPORT}}` | Redis port. |
| `REDIS_PASSWORD` | `${{Redis.REDIS_PASSWORD}}` | Redis password. |
| `LITELLM_CACHE_MODE` | `default_off` | Response cache mode: default_off = opt-in per request (cache: {use-cache: true}); default_on = cache every identical request. |
| `LITELLM_MODE` | `PRODUCTION` | Production mode: no .env loading inside the container. |
| `LITELLM_LOG` | `INFO` | Log level (DEBUG, INFO, WARNING, ERROR). Use ERROR at high request volume. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `120` | Seconds the old deployment keeps serving in-flight (streaming) requests during a redeploy. |
| `OPENAI_API_KEY` | `` | Optional. Default OpenAI key used when a model has no key of its own. |
| `ANTHROPIC_API_KEY` | `` | Optional. Default Anthropic key. |
| `GEMINI_API_KEY` | `` | Optional. Default Google AI Studio (Gemini) key. |
| `OPENROUTER_API_KEY` | `` | Optional. Default OpenRouter key. |

### Postgres

Railway base Postgres (`_railway-base/postgres.json`, verbatim).

- Source: image `ghcr.io/railwayapp-templates/postgres-ssl:18`
- Start command: image default
- Pre-deploy command: none
- Healthcheck: none (no HTTP listener / not applicable)
- Public domain: no
- TCP proxy: none
- Volume: `/var/lib/postgresql/data`
- Restart policy: ON_FAILURE (max 10 retries); replicas: 1; cron: none

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Location where the database will be initialized |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `PGPORT` | `5432` | Port to connect to Postgres. |
| `PGUSER` | `${{ POSTGRES_USER }}` | Required variable for Data panel |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Required variable for the data panel. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Required variable for Data panel |
| `POSTGRES_DB` | `railway` | Default database created when image is started. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | URL to connect to Postgres database. |
| `POSTGRES_USER` | `postgres` | User to connect to Postgres DB |
| `SSL_CERT_DAYS` | `820` | SSL certificate expiry in days. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Password to connect to DB |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Allow Postgres to cleanly shut down |

### Redis

Railway base Redis (`_railway-base/redis.json`, verbatim).

- Source: image `redis:8.2`
- Start command: `/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH"`
- Pre-deploy command: none
- Healthcheck: none (no HTTP listener / not applicable)
- Public domain: no
- TCP proxy: none
- Volume: `/data`
- Restart policy: ON_FAILURE (max 10 retries); replicas: 1; cron: none

| Variable | Value | Description |
|---|---|---|
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private network hostname of the Redis service, only resolvable from services in the same environment |
| `REDISPORT` | `6379` | Port that Redis listens on |
| `REDISUSER` | `default` | Username for authenticating with Redis |
| `REDIS_URL` | `redis://${{ REDISUSER }}:${{ REDIS_PASSWORD }}@${{ REDISHOST }}:${{ REDISPORT }}` | Connection string for connecting to Redis using the private network |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | Alias of REDIS_PASSWORD for clients that expect the unseparated name |
| `REDIS_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Randomly generated password for authenticating with Redis |

## Improvements over market templates

1. **Current stable release pinned** (`v1.104.2`, 2026-10-08) instead of `v1.100.4` / `main-stable` / unpinned images.
2. **Redis actually used for caching and router state.** Market templates set `REDIS_URL`/`REDIS_HOST` but without `litellm_settings.cache` and `router_settings.redis_*` LiteLLM does not enable the Redis cache or share rate-limit state. The baked `config.yaml` turns both on (cache opt-in per request by default, switchable with `LITELLM_CACHE_MODE`).
3. **Production settings from LiteLLM's production guide:** `LITELLM_MODE=PRODUCTION`, batched spend writes (`proxy_batch_write_at: 60`), bounded DB pool per worker, 600 s request timeout, JSON logs.
4. **Sensible worker count:** `NUM_WORKERS=2` (env-driven, no rebuild needed), with guidance to scale via replicas.
5. **Admin UI credentials generated** (`UI_USERNAME=admin`, generated `UI_PASSWORD`) instead of empty values or reusing the master key.
6. **IPv6-safe private access:** binds `::` so other services in the project can use the gateway over the private network (market enforced `0.0.0.0`).
7. **Readable, forkable configuration** (`services/litellm/config.yaml`) instead of a 9 KB embedded script.
8. Railway base Postgres 18 / Redis 8.2 definitions, no public TCP proxies on the databases.

## Resource estimate

| Service | Idle RAM | Typical RAM | Typical vCPU | Volume |
|---|---|---|---|---|
| LiteLLM (2 workers) | 1.2 GB | 2.0 GB | 0.5 | – |
| Postgres | 0.1 GB | 0.3 GB | 0.1 | 1–5 GB (spend logs grow with traffic) |
| Redis | 0.02 GB | 0.15 GB | 0.02 | < 0.5 GB |
| **Total** | **~1.3 GB** | **~2.5 GB** | **~0.6** | **~1.5–5.5 GB** |

- **Idle:** 1.3 GB × $10 + ~0.1 vCPU × $20 + 1.5 GB × $0.15 ≈ **$15/month**.
- **Typical (team gateway, tens of thousands of requests/day):** 2.5 GB × $10 + 0.6 vCPU × $20 + 5 GB × $0.15 ≈ **$38/month**. LLM provider charges are billed by the providers, not Railway.

**Minimum plan: Hobby.** A single LiteLLM worker needs roughly 0.8-1 GB at startup, so Trial's 1 GB/service only works with `NUM_WORKERS=1` and little headroom. Pro is recommended for replicas and higher limits.

## Verification checklist

1. `LiteLLM` healthcheck `/health/readiness` returns 200 and reports the database as connected and the Redis cache as configured.
2. `GET https://<domain>/health/liveliness` returns 200.
3. Open `https://<domain>/ui`, log in with `UI_USERNAME` / `UI_PASSWORD`, add a model (e.g. an OpenAI model with your key).
4. Create a virtual key in the UI with a small budget; call `POST /v1/chat/completions` with it; spend appears under Usage after ~60 s (batched writes).
5. Send the same request twice with `"cache": {"use-cache": true}` — the second response is a cache hit (cache headers / Usage view show it was served from Redis).
6. Set an RPM limit on the key, exceed it, and get HTTP 429 — repeat with 2 replicas to confirm limits are shared via Redis.
7. Redeploy Postgres and LiteLLM: models, keys and spend history persist.

## Open questions

- **Cache default:** `LITELLM_CACHE_MODE=default_off` keeps answers uncached unless a request opts in. Set `default_on` if you want every identical request cached for 1 hour.
- **Startup memory:** LiteLLM runs Prisma migrations at boot; with several replicas each replica runs `prisma migrate deploy` (safe: Prisma uses an advisory lock), which briefly raises memory.
- **Postgres max_connections:** each worker opens up to 10 connections. With many replicas × workers, raise Postgres `max_connections` or lower `database_connection_pool_limit`.
- Not deploy-tested in this package (local files only).
