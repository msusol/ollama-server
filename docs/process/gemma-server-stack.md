# gemma-server stack

## Prerequisites

- Docker + NVIDIA Container Toolkit on the DGX Spark (`spark-db62`)
- Access to `spark-db62` (Tailscale or local)

## Steps

1. Create the external data volume (first run only):
   ```zsh
   docker volume create gemma_server_data
   ```
2. Bring up the service:
   ```zsh
   docker compose up -d
   ```
3. Pull the model:
   ```zsh
   docker exec gemma-server ollama pull gemma4:26b
   ```
4. Verify it's serving:
   ```zsh
   curl http://127.0.0.1:11436/api/tags
   ```

## Expected output

`/api/tags` lists `gemma4:26b`. `docker logs gemma-server` shows the model
loaded on GPU with no OOM errors.

## Troubleshooting

- **Port 11436 already in use**: check nothing else bound it —
  `ollama-poc` uses 11435, `clp-ollama` has no host port mapping (internal
  only), and a host systemd Ollama may already hold 11434.
- **GPU OOM**: `OLLAMA_MAX_LOADED_MODELS=1` is already set in
  `compose.yaml`; if OOM still occurs, check no other GPU-resident
  container (`ollama-poc`, `clp-ollama`, host systemd Ollama) is loaded at
  the same time — see
  `mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md` for
  why running all three simultaneously was previously rejected.

## Related docs

- [Standalone shared Gemma server ADR](../adr/0001-standalone-shared-gemma-server.md)
- [Qwen → Gemma 4 migration plan](../plans/2026-09-27-qwen-to-gemma4-migration.md)
