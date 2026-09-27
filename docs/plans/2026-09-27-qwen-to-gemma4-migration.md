# Qwen → Gemma 4 Migration Implementation Plan

## Goal

Stand up `gemma-server` (Ollama, `gemma4:26b`) and cut `mattermost/bridge`
and `accountant_agent` over to it from `ollama-poc` (`qwen3:14b`), with the
existing mocked test suites and a behavioral eval both passing before
`ollama-poc` is retired.

## Context

Per `docs/adr/0001-standalone-shared-gemma-server.md`, `ollama-poc` (port
11435, `qwen3:14b`) is consumed by exactly three call sites, all outside
`clp_parcel_ai` (which already runs its own Gemma model, `clp-ollama`, and
is **out of scope** for this migration):

| Call site | File | Env vars (defaults) |
|---|---|---|
| Lori chat backend, tool-calling | `mattermost/bridge/backends.py` | `OLLAMA_URL=http://ollama-poc:11434`, `OLLAMA_MODEL=qwen3:14b` |
| Transaction categorization | `accountant_agent/categorize/llm.py` | `OLLAMA_URL=http://127.0.0.1:11435`, `OLLAMA_MODEL=qwen3:14b` |
| Wiki document extraction | `accountant_agent/wiki/extract.py` | `OLLAMA_URL=http://127.0.0.1:11435`, `OLLAMA_MODEL=qwen3:14b` |

Existing test coverage for these mocks the HTTP/`ollama.chat` boundary, so
it won't break mechanically on a model swap — but assertions may encode
Qwen-specific response shapes:

- `accountant_agent/tests/fake_ollama.py` + `test_categorize_llm.py`
- `mattermost/bridge/tests/test_doc_requests.py` (`scripted_ollama()`)
- `mattermost/bridge/tests/test_schedule_e_form.py` (`fake_ollama()`)

`mattermost/docs/plans/lori-model-evaluation-plan.md` already ran a
qwen3:14b-vs-Gemma-26B behavioral comparison and found Gemma more literal/
conservative on tool-use prompts (it refused a question Qwen answered) —
that's the real regression risk here, not the mocked unit tests.

## Tasks

### Stand up gemma-server

- [ ] Bring up `compose.yaml` in this repo (`docker compose up -d`) on
      `spark-db62`
- [ ] Create the external `gemma_server_data` volume before first run
      (`docker volume create gemma_server_data`)
- [ ] `docker exec gemma-server ollama pull gemma4:26b`
- [ ] Smoke-test: `curl http://127.0.0.1:11436/api/tags` shows the model

### Validate before cutover (behavioral, not just unit tests)

- [ ] Re-run `mattermost/docs/plans/lori-model-evaluation-plan.md`'s
      evaluation prompts against `gemma-server`'s `gemma4:26b`, not just
      the old host-systemd Ollama instance it was originally tested on
- [ ] Confirm the 3 system prompts in `backends.py` still produce correct
      `tool_calls` output shape against Gemma (per
      `mattermost/docs/adr/0004-tool-calling-with-bridge-enforced-entity-scope.md`)
- [ ] Run `accountant_agent`'s categorization eval set (if one exists;
      otherwise spot-check a sample of real transactions) against Gemma
      and compare category accuracy to the Qwen baseline
- [ ] Run `accountant_agent`'s wiki-extraction against a sample of real
      documents and compare extraction quality to the Qwen baseline

### Cut over

- [ ] Update `mattermost/bridge/backends.py` defaults (or its `.env`) —
      `OLLAMA_URL` → `gemma-server`'s address, `OLLAMA_MODEL=gemma4:26b`
- [ ] Update `accountant_agent/categorize/llm.py` and
      `accountant_agent/wiki/extract.py` defaults (or shared `.env`) the
      same way
- [ ] Re-run `mattermost/bridge/tests/` and `accountant_agent/tests/` full
      suites — confirm all pass; update any assertion that encoded
      Qwen-specific response text/formatting
- [ ] Update `accountant_agent/docs/process/categorization.md`,
      `document-wiki.md`, `daily-run.md` and
      `accountant_agent/docs/specs/lori-system-overview.md` — replace
      `ollama-poc`/qwen3:14b references with `gemma-server`/gemma4:26b
- [ ] Update `mattermost/docs/process/mattermost-stack.md` the same way

### Retire ollama-poc

- [ ] Confirm `gemma-server` has run cutover in production for a burn-in
      period with no regressions reported
- [ ] Remove the `ollama-poc` service from `mattermost/docker-compose.yml`
      and its `ollama_poc_data` volume
- [ ] Mark `mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md`
      status as `Superseded by gemma-server/docs/adr/0001` (cross-repo
      cross-reference — note this explicitly in that ADR's status line
      since it lives in a different repo)

## Notes

This plan intentionally excludes `clp_parcel_ai`/`clp-ollama` — it has no
Qwen dependency to migrate. Whether to later consolidate `clp-ollama` onto
`gemma-server` is an open item tracked in
`docs/adr/0001-standalone-shared-gemma-server.md`, not part of this plan.
