# Hindsight Local Stack (Docker Compose)

A complete, self-contained local stack: PostgreSQL with pgvector, the
Hindsight API, and the Control Plane UI. The server's LLM is Zhipu
`glm-5.3-flash` via the Coding Plan chat endpoint, reusing the API key
already stored by Codex, so no separate key setup is needed.
Embeddings and reranking also run remotely (Zhipu `embedding-3` at 1024
dimensions, plus the built-in zero-dependency RRF reranker), which keeps
the API container free of torch and local ML models.
Every provider, model, and endpoint is configurable from the `.env` file
next to the compose file - swapping providers never requires editing
`docker-compose.yaml`.

The stack runs as three services (`docker compose ps`):

- `db` — PostgreSQL 18 with pgvector
- `hindsight-api` — the FastAPI dataplane; the LLM client and the local
  embedding/reranker models live here, so this container holds most of the
  stack's memory
- `control-plane` — the Next.js admin UI (the `next-server` process),
  pointing at `hindsight-api` over the Compose network

The API and the UI ship inside one published image; each container enables
only its half via `HINDSIGHT_ENABLE_API` / `HINDSIGHT_ENABLE_CP`, which
keeps their memory footprints visible and independently limitable.

## Prerequisites

- Docker (Docker Desktop is fine; the Hindsight image is multi-arch)
- Codex CLI installed and logged in (`OPENAI_API_KEY` present in
  `~/.codex/auth.json`)

## Start / stop

```bash
docker compose -f docker/docker-compose/local/docker-compose.yaml up -d
docker compose -f docker/docker-compose/local/docker-compose.yaml down
```

- API: http://localhost:12888
- Control Plane: http://localhost:12999

Both ports bind to all interfaces (`0.0.0.0`), so LAN and Tailscale clients
can reach the stack on servers. The API and Control Plane have no built-in
auth in this configuration — restrict access with a firewall, a Tailscale
ACL, or set `HINDSIGHT_CP_ACCESS_KEY` before exposing them beyond a trusted
network. Postgres is not published to the host at all; it is reachable only
inside the Compose network.

## Model provider configuration

The API key lives in a `.env` file next to the compose file (loaded via
`env_file` and used for compose interpolation, gitignored). Copy
`.env.example` to `.env` and adjust the values. The defaults reproduce the
Zhipu setup: `glm-5.3-flash` via the Coding Plan chat endpoint and
`embedding-3` via the general OpenAI-compatible endpoint, sharing one key.

Chat uses the Coding Plan endpoint (`/api/coding/paas/v4`), where
plan-covered models (`glm-5.3-flash`, `glm-5.2`) are entitled — the
general endpoint rejects them with error 1113. Embeddings use the general
endpoint (`/api/paas/v4`) with `embedding-3`.

To switch providers, uncomment one of the ready-made blocks in `.env.example`
(DeepSeek + SiliconFlow, OpenAI, Anthropic, fully local via Ollama) or set
the variables yourself. The chat and embedding providers are fully
independent; the embedding path uses provider `openai` plus a custom
`HINDSIGHT_API_EMBEDDINGS_OPENAI_BASE_URL`, which covers any
OpenAI-compatible endpoint (Zhipu, SiliconFlow, Ollama, vLLM, LM Studio).
For example, DeepSeek chat with SiliconFlow embeddings:

```bash
HINDSIGHT_API_LLM_API_KEY=your-deepseek-api-key
HINDSIGHT_API_LLM_PROVIDER=deepseek
HINDSIGHT_API_LLM_BASE_URL=https://api.deepseek.com/v1
HINDSIGHT_API_LLM_MODEL=deepseek-chat

HINDSIGHT_API_EMBEDDINGS_OPENAI_API_KEY=your-siliconflow-api-key
HINDSIGHT_API_EMBEDDINGS_PROVIDER=openai
HINDSIGHT_API_EMBEDDINGS_OPENAI_BASE_URL=https://api.siliconflow.cn/v1
HINDSIGHT_API_EMBEDDINGS_OPENAI_MODEL=BAAI/bge-m3
HINDSIGHT_API_EMBEDDINGS_OPENAI_DIMENSIONS=1024
```

Two operational notes:

- pgvector's HNSW index caps vectors at 2000 dimensions, so keep
  `HINDSIGHT_API_EMBEDDINGS_OPENAI_DIMENSIONS` at 2000 or below. Wider
  models (e.g. Qwen3-Embedding-4B, native 2560) must be truncated via this
  setting.
- To point at a server on your host (Ollama, LM Studio, vLLM), use
  `host.docker.internal` instead of `localhost`; the compose file maps that
  name on Linux automatically, and Docker Desktop defines it natively.

## 2048-dimension embeddings

The database image ships both pgvector and pgvectorscale (DiskANN). The
default backend stays on pgvector HNSW at 1024 dimensions; to run embedding-3
at its native 2048 dimensions, set in `.env`:

```bash
HINDSIGHT_API_VECTOR_EXTENSION=pgvectorscale
HINDSIGHT_API_EMBEDDINGS_OPENAI_DIMENSIONS=2048
```

The stack pulls this prebuilt image from Docker Hub by default (public, no
login required):

```text
youbuwei/hindsight-db-pgvectorscale:pg18-pgvs0.9.1
```

Set `HINDSIGHT_DB_IMAGE` in `.env` only to pin a different tag.

Dimension changes require a volume wipe — stored vectors cannot be widened
in place. To discard all existing memory data and start fresh:

```bash
docker compose -f docker/docker-compose/local/docker-compose.yaml down -v
docker compose -f docker/docker-compose/local/docker-compose.yaml up -d
```

If you want to preserve existing memories instead, export each bank first with
`hindsight-admin export-bank`, then re-import them after the restart —
`import-bank` re-embeds everything at the new dimension.

## Wire up your agents

Codex and Claude Code (they share one memory bank per repository):

```bash
npx @vectorize-io/hindsight-coding-agents install codex \
  --server self-hosted --api-url http://localhost:12888
npx @vectorize-io/hindsight-coding-agents install claude-code \
  --server self-hosted --api-url http://localhost:12888
```

Hermes (uses its own `hermes` bank):

```bash
hermes plugins install hindsight
hermes memory setup   # choose hindsight, mode "local_external", URL http://localhost:12888
```

 If GitHub is unreachable (the catalog install clones from
 github.com), install from a local snapshot of this repo instead:

 ```bash
 cp -R hindsight-integrations/hermes /tmp/hindsight-hermes-plugin
 cd /tmp/hindsight-hermes-plugin && git init && git add -A \
   && git -c user.email=local@hindsight -c user.name=local commit -m snapshot
 hermes plugins install "file:///tmp/hindsight-hermes-plugin" --enable --yes-deps
 ```

 Then point it at this stack (writes `~/.hermes/hindsight/config.json`):

 ```json
 { "mode": "local_external", "api_url": "http://localhost:12888" }
 ```

Optionally disable Hermes' built-in markdown memory so the two do not
compete:

```bash
hermes config set memory.memory_enabled false
```

## Data and upgrades

All memory lives in the `pg_data` volume; `docker compose down` preserves
it (add `--volumes` only when you really want to wipe memory). To upgrade,
pull the latest image and recreate:

```bash
docker compose -f docker/docker-compose/local/docker-compose.yaml pull
docker compose -f docker/docker-compose/local/docker-compose.yaml up -d
```
