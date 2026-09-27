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
- [ ] Validate `accountant_agent` categorization + wiki-extraction quality
      against Gemma vs. the Qwen baseline
- [ ] Cut over `mattermost/bridge/backends.py` and
      `accountant_agent/categorize/llm.py` + `wiki/extract.py` env vars
- [ ] Re-run `mattermost/bridge/tests/` and `accountant_agent/tests/` —
      fix any Qwen-specific assertions
- [ ] Update `accountant_agent` and `mattermost` process/spec docs to
      reference `ollama-server` instead of `ollama-poc`
- [ ] Retire `ollama-poc` service + volume from `mattermost/docker-compose.yml`,
      and `clp-ollama` from `ColoradoLandPartners/compose.yaml`, once all
      three consumers have run on `ollama-server` through a burn-in period

## Next steps

### Qwen → Gemma 4 migration
1. Validate `accountant_agent`'s categorization/wiki-extraction quality against Gemma.
2. Cut over `mattermost/bridge` + `accountant_agent` env vars once validated.
