# ollama-server stack

## Prerequisites

- Docker + NVIDIA Container Toolkit on the DGX Spark (`spark-db62`)
- Access to `spark-db62` (Tailscale or local)
- **This host has two Docker CLI contexts**: `default` (the real engine
  everything runs under) and `desktop-linux` (an unused, near-empty Docker
  Desktop engine). Plain `docker` has picked `desktop-linux`
  inconsistently across shell invocations — always pass
  `--context default` or export `DOCKER_CONTEXT=default` explicitly.

## Steps

1. Create the external data volume (first run only):
   ```zsh
   docker --context default volume create ollama_server_data
   ```
2. If migrating from `ollama-poc`, copy its existing model blobs instead
   of re-pulling (saves ~18.6GB re-download for `gemma4:26b`). Host is
   arm64 (DGX Spark/Grace) — pass `--platform linux/arm64` explicitly,
   since a plain `alpine` pull can resolve to `amd64` and fail with
   `exec format error`:
   ```zsh
   docker --context default run --rm --platform linux/arm64 \
     -v ollama_poc_data:/from \
     -v ollama_server_data:/to \
     alpine \
     sh -c "cp -a /from/. /to/."
   ```
3. Bring up the service:
   ```zsh
   DOCKER_CONTEXT=default docker compose up -d
   ```
4. If not migrating from an existing volume, pull the model fresh:
   ```zsh
   docker --context default exec ollama-server ollama pull gemma4:26b
   ```
5. Verify it's serving:
   ```zsh
   curl http://127.0.0.1:11436/api/tags
   ```
6. Confirm real inference works (not just that the model is listed):
   ```zsh
   curl -s http://127.0.0.1:11436/api/chat -d '{
     "model": "gemma4:26b",
     "messages": [{"role": "user", "content": "Reply with exactly one word: OK"}],
     "stream": false
   }'
   ```

## Expected output

`/api/tags` lists `gemma4:26b`. `docker logs ollama-server` shows the GPU
detected (`NVIDIA GB10` on this host) with no OOM errors. The first
`/api/chat` request after a fresh container start takes significantly
longer (~60s observed) due to cold model load onto the GPU; subsequent
requests are much faster while the model stays resident.

## Troubleshooting

- **Wrong/empty containers and volumes shown by `docker ps`/`docker volume
  ls`**: check `docker context ls` — if it shows `desktop-linux` as
  current instead of `default`, every command is hitting the wrong,
  effectively-empty engine. Use `--context default` explicitly.
- **`exec format error` running a throwaway container (e.g. `alpine`) for
  a volume copy**: the image resolved to the wrong CPU architecture. Pass
  `--platform linux/arm64` on this host.
- **Port 11436 already in use**: check nothing else bound it —
  `ollama-poc` uses 11435, `clp-ollama` has no host port mapping (internal
  only), and a host systemd `ollama.service` (not a container) may already
  hold 11434.
- **GPU OOM**: `OLLAMA_MAX_LOADED_MODELS=1` is already set in
  `compose.yaml`; if OOM still occurs, check no other GPU-resident
  container (`ollama-poc`, `clp-ollama`) is loaded at the same time — see
  `mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md` for
  why running multiple simultaneously was previously rejected. This host's
  GPU (`NVIDIA GB10`) has 121.7 GiB unified VRAM, so this is less pressing
  here than on a discrete-GPU host, but still the reason for consolidating
  onto one server per ADR 0002.

## Related docs

- [ADR 0001 — Standalone shared Gemma server](../adr/0001-standalone-shared-gemma-server.md) *(superseded)*
- [ADR 0002 — Consolidate all project Ollama onto ollama-server](../adr/0002-consolidate-all-project-ollama-onto-ollama-server.md)
- [Ollama consolidation & Qwen → Gemma 4 migration plan](../plans/2026-09-27-qwen-to-gemma4-migration.md)
