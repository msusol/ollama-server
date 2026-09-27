# TODO

_Last synced: 2026-09-27_

---

## Qwen → Gemma 4 Migration

See [`docs/plans/2026-09-27-qwen-to-gemma4-migration.md`](2026-09-27-qwen-to-gemma4-migration.md)
for full context. Scope: `mattermost/bridge` + `accountant_agent` only —
`clp_parcel_ai` already runs Gemma via its own `clp-ollama` and is out of
scope.

- [x] Repo scaffolded (`compose.yaml`, `docs/{adr,plans,process,specs}`)
- [ ] Stand up `gemma-server` container, pull `gemma4:26b`
- [ ] Re-run `lori-model-evaluation-plan.md`-style behavioral eval against
      `gemma-server`
- [ ] Validate `accountant_agent` categorization + wiki-extraction quality
      against Gemma vs. the Qwen baseline
- [ ] Cut over `mattermost/bridge/backends.py` and
      `accountant_agent/categorize/llm.py` + `wiki/extract.py` env vars
- [ ] Re-run `mattermost/bridge/tests/` and `accountant_agent/tests/` —
      fix any Qwen-specific assertions
- [ ] Update `accountant_agent` and `mattermost` process/spec docs to
      reference `gemma-server` instead of `ollama-poc`
- [ ] Retire `ollama-poc` service + volume from `mattermost/docker-compose.yml`

## Next steps

### Qwen → Gemma 4 migration
1. Bring up `gemma-server` and pull `gemma4:26b` on `spark-db62`.
2. Re-run the behavioral eval before touching any consumer's env vars.
