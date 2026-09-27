# ollama-server docs

Shared Ollama model-serving container for the `LosusAIOps` cluster,
currently serving Gemma 4 (`gemma4:26b`). Replaced `ollama-poc`
(`qwen3:14b`) and `clp-ollama` (`gemma4:12b`), both retired 2026-09-27.

## Start here

- [ADR 0001 — Standalone shared Gemma server](adr/0001-standalone-shared-gemma-server.md) *(superseded by ADR 0002)*
- [ADR 0002 — Consolidate all project Ollama onto ollama-server](adr/0002-consolidate-all-project-ollama-onto-ollama-server.md)
- [Ollama consolidation & Qwen → Gemma 4 migration plan](plans/archive/2026-09-27-qwen-to-gemma4-migration.md) *(archived — migration complete)*
- [Stack process doc](process/ollama-server-stack.md)

## Layout

- `adr/` — architecture decisions
- `plans/` — implementation plans and `TODO.md`
- `process/` — operational how-tos
- `specs/` — (none yet)
