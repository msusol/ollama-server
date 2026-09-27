# Ollama Consolidation & Qwen → Gemma 4 Migration Implementation Plan

## Goal

Stand up `ollama-server` as the single NVIDIA-Docker-backed Ollama
container for all projects, and cut `mattermost/bridge`,
`accountant_agent`, **and `clp_parcel_ai`** over to it from `ollama-poc`
(`qwen3:14b`) and `clp-ollama` (`gemma4:12b`) respectively, with existing
test suites and a behavioral eval both passing before either old
container is retired.

## Context

Per `docs/adr/0002-consolidate-all-project-ollama-onto-ollama-server.md`
(which supersedes `docs/adr/0001`), this now covers **four** call sites
across two source containers:

| Call site | File | Source container | Env vars (defaults) |
|---|---|---|---|
| Lori chat backend, tool-calling | `mattermost/bridge/backends.py` | `ollama-poc` | `OLLAMA_URL=http://ollama-poc:11434`, `OLLAMA_MODEL=qwen3:14b` |
| Transaction categorization | `accountant_agent/categorize/llm.py` | `ollama-poc` | `OLLAMA_URL=http://127.0.0.1:11435`, `OLLAMA_MODEL=qwen3:14b` |
| Wiki document extraction | `accountant_agent/wiki/extract.py` | `ollama-poc` | `OLLAMA_URL=http://127.0.0.1:11435`, `OLLAMA_MODEL=qwen3:14b` |
| Owner-name parsing, lead-response classification, email digest | `clp_parcel_ai/etl/lib/{ownership_parser,lead_response_analysis,ollama_summarizer}.py` | `clp-ollama` | `OLLAMA_HOST=http://ollama:11434` (compose-internal), `config.yaml → ollama.model: gemma4:12b` |

**Live-state correction (verified on `spark-db62`, 2026-09-27):** the
"pull Gemma 4 alongside Qwen" step is **already done** — `ollama-poc`
already has `gemma4:26b` pulled (since 2026-09-22) next to `qwen3:14b`.
There is no need to re-download that ~18.6GB model into `ollama-server`;
copy the existing blob from the `ollama_poc_data` volume instead (see
Tasks).

There is also a fourth, unrelated Ollama process on this host — a native
systemd `ollama.service` on `127.0.0.1:11434` — which is **not a Docker
container**, is **not consumed by any project**, and is **out of scope**
for this migration entirely (it's Mark's own CLI tool).

Existing test coverage mocks the HTTP/`ollama.chat` boundary, so it won't
break mechanically on a model swap — but assertions may encode
model-specific response shapes:

- `accountant_agent/tests/fake_ollama.py` + `test_categorize_llm.py`
- `mattermost/bridge/tests/test_doc_requests.py` (`scripted_ollama()`)
- `mattermost/bridge/tests/test_schedule_e_form.py` (`fake_ollama()`)
- `clp_parcel_ai/etl/tests/test_lead_response_analysis.py` (patches
  `ollama.chat` directly)

`mattermost/docs/plans/lori-model-evaluation-plan.md` already ran a
qwen3:14b-vs-Gemma-26B behavioral comparison and found Gemma more literal/
conservative on tool-use prompts (it refused a question Qwen answered) —
that's the real regression risk for the chat/categorization consumers.
`clp_parcel_ai` has a separate, new risk: `gemma4:12b` → `gemma4:26b` is a
model-size change, not just a family change, so its parsing/classification
accuracy needs its own spot-check, not just a code-level test pass.

## Tasks

### Stand up ollama-server (reuse existing model data, don't re-pull)

- [x] Create the external `ollama_server_data` volume — done 2026-09-27.
      Note: this host has two Docker CLI contexts (`default`, the real
      engine everything runs under, and `desktop-linux`, an unused/empty
      Docker Desktop engine); commands must target `--context default`
      explicitly or `DOCKER_CONTEXT=default`, since plain `docker` picked
      `desktop-linux` inconsistently across shell invocations
- [x] Copy the `gemma4:26b` blobs from `ollama_poc_data` into
      `ollama_server_data` via a throwaway container — done 2026-09-27
      (26GB copied, includes `qwen3:14b` too since `ollama` dedupes by
      digest; harmless, can be pruned later since only `gemma4:26b` is
      needed going forward). Host is arm64 (DGX Spark/Grace) — use
      `--platform linux/arm64` explicitly or the default `alpine:latest`
      pull may resolve to `amd64` and fail with `exec format error`:
      ```zsh
      docker --context default run --rm --platform linux/arm64 \
        -v ollama_poc_data:/from \
        -v ollama_server_data:/to \
        alpine \
        sh -c "cp -a /from/. /to/."
      ```
- [x] Bring up `compose.yaml` in this repo (`docker compose up -d`) on
      `spark-db62` — done 2026-09-27, container `ollama-server` up on
      `127.0.0.1:11436`
- [x] Smoke-test: `curl http://127.0.0.1:11436/api/tags` shows `gemma4:26b`
      without a fresh download — confirmed 2026-09-27
- [x] Real inference smoke-test via `/api/chat` — confirmed 2026-09-27;
      GPU detected (`NVIDIA GB10`, 121.7 GiB unified VRAM); first request
      took ~66s (cold model load), subsequent requests should be much
      faster while the model stays resident
- [ ] If `clp_parcel_ai`'s evaluation (below) shows `gemma4:12b` is still
      needed for that pipeline's accuracy/latency tradeoff, pull it too:
      `docker exec ollama-server ollama pull gemma4:12b`

### Validate before cutover (behavioral, not just unit tests)

- [ ] Re-run `mattermost/docs/plans/lori-model-evaluation-plan.md`'s
      evaluation prompts against `ollama-server`'s `gemma4:26b`
- [ ] Confirm the 3 system prompts in `backends.py` still produce correct
      `tool_calls` output shape against Gemma (per
      `mattermost/docs/adr/0004-tool-calling-with-bridge-enforced-entity-scope.md`)
- [ ] Run `accountant_agent`'s categorization eval set (if one exists;
      otherwise spot-check a sample of real transactions) against Gemma
      vs. the Qwen baseline
- [ ] Run `accountant_agent`'s wiki-extraction against a sample of real
      documents vs. the Qwen baseline
- [x] `clp_parcel_ai`'s owner-name parsing and lead-response classification —
      **done 2026-09-27**, via a reusable eval harness
      (`clp_parcel_ai/etl/ollama_eval/`), not just a spot-check. 13/13 scenarios
      pass on both `gemma4:12b` and `gemma4:26b` after fixing two eval-harness
      bugs (a name-fragment check compared against a dict's `str()` repr; a
      summarizer scenario's synthetic input didn't match the real caller's
      format) and one genuine prompt gap (`intent_unclear` misclassifying an
      emoji-only reply). Full writeup:
      `ColoradoLandPartners/docs/plans/2026-09-27-ollama-server-cutover-eval.md`

### Cut over

- [ ] Update `mattermost/bridge/backends.py` defaults (or its `.env`) —
      `OLLAMA_URL` → `ollama-server`'s address, `OLLAMA_MODEL=gemma4:26b`
- [ ] Update `accountant_agent/categorize/llm.py` and
      `accountant_agent/wiki/extract.py` defaults (or shared `.env`) the
      same way
- [x] Update `clp_parcel_ai/compose.yaml`'s `app` service — `OLLAMA_HOST` →
      `ollama-server`'s address, via a new `ollama-server_default` external
      network (the `ollama`/`clp-ollama` service is kept, undeleted, for
      rollback — see "Retire" below) — done 2026-09-27
- [x] Update `clp_parcel_ai/config.yaml` → `ollama.model: gemma4:26b` — done
      2026-09-27
- [x] Re-ran the full `clp_parcel_ai/etl/tests/` suite (203 tests, not just
      `test_lead_response_analysis.py`) — all pass, confirmed inside the
      recreated `clp-app` container with the real cutover config (no
      override flags). Also added `test_ownership_parser_llm.py`, mocked
      unit test coverage for `parse_ownership()` that didn't exist before
      (`accountant_agent`/`mattermost` re-run still pending — see above)
- [ ] Update `accountant_agent/docs/process/categorization.md`,
      `document-wiki.md`, `daily-run.md`,
      `accountant_agent/docs/specs/lori-system-overview.md`,
      `mattermost/docs/process/mattermost-stack.md`, and `clp_parcel_ai`'s
      own docs referencing `clp-ollama` — replace with `ollama-server`

### Retire ollama-poc and clp-ollama

- [ ] Confirm `ollama-server` has run cutover in production for a burn-in
      period with no regressions reported, for **both** consumer groups
- [ ] Remove the `ollama-poc` service from `mattermost/docker-compose.yml`
      and its `ollama_poc_data` volume
- [ ] Remove the `ollama`/`clp-ollama` service from
      `ColoradoLandPartners/compose.yaml` and its volume
      (`coloradolandpartners_ollama_data`)
- [ ] Mark `mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md`
      status as `Superseded by ollama-server/docs/adr/0002` (cross-repo
      cross-reference — note this explicitly in that ADR's status line
      since it lives in a different repo)

## Notes

The host systemd `ollama.service` (native, port 11434, not a container) is
explicitly out of scope for this entire plan — no task here touches it.
