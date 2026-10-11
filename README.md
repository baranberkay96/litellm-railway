# Deploy and Host LiteLLM with Railway

LiteLLM is an open-source AI gateway that puts OpenAI, Anthropic, Gemini, Bedrock, Azure, OpenRouter and 100+ other model providers behind one OpenAI-compatible API. This community template runs the LiteLLM proxy with Postgres for keys and spend tracking and Redis for caching and shared rate limits.

## About Hosting LiteLLM

The LiteLLM proxy is a Python service that translates OpenAI-style requests to each provider's API, then records who called which model and what it cost. Running it as a shared gateway needs a database for virtual keys, teams, budgets and spend logs, and a shared cache so rate limits and cooldowns stay consistent when the proxy runs several processes or replicas. This template deploys the official LiteLLM database image with a small production configuration: models are managed from the Admin UI, spend is written in batches, the Redis cache and router state are enabled, and the master key, salt key and Admin UI password are generated for you at deploy time.

## Common Use Cases

- Give every team or application its own virtual key with a monthly budget, model allow-list and RPM/TPM limits.
- Switch between OpenAI, Anthropic, Gemini or self-hosted models without changing application code.
- Load-balance and fail over across several deployments or API keys of the same model.
- Track LLM spend per key, user and team, and export it to observability tools such as Langfuse.
- Offer one internal OpenAI-compatible endpoint to tools like n8n, Open WebUI or coding assistants.

## Dependencies for LiteLLM Hosting

- LiteLLM proxy image `ghcr.io/berriai/litellm-database:v1.104.2`
- PostgreSQL (Railway Postgres 18)
- Redis (Railway Redis 8.2)
- At least one LLM provider API key (added in the Admin UI or as an optional variable)

### Deployment Dependencies

- LiteLLM proxy docs: https://docs.litellm.ai/docs/simple_proxy
- Production best practices: https://docs.litellm.ai/docs/proxy/prod
- Caching: https://docs.litellm.ai/docs/proxy/caching
- Virtual keys and budgets: https://docs.litellm.ai/docs/proxy/virtual_keys
- Source and releases: https://github.com/BerriAI/litellm/releases

### Implementation Details

| Service | Role | Public | Storage |
|---|---|---|---|
| LiteLLM | OpenAI-compatible API, Admin UI (`/ui`) | Yes (port 4000) | – |
| Postgres | Models, keys, teams, budgets, spend logs | No | Volume |
| Redis | Response cache, rate-limit and router state | No | Volume |

**First login**

1. Wait for the LiteLLM healthcheck (`/health/readiness`) to pass.
2. Open `https://<your-domain>/ui` and sign in with `UI_USERNAME` (`admin`) and the generated `UI_PASSWORD` from the LiteLLM service variables.
3. Add a model under Models (paste the provider key there, or set the optional `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GEMINI_API_KEY` / `OPENROUTER_API_KEY` variables).
4. Create a virtual key and call `https://<your-domain>/v1/chat/completions` with it. Keep `LITELLM_MASTER_KEY` for administration only.

**Caching:** the Redis cache is opt-in per request by default (`LITELLM_CACHE_MODE=default_off`); send `"cache": {"use-cache": true}` or set the variable to `default_on`.

**Scaling:** increase `NUM_WORKERS` for more processes in one container, or add replicas to the LiteLLM service; Redis keeps rate limits consistent across them. Each worker uses up to 10 Postgres connections.

**Pinning and upgrades:** the version is set in `services/litellm/Dockerfile` (`FROM ghcr.io/berriai/litellm-database:v1.104.2`). Change the tag to a newer stable release and redeploy; database migrations run automatically on start. Never change `LITELLM_SALT_KEY` after models have been added.

### Why Deploy LiteLLM on Railway?

Railway gives the gateway a public HTTPS domain, private networking to its Postgres and Redis, generated secrets and usage-based billing, and other services in the same project can reach it privately. Scaling is a replica or worker setting, and upgrades are a one-line tag change.
