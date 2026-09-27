# ollama-server

A minimal, GPU-shared Ollama server for a DGX Spark (or any single-GPU
NVIDIA-Docker host), built to be the **one** local LLM backend for multiple
projects/services instead of each one running its own Ollama container.

## Why

On a single-GPU box, running one Ollama container per project fragments GPU
memory and duplicates model downloads. This repo runs exactly one
NVIDIA-Docker-backed Ollama container that every consumer on the host talks
to over the Docker network (or `127.0.0.1` for host-side clients) — the
runtime is shared, not the model choice: swapping models later doesn't
require renaming the repo or the container.

See [`docs/adr/0002-consolidate-all-project-ollama-onto-ollama-server.md`](docs/adr/0002-consolidate-all-project-ollama-onto-ollama-server.md)
for the full reasoning, including a real live-inspection finding: three
separate Ollama processes were already running on one host before this
consolidation, each holding its own GPU-resident model.

## Quick start

```zsh
# One-time: external volume for pulled models to persist across recreates
docker volume create ollama_server_data

docker compose up -d
docker exec ollama-server ollama pull <model>
curl http://127.0.0.1:11436/api/tags
```

Real inference:

```zsh
curl http://127.0.0.1:11436/api/chat -d '{
  "model": "<model>",
  "messages": [{"role": "user", "content": "hello"}],
  "stream": false
}'
```

## Connecting another project's container to this one

If the consumer is itself a Docker Compose service, declare this stack's
network as external and point the consumer's `OLLAMA_HOST`/`OLLAMA_URL` at
the container name:

```yaml
services:
  your-app:
    networks:
      - default
      - ollama-server_default
    environment:
      - OLLAMA_HOST=http://ollama-server:11434

networks:
  ollama-server_default:
    external: true
```

A host-side (non-containerized) client just uses the published port
(`http://127.0.0.1:11436` by default — see `compose.yaml`).

## DGX Spark / arm64 gotchas found while building this

- **Two Docker CLI contexts on one host is a real trap.** If `docker context
  ls` shows more than one context, plain `docker` commands can silently
  resolve to the wrong (possibly empty) one across different shell
  invocations. Pin `--context <name>` or `DOCKER_CONTEXT=<name>` explicitly
  rather than assuming the default.
- **Pull `--platform linux/arm64` explicitly for throwaway utility images**
  (e.g. `alpine` for a volume-copy step). An `amd64` image can get resolved
  and fail with `exec format error` on Grace/arm64 hardware if the platform
  isn't pinned.
- **`OLLAMA_MAX_LOADED_MODELS=1`** keeps only one model resident on the GPU
  at a time — the right default for a single-GPU shared server serving
  several different consumers' models.

## Migrating an existing model between Ollama instances without re-pulling

If you're consolidating an existing Ollama container's data into this one,
copy the volume directly instead of re-pulling (can save tens of GB):

```zsh
docker run --rm --platform linux/arm64 \
  -v <old_volume>:/from \
  -v ollama_server_data:/to \
  alpine \
  sh -c "cp -a /from/. /to/."
```

## Layout

- `compose.yaml` — the service definition
- `docs/adr/` — architecture decisions
- `docs/plans/` — implementation plans (`archive/` holds completed ones)
- `docs/process/` — operational how-tos (see
  [`docs/process/ollama-server-stack.md`](docs/process/ollama-server-stack.md)
  for the full bring-up/troubleshooting guide)

## License

No license file yet — treat as "all rights reserved" until one is added.
