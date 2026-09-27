# CLAUDE.md — gemma-server

Shared Ollama/Gemma 4 model-serving container for the `LosusAIOps`
cluster (`mattermost/bridge`, `accountant_agent`). Replaces `ollama-poc`
(`qwen3:14b`).

See `docs/index.md` for the full doc set — start with
`docs/adr/0001-standalone-shared-gemma-server.md` and
`docs/plans/2026-09-27-qwen-to-gemma4-migration.md`.

## Quick reference

```zsh
docker compose up -d
docker exec gemma-server ollama pull gemma4:26b
curl http://127.0.0.1:11436/api/tags
```

Out of scope for this repo: `clp_parcel_ai`'s `clp-ollama` (already
Gemma-based, separately owned in `ColoradoLandPartners`).
