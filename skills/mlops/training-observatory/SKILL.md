---
name: training-observatory
description: Integrate dataset/train/eval jobs with a self-hosted observability app.
---

# Training observatory — event contract for edge-ML jobs

A **training observatory** is a self-hosted web app (typically Next.js +
SQLite) that ingests structured events from dataset generation, training,
eval, export, and hardware probes. The agent orchestrates project scripts;
emitters push events so progress is visible from any machine on the LAN or
Tailscale.

Set `$OBSERVATORY_ROOT` to your observatory app repo and `$OBSERVATORY_URL`
to its base URL (e.g. `http://localhost:3000`).

## Endpoints (typical shape)

- **Ingest** (called by project scripts, not the agent directly):
  `POST $OBSERVATORY_URL/api/events` with header
  `Authorization: Bearer $OBSERVATORY_RUN_TOKEN`. Body is one event or a
  JSON array. Invalid shape → 400; bad token → 401.
- **Job trigger** (UI or agent):
  `POST $OBSERVATORY_URL/api/jobs` with body
  `{ "job": "dataset_gen"|"eval"|"benchmark"|"train", "project": "<name>", "mode": "direct"|"agent" }`.
  `direct` = observatory shells out to the project script. `agent` = observatory
  opens an agent run via the Hermes Runs API. Both modes should emit the same
  event types.

## Event envelope

```json
{
  "ts": 1722953023.45,
  "project": "my-agent-harness",
  "node": "edge-1",
  "stage": "dataset_gen",
  "event_type": "row_written",
  "run_id": "my-agent-harness-20260806-001",
  "payload": {}
}
```

- `project`: logical project name (string you choose)
- `node`: `"laptop"` | `"edge-1"` | `"edge-2"` | `"unknown"`
- `stage`: `"dataset_gen"` | `"split"` | `"contamination"` | `"train"` |
  `"eval"` | `"benchmark"` | `"export"` | `"hw"` | `"alert"`
- `run_id`: `<project>-<YYYYMMDD>-<NN>` — one ID per end-to-end job

## Common event types

Dataset: `row_written`, `validation_fail`, `split`, `contamination`

Training: `step`, `checkpoint`, `eval_checkpoint`

Eval: `eval_item`, `eval_summary`, `benchmark`

Hardware: `hw_sample` (GPU util, mem, power, temp)

Alerts: `alert` with `kind` ∈ `thermal|budget|oom|failed|regression`

## How the agent reports progress

**Run the project scripts.** Scripts should call a small emitter helper
(gated by `PROJECT_OBSERVATORY=1`) that POSTs to `/api/events`. The agent
does not need to wrap every step manually.

At job start, mint a `run_id` and export `OBSERVATORY_RUN_ID=$run_id` to
every child process so all events share the same run.

## Watchdog (optional)

A shell watchdog can poll the observatory API every N minutes and alert on
down API or stuck runs (no new event for >15 min while status=active).
Register via `hermes cron create` pointing at a script under
`~/.hermes/scripts/`.

## Sister skills

`mlops/edge-qlora-pipeline`, `mlops/finetune-recipe-picker`,
`mlops/eval-rubric-design`, `mlops/trackio`
