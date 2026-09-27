# 0001. Run a standalone, shared Gemma 4 Ollama server

## Status

Accepted

## Context

Prior to this decision, local LLM inference across the Losus AI / WLP
projects was split across three separately-managed Ollama instances:

- `ollama-poc` (`mattermost/docker-compose.yml`, host port 11435) —
  `qwen3:14b`, used by `mattermost/bridge/backends.py` (Lori's chat
  backend) and by `accountant_agent/categorize/llm.py` +
  `accountant_agent/wiki/extract.py`.
- `clp-ollama` (`ColoradoLandPartners/compose.yaml`, internal-only) —
  already `gemma4:12b`, used by `clp_parcel_ai`'s owner-name parsing,
  lead-response classification, and pipeline-email summarization.
- A host systemd Ollama instance (loopback 11434, `gemma4:26b`) — stood up
  for evaluation (`mattermost/docs/plans/lori-model-evaluation-plan.md`)
  but not wired into any running service.

`mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md`
recorded the original decision to stand up `ollama-poc` as a one-off POC
service. `lori-model-evaluation-plan.md` later found that a larger Gemma
model (`gemma4:26b`) behaves more literally/conservatively than
`qwen3:14b` on tool-use prompts — a real behavioral difference to
validate, not just a drop-in swap.

Now that `LosusAIOps/` is being split into standalone repos
(`accountant-agent/`, `mattermost/`, `gemma-server/`), continuing to run
one Ollama container per consumer duplicates GPU memory and config across
repos that all run on the same physical host (`spark-db62`).

## Decision

We will run one shared Ollama service, `gemma-server`, serving
`gemma4:26b`, as its own repo/container under `LosusAIOps/`. It replaces
`ollama-poc` for `mattermost/bridge` and `accountant_agent`'s
categorization/wiki-extraction, and is reachable by any other project on
the same Docker network / Tailscale (`clp_parcel_ai`'s existing
`clp-ollama` stays separate for now — see "Consequences").

`OLLAMA_MAX_LOADED_MODELS=1` is set explicitly so the GPU only ever holds
one model resident, matching the single-model-at-a-time usage pattern of
every current consumer.

## Consequences

- One GPU-resident model to manage instead of three overlapping Ollama
  containers.
- `mattermost/bridge` and `accountant_agent` must be repointed
  (`OLLAMA_URL`/`OLLAMA_MODEL` env vars) from `ollama-poc` to
  `gemma-server`, and their mocked test suites' assertions re-checked for
  Qwen-specific response shapes — see
  `docs/plans/2026-09-27-qwen-to-gemma4-migration.md`.
- Behavioral regressions are possible: `lori-model-evaluation-plan.md`
  already documented Gemma being more literal/conservative than Qwen on
  tool-calling prompts. The migration plan re-runs that style of
  evaluation before cutover, not just the unit test suites.
- `clp_parcel_ai`'s `clp-ollama` (`gemma4:12b`) is **not** merged into this
  server in this ADR — it's a separate, smaller model tuned for that
  pipeline's needs, and consolidating it is an open question, not a
  decision made here (see Open items).
- `ollama-poc` is retired once cutover is verified; its GPU/volume
  footprint is freed.

## Alternatives considered

- **Keep `ollama-poc`, just repull a Gemma tag into it.** Rejected —
  `ollama-poc` lives inside the `mattermost` repo's compose file, which is
  being split out of `ColoradoLandPartners` anyway; a shared cross-repo
  service belongs in its own repo, not owned by one consumer.
- **Merge everything into `clp-ollama`.** Rejected for now — `clp-ollama`
  is scoped to `clp_parcel_ai`'s pipeline and already tuned/sized
  (`gemma4:12b`) for that workload; consolidating a second, larger model
  onto it wasn't evaluated and isn't required to unblock the
  `ollama-poc` retirement.

## Open items

- Whether `clp-ollama` should eventually be retired in favor of this
  server too is left for a future ADR once `gemma4:26b` vs `gemma4:12b`
  tradeoffs for the ETL pipeline's workload are evaluated.
