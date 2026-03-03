# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

LiteLLM Proxy stack — a Docker-based LLM gateway that routes OpenAI-compatible requests to local llama-server backends with ordered fallback to cloud APIs (OpenAI, Anthropic). Consumed by Open WebUI, LangChain, and other tools.

## Commands

```bash
# Start / stop
docker compose up -d
docker compose down

# Status and logs
docker compose ps
docker compose logs -f

# Apply config.yaml or .env changes
docker compose restart

# Update LiteLLM image
docker compose pull && docker compose up -d

# Health checks (replace sk-xxxxx with LITELLM_MASTER_KEY)
curl.exe http://localhost:4000/health/readiness
curl.exe http://localhost:4000/health -H 'Authorization: Bearer sk-xxxxx'
```

## Architecture

```
Clients (Open WebUI, LangChain, etc.)
  │
  └──→ LiteLLM Proxy (Docker, port 4000)
         │  config.yaml  ← model routing + fallback order
         │  .env         ← secrets, endpoints, model names
         │
         ├──→ llama-server on GPU host   (LLAMA_SERVER_URL, primary)
         ├──→ llama-server on Mac        (PORTABLE_LLM_SERVER_URL, secondary)
         └──→ OpenAI / Anthropic         (CLOUD_API_KEY, cloud fallback)
```

### Key Files

| File | Role |
|------|------|
| `config.yaml` | LiteLLM model_list with ordered fallback routing, router/general/litellm settings. All values resolve from env vars via `os.environ/` syntax. |
| `.env` | Secrets and endpoint URLs (gitignored). Copy from `.env.example`. |
| `docker-compose.yml` | Single `litellm-proxy` service. Mounts config.yaml read-only, injects a custom CA cert, runs healthcheck. |
| `docker-compose.override.yml` | Joins litellm-proxy to Open WebUI's Docker network for same-host co-location. Delete if running on a separate host. |

### Model Routing (config.yaml)

Three virtual model names, each with ordered fallback (`order: N` + `enable_pre_call_checks: true`):

| Virtual model | Purpose | Fallback chain |
|---------------|---------|----------------|
| `gpt-oss-120b` | Direct access from Open WebUI | GPU host only |
| `reasoner` | Primary inference | GPU host → Cloud |
| `judge` | Evaluation (LangChain Judge chain) | Mac → GPU host → Cloud |

Actual backend model names and API keys are environment variables, not hardcoded.

### Custom CA Certificate

The entrypoint appends a CA cert (`CA_CERT_PATH` env var) to the system trust store before starting LiteLLM. This enables TLS connections to local servers using a private CA.

## Editing Guidelines

- `config.yaml` changes require `docker compose restart` to take effect.
- All endpoints, model identifiers, and API keys must be environment variables — never hardcode secrets or host-specific values in config.yaml.
- The `order` field on litellm_params controls fallback priority (lower = preferred). This only works with `enable_pre_call_checks: true` in router_settings.

## LangChain Project: Phase Workflow

This repo participates in the LangChain phased implementation.
Canonical rules: `../ai-specs/CLAUDE.md` § "LangChain Project: Phase Workflow"

Phase handoffs and completion reports are stored in `.context-private/`.
