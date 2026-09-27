# CLAUDE.md — ollama-server

Shared Ollama model-serving container for the `LosusAIOps` cluster
(`mattermost/bridge`, `accountant_agent`, and `clp_parcel_ai`), currently
serving Gemma 4 (`gemma4:26b`). Named for the runtime (Ollama), not the
model, since a future model swap shouldn't require another repo rename.
Replaces `ollama-poc` (`qwen3:14b`) and `clp-ollama` (`gemma4:12b`).

See `docs/index.md` for the full doc set — start with
`docs/adr/0002-consolidate-all-project-ollama-onto-ollama-server.md` and
`docs/plans/2026-09-27-qwen-to-gemma4-migration.md`.

## Quick reference

```zsh
docker compose up -d
docker exec ollama-server ollama pull gemma4:26b
curl http://127.0.0.1:11436/api/tags
```

Out of scope for this repo: the host's native systemd `ollama.service`
(port 11434, not a Docker container) — that's Mark's own CLI tool, not a
project dependency.
