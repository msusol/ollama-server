# 0002. Consolidate all project Ollama traffic onto one NVIDIA-Docker `ollama-server`

## Status

Accepted

## Context

ADR 0001 decided to stand up a shared replacement for `ollama-poc`
(originally named `gemma-server`), but explicitly left `clp-ollama`
(`clp_parcel_ai`'s own Ollama container, `gemma4:12b`) out of scope as an
"open item."

Live inspection of `spark-db62` on 2026-09-27 found the actual running
state is more fragmented than the docs described:

| Instance | How it runs | Reachable from host? | Models pulled |
|---|---|---|---|
| `ollama-poc` | Docker container (`mattermost/docker-compose.yml`) | `127.0.0.1:11435` | `qwen3:14b`, **and already `gemma4:26b`** (pulled 2026-09-22 — Phase 1's "pull Gemma alongside Qwen" step is already done) |
| `clp-ollama` | Docker container (`ColoradoLandPartners/compose.yaml`) | Not published — internal Docker network only | `gemma4:12b` |
| host `ollama.service` | Native systemd process, **not a Docker container** | `127.0.0.1:11434` | `gemma4:26b` |

The host systemd `ollama.service` is not consumed by any project code —
it exists for interactive `ollama` CLI use on the host directly. It does
not run under the NVIDIA Docker runtime the way the containerized
instances do, and this decision does not touch it.

Running three separate GPU-capable Ollama processes (two containers plus
the host service) on one GPU is the same class of resource fragmentation
`mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md`
already flagged as a risk for a single POC container — it's worse with
three.

Separately: this repo was originally named `gemma-server`, after the
model it serves. Renamed to `ollama-server` in the same round of changes
that produced this ADR — the repo/container should be named for the
runtime (Ollama), which is stable, not the model, which will change again
(the repo already migrated from serving Qwen to serving Gemma once; naming
it after any one model just guarantees another rename next time).

## Decision

`ollama-server` (this repo, container `ollama-server`) is the **single,
NVIDIA-Docker-backed Ollama container serving every project**:
`clp_parcel_ai`, `accountant_agent`, `mattermost/bridge`, and future
`investment_agent`/`analyst_agent`. It can hold and serve whichever model a
caller requests (`gemma4:26b` for chat/tool-calling consumers;
`gemma4:12b` stays available if `clp_parcel_ai`'s pipeline still needs
that specific size after evaluation).

This **broadens ADR 0001's scope**: `ollama-server` now replaces both
`ollama-poc` **and** `clp-ollama`, not just `ollama-poc`.

The host systemd `ollama.service` (native, non-Docker, port 11434) is
explicitly **out of scope** — it's Mark's own CLI tool, not a project
dependency, and is left running untouched.

## Consequences

- One GPU-resident Ollama process for all project traffic instead of
  three, removing the resource-fragmentation risk called out above.
- `clp_parcel_ai`'s `compose.yaml`/`config.py` `OLLAMA_HOST` must be
  repointed from `clp-ollama` to `ollama-server`, in addition to the
  `mattermost`/`accountant_agent` repointing already planned in ADR 0001.
- `clp_parcel_ai`'s mocked tests (`test_lead_response_analysis.py`, and
  any others patching `ollama.chat`) need the same "won't break
  mechanically, but check behavior" treatment as `mattermost`/
  `accountant_agent`'s tests — see the updated migration plan.
- A model-size behavior check is now needed for `clp_parcel_ai` too:
  `gemma4:12b` (current) vs `gemma4:26b` (target), not just the
  Qwen-vs-Gemma comparison already planned for the chat/categorization
  consumers.
- `clp-ollama` is retired (container + volume removed) once
  `clp_parcel_ai`'s cutover is verified, same as `ollama-poc`.
- Host systemd `ollama.service` stays as-is — no action, no retirement,
  not part of this consolidation.
- Naming the repo/container after the runtime, not the model, means a
  future model change (e.g. to a non-Gemma model) doesn't force another
  repo rename.

## Alternatives considered

- **Leave `clp-ollama` separate, as ADR 0001 originally proposed.**
  Rejected — the live-state inspection that prompted this ADR showed the
  fragmentation cost is real and immediate (three simultaneous GPU-capable
  processes today), not a hypothetical future problem worth deferring.
- **Keep the `gemma-server` name.** Rejected — ties the repo/container
  identity to a model choice that has already changed once (Qwen →
  Gemma) and will change again; `ollama-server` names the stable part
  (the runtime), matching the project's naming pattern of naming
  infrastructure after what it is, not what it currently runs
  (`clp-ollama`, `ollama-poc` are already runtime-named, not model-named).

## Related decisions

- Supersedes: `0001-standalone-shared-gemma-server.md`
- Related: `mattermost/docs/adr/0001-run-poc-ollama-on-nvidia-docker-engine.md`
