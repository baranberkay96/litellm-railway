# LiteLLM proxy (database image) with a Railway-oriented config baked in.
# The upstream entrypoint (docker/prod_entrypoint.sh -> `litellm "$@"`) is kept;
# HOST, PORT and NUM_WORKERS are read from environment variables by the LiteLLM CLI.
FROM ghcr.io/berriai/litellm-database:v1.104.2

COPY config.yaml /app/config.railway.yaml

CMD ["--config", "/app/config.railway.yaml"]
