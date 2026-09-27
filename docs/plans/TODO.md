# TODO

_Last synced: 2026-09-27_

---

## Qwen → Gemma 4 Migration

See [`docs/plans/2026-09-27-qwen-to-gemma4-migration.md`](2026-09-27-qwen-to-gemma4-migration.md)
for full context. Scope: `mattermost/bridge`, `accountant_agent`, **and**
`clp_parcel_ai` — broadened 2026-09-27 (ADR 0002) once live inspection found
`clp-ollama` running as a 3rd separate GPU-capable Ollama process, not actually
out of scope.

- [x] Repo scaffolded (`compose.yaml`, `docs/{adr,plans,process,specs}`)
- [x] Renamed repo/container to `ollama-server`; ADR 0002 broadens scope
      to also consolidate `clp-ollama`
- [x] `ollama-server` standing up on `spark-db62` — `gemma4:26b` reused
      from `ollama-poc`'s existing data (no re-download), real inference
      confirmed via `/api/chat` — 2026-09-27
- [x] Ran `mattermost`'s behavioral eval (`mattermost/bridge/ollama_eval/`,
      10 scenarios) against `ollama-server` — 9/10 pass consistently across
      3 runs; one real gap (`document_search_not_found` skips an
      acknowledgment when combined with `other_documents`) and one
      non-determinism note (`heloc_status_forced` flipped once). Not yet
      cut over — see below.
- [x] **`clp_parcel_ai` fully cut over** — 2026-09-27, ahead of
      `mattermost`/`accountant_agent`. 13/13 eval scenarios +
      full 203-test suite pass on both models; `compose.yaml`/`config.yaml`
      repointed to `ollama-server`/`gemma4:26b`; `clp-ollama` kept running
      for rollback. Full writeup:
      `ColoradoLandPartners/docs/plans/2026-09-27-ollama-server-cutover-eval.md`
- [x] **`accountant_agent` fully cut over** — 2026-09-27. 11/11 eval
      scenarios pass on `gemma4:26b` (10/11 on the `qwen3:14b` baseline —
      the one baseline failure, falling for a document prompt-injection
      that `gemma4:26b` resists, favors the cutover); full 547-test suite
      passes. `categorize/llm.py`/`wiki/extract.py` repointed via
      `config.get_setting()` (env var → `accountant_agent/.env` → default),
      not a bare `os.environ.get()` — found there was no override mechanism
      at all beforehand. `ollama-poc` untouched (it's `mattermost`'s own
      compose service, not `accountant_agent`'s). Full writeup:
      `ColoradoLandPartners/accountant_agent/docs/plans/2026-09-27-ollama-server-cutover-eval.md`
- [ ] Cut over `mattermost/bridge/backends.py` env vars (the last remaining
      consumer)
- [ ] Re-run `mattermost/bridge/tests/` — fix any Qwen-specific assertions
- [ ] Update `mattermost`'s and `accountant_agent`'s remaining process/spec
      docs (`daily-run.md`, `lori-system-overview.md`,
      `mattermost-stack.md`) to reference `ollama-server` instead of
      `ollama-poc`
- [ ] Retire `ollama-poc` service + volume from `mattermost/docker-compose.yml`,
      and `clp-ollama` from `ColoradoLandPartners/compose.yaml`, once all
      three consumers have run on `ollama-server` through a burn-in period

## Next steps

### Qwen → Gemma 4 migration
1. Cut over `mattermost/bridge`'s env vars — the last remaining consumer.
2. Re-run `mattermost/bridge/tests/` and fix any Qwen-specific assertions.
