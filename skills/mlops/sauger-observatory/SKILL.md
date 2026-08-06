---
name: sauger-observatory
description: sauger is the live observability app for orzo/mesabi/pi5 dataset + training work; vera is its primary orchestrator.
---

# sauger observatory — how vera drives and observes edge-ML jobs

`sauger` is a standalone Next.js observability app (self-hosted on the dev
laptop, reachable over Tailscale from `heater`/`pi5`) that ingests structured
events from `orzo`, `mesabi`, `pi5`, and `vera` itself into one SQLite store
and renders live dataset / training / eval / benchmark / hardware / export
views. **vera is sauger's primary orchestrator**: when a user (or the sauger
UI) asks for a dataset-gen / eval / benchmark / training job, vera runs the
project scripts via its skills + terminal, and the resulting events flow
into sauger automatically.

Full background: `sauger/README.md` and the plan file
`~/.cursor/plans/edge_training_observatory_3113049b.plan.md`.

## Endpoints vera needs to know

- **Ingest** (emitters use this, not vera directly):
  `POST http://localhost:3000/api/events` with header
  `Authorization: Bearer $SAUGER_RUN_TOKEN`. Body is one event object or a
  JSON array of events. Zod-validated; bad shape = 400, bad token = 401.
- **Dual trigger** (UI → job):
  `POST http://localhost:3000/api/jobs` with body
  `{ "job": "dataset_gen"|"eval"|"benchmark"|"train", "project": "orzo"|"mesabi", "mode": "direct"|"vera" }`.
  `direct` = sauger shells out to the project script. `vera` = sauger POSTs a
  prompt to vera's Runs API (`POST /v1/runs`, model `vera`, auth
  `VERA_API_KEY`). Both modes produce the same sauger events.
- **Vera session proxy** (automatic, not vera's concern):
  `POST http://localhost:3000/api/vera/runs` opens a vera run and relays its
  SSE back to the UI while persisting `vera_*` events to SQLite. Vera does
  not call this; sauger calls vera.

## Event envelope (the contract)

Every event is a single JSON object with this shape:

```json
{
  "ts": 1722953023.45,
  "project": "orzo",
  "node": "heater",
  "stage": "dataset_gen",
  "event_type": "row_written",
  "run_id": "orzo-20260806-001",
  "payload": {}
}
```

- `project`: `"orzo"` | `"mesabi"` | `"system"`
- `node`: `"laptop"` | `"heater"` | `"pi5"` | `"unknown"`
- `stage`: `"dataset_gen"` | `"chatml"` | `"split"` | `"contamination"` |
  `"train"` | `"eval"` | `"benchmark"` | `"export"` | `"hw"` | `"vera"` |
  `"alert"`
- `run_id`: `<project>-<YYYYMMDD>-<NN>` (e.g. `orzo-20260806-001`).

## Known event types + payloads

Dataset / split / contamination:

- `dataset_line` / `row_written`: `{ task, spec_id, row: {...full JSONL object...} }`
- `validation_fail`: `{ spec_id, reason, attempt }`
- `split`: `{ split, n_rows, by_task: {...} }`
- `contamination`: `{ spec_id, jaccard, vs: "test_set" }`
- `chatml_converted` (mesabi): `{ spec_id, source_row, chatml_row }`

Training / eval / benchmark / export:

- `step`: `{ step, epoch, loss, grad_norm, lr, samples_s, eta_min }`
- `checkpoint`: `{ step, path, size_mb, is_best }`
- `eval_checkpoint`: `{ step, slice, pass_rate, n, p50_s, p95_s }`
- `eval_item` (mesabi): `{ run_id, slice, id, question, answer, citations, passed, latency_s, verdict_failures }`
- `eval_item` (orzo): `{ run_id, task, spec_id, passed, json_ok, ast_ok, smoke_ok, errors }`
- `eval_summary`: `{ run_id, slice, pass_rate, p50_s, p95_s, n }`
- `benchmark`: `{ benchmark_id, base_run, ft_run, slice, base_pass, ft_pass, delta, base_p50, ft_p50 }`
- `export`: `{ stage: "merge"|"quantize"|"register", gguf_size_mb, modelfile, ollama_model }`

Hardware / alerts:

- `hw_sample` (Jetson): `{ gpu_util, gpu_mem_used_mb, gpu_mem_total_mb, power_w, temp_c, swap_mb }`
- `hw_sample` (Pi 5): `{ cpu_temp_c, hailo_temp_c, hailo_util, mem_mb }`
- `alert`: `{ kind: "thermal"|"budget"|"oom"|"failed"|"regression", severity, msg }`

Vera's own events (proxied from hermes `/v1/runs` SSE by sauger — vera does
NOT emit these itself):

- `vera_session`: `{ vera_run_id, prompt, status, started_at, ended_at, cost_usd, n_turns }`
- `vera_tool_call`: `{ vera_run_id, tool, args, status, duration_s }`
- `vera_delegation`: `{ vera_run_id, child, task, status }`
- `vera_message`: `{ vera_run_id, role, content }`

## How vera reports job progress to sauger

**Just run the project scripts.** The project scripts (`gen_dataset.py`,
`split_dataset.py`, `train_qlora.py`, `eval/run_eval.py`, `export_gguf.sh`,
`benchmark.py`, `tegrastats_emit.sh`) already call `sauger-emit` (gated by
`ORZO_SAUGER=1` / `MESABI_SAUGER=1`) and push `dataset_gen` / `eval` /
`benchmark` / `train` / `hw_sample` / `export` events themselves. Vera does
not need to wrap every step in a tool call that posts to `/api/events`.

Vera's own reasoning, tool calls, and subagent delegations are captured
**automatically** by sauger's `/api/vera/runs` proxy, which subscribes to
hermes `/v1/runs/{id}/events` SSE and translates hermes tool/progress events
into `vera_*` sauger events. So when vera runs a job:

1. sauger opens a vera run via `/api/vera/runs` (or vera is invoked directly
   by the user) → `vera_session` + `vera_message` + `vera_tool_call` events
   flow into sauger.
2. vera's terminal calls invoke the project scripts → those scripts emit
   their own `dataset_gen`/`eval`/`benchmark`/`train`/`hw_sample` events
   via `sauger-emit`.
3. Both streams share the same `run_id` if vera sets `SAUGER_RUN_ID` in the
   child env (recommended: `SAUGER_RUN_ID=<project>-<YYYYMMDD>-<NN>`).

## run_id convention

`<project>-<YYYYMMDD>-<NN>` — e.g. `orzo-20260806-001`. One run_id per
end-to-end job (dataset gen → split → train → eval → export). Vera should
mint the run_id at job start and export it as `SAUGER_RUN_ID` to every
child process so the project emitters tag their events onto the same run.

## Watchdog + cron

A shell-only watchdog polls sauger every 5 minutes and alerts on down API
or stuck runs (no new event for >15 min while `status=active`):

- Script: `~/.hermes/scripts/sauger_watchdog.sh` (symlinked from
  `~/.hermes/profiles/vera/scripts/sauger_watchdog.sh`).
- Register the cron (run once; do NOT run from inside the script):

  ```bash
  hermes -p vera cron create "every 5m" \
    --name "sauger watchdog" \
    --script ~/.hermes/scripts/sauger_watchdog.sh \
    --no-agent --deliver local
  ```

- The existing `jig projects` cron (`079ddc94dca1`) was erroring because it
  looked for `jig_project_watchdog.sh` under `profiles/vera/scripts/` while
  the script lived under `~/.hermes/scripts/`. That is now fixed via
  symlinks under `~/.hermes/profiles/vera/scripts/` (jig / orzo_gen /
  orzo_hs / sauger all point at `../../../scripts/<name>.sh`), so the
  existing cron should fire cleanly on its next tick.

## Pointers

- sauger app: `sauger/` repo (Next.js). README at `sauger/README.md`.
- Plan: `~/.cursor/plans/edge_training_observatory_3113049b.plan.md`
  (sections "Vera integration" and "Changes by repo → `~/.hermes`").
- Sister skills: `mlops/orzo-pipeline`, `mlops/finetune-recipe-picker`,
  `mlops/eval-rubric-design`, `mlops/jig-trainer`.
