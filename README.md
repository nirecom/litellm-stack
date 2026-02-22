# LiteLLM Stack

LLM Gateway for routing requests to local and cloud LLM backends with automatic fallback and health checking.

## Architecture

```
Open WebUI ───┐
LangChain ────┤──→ LiteLLM Proxy ──→ llama-server on GPU host (local)
Other tools ──┘         │          ──→ llama-server on Mac (local)
                        └────────────→ OpenAI / Anthropic (cloud fallback)
```

## Quick Start

```bash
git clone <this-repo> && cd litellm-stack
cp .env.example .env

# Generate a master key
echo "sk-$(openssl rand -hex 24)"
# Paste the generated key into .env as LITELLM_MASTER_KEY

vi .env  # Fill in all values
docker compose up -d
```

## Verify

```bash
# Health check
curl http://localhost:4000/health/readiness

# Test a model (replace sk-xxxxx with your LITELLM_MASTER_KEY)
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-xxxxx" \
  -d '{"model": "gpt-oss-120b", "messages": [{"role": "user", "content": "Hello"}]}'

# Check which backends are healthy
curl http://localhost:4000/health \
  -H "Authorization: Bearer sk-xxxxx"
```

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `LITELLM_MASTER_KEY` | Admin API key (must start with `sk-`) | `sk-a3b7f2...` |
| `LITELLM_PORT` | Proxy port (default: 4000) | `4000` |
| `LLAMA_SERVER_URL` | Primary llama-server endpoint | `http://<server-ip>:8080/v1` |
| `PORTABLE_LLM_SERVER_URL` | Portable host llama-server (optional) | `https://<server-ip>:8443/v1` |
| `OPENAI_API_KEY` | OpenAI API key for cloud fallback (optional) | `sk-...` |
| `ANTHROPIC_API_KEY` | Anthropic API key for cloud fallback (optional) | `sk-ant-...` |

## Model Routing

| model_name | Purpose | Fallback order |
|------------|---------|----------------|
| `gpt-oss-120b` | Direct model access from Open WebUI | GPU host only |
| `reasoner` | LangChain Judge chain (inference) | GPU host → Cloud |
| `judge` | LangChain Judge chain (evaluation) | Mac → GPU host → Cloud |

## Daily Operations

```bash
docker compose ps                             # Check status
docker compose logs -f                        # Follow logs
docker compose restart                        # Apply config changes
docker compose pull && docker compose up -d   # Update image
```

## ⚠ Caution

- `.env` contains secrets — it is excluded from Git via `.gitignore`
- Changing `config.yaml` requires `docker compose restart`
