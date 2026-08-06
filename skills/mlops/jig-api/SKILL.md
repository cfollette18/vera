---
name: jig-api
description: jig FastAPI backend REST surface for advancing projects.
---

# jig — backend REST surface (live as of 2026-08-04)

Base URL: `http://localhost:8001` on the dev laptop. On heater, port
8001 over Tailscale at `http://heater:8001`. All paths below are
mounted under `/api` if FastAPI is configured with a global prefix
(check `backend/jig_server/main.py`); the routers use `/projects/...`.

Health: `GET /api/health` → `{"status": "ok"}`
SSE: `GET /api/stream` (text/event-stream) — global run-event fan-out
from `services/sse.py`. Subscribe here when watching for run progress.

## Endpoints (9 routers, 21 routes)

Projects (`/projects`):
  GET    /projects                                — list_projects()
  POST   /projects                                — create_project(ProjectCreate)
  GET    /projects/{project_id}                   — get_project()
  GET    /projects/{project_id}/status            — project_status()

Runs (`/projects/{project_id}/runs`):
  GET    /projects/{project_id}/runs              — list_runs()
  GET    /projects/{project_id}/runs/{run_id}     — get_run()
  POST   /projects/{project_id}/runs/{run_id}/cancel — cancel_run()

Specs (`/projects/{project_id}/specs`):
  POST   /projects/{project_id}/specs             — start_specs()

Dataset (`/projects/{project_id}/dataset`):
  GET    /projects/{project_id}/dataset           — get_dataset_progress()
  POST   /projects/{project_id}/dataset/{task}    — start_dataset_task(task)
  GET    /projects/{project_id}/dataset/examples  — list_examples()

Splits (`/projects/{project_id}/splits`):
  POST   /projects/{project_id}/splits            — create_splits()

Training (`/projects/{project_id}/training`):
  GET    /projects/{project_id}/training          — list_training_runs()
  POST   /projects/{project_id}/training          — start_training()
  GET    /projects/{project_id}/training/{run_id} — get_training_run()

Eval (`/projects/{project_id}/eval`):
  GET    /projects/{project_id}/eval              — list_eval_runs()
  POST   /projects/{project_id}/eval              — start_eval()
  GET    /projects/{project_id}/eval/{run_id}     — get_eval_run()

Export (`/projects/{project_id}/export`):
  GET    /projects/{project_id}/export            — list_export_runs()
  POST   /projects/{project_id}/export            — start_export()

System (`/projects/{project_id}/system`):
  GET    /projects/{project_id}/system            — system_metrics()

## Body shapes (Pydantic, from models.py)

`ProjectCreate` — at minimum `{name: str, path: str}`. `path` is the
absolute path on disk where the project lives (e.g. `/home/cfollette18/
orzo`). jig creates the project record + SQLite state but does NOT
create the directory.

`start_dataset_task(task: str)` — `task` ∈ `{"harness_scaffold",
"react_trace", "tool_schema", "rules", "guardrails", "hooks",
"skills"}`. Validators are per-task strict; failed examples are
counted but the run still completes.

`start_training()` body (from `scripts/qlora.py` CLI surface):
  `{recipe?: str, base_model?: str, data: str, valid?: str, output:
  str, epochs?: float, lr?: float, seed_list?: int[], seq_len?:
  int, batch?: int, grad_accum?: int, lora_r?: int, lora_alpha?:
  int}`. Defaults match the orzo defaults (QLoRA r=16, seq 2048,
  paged 8-bit Adam, bf16).

`start_eval()` body: `{model: str, specs: str, task: str, smoke?:
bool, out: str}`. `model` is an Ollama-served name OR an HF id; the
router talks to the local Ollama endpoint (default
`http://localhost:11434/v1`; or `http://heater:11434/v1` from the
laptop).

`start_export()` body: `{checkpoint: str, base_model: str,
quant?: "Q4_K_M" | "Q5_K_M" | "Q8_0" | "IQ2_M"}`. Default
quantization = Q4_K_M.

## Run lifecycle (from models.py)

`Run.kind` ∈ `{"specs","dataset","splits","training","eval",
"export","system"}`
`Run.status` ∈ `{"queued","running","succeeded","failed","canceled"}`
`RunMetric` rows store per-step metrics (loss, lr, vram, etc.)
`RunLogLine` rows store stdout/stderr fragments — streamed to SSE.
`SseEvent` is `{type: str, payload: str}` — emitted by the runner.

## Conventions

- `POST` endpoints return `{run_id: str, ...}` and immediately start
  the subprocess; subsequent progress comes via SSE on `/api/stream`
  or by polling `GET .../runs/{run_id}`.
- All `{project_id}` placeholders refer to the SQLite project row id
  (integer or UUID depending on `db.py` schema). Get it from
  `GET /projects` first if you don't have it.
- Errors are RFC 7807 problem-detail JSON; HTTP 4xx with `detail`
  field for validation, 5xx for runner errors.

## How to drive the API from this agent

Pattern for starting a long-running stage:
  1. `POST` to start, capture `run_id`.
  2. Subscribe to `GET /api/stream` (curl with `-N` or Python's
     `httpx` stream) to watch events.
  3. Poll `GET .../runs/{run_id}` until status ∈
     {`succeeded`,`failed`,`canceled`}.
  4. On failure, `GET .../runs/{run_id}` returns the last log lines
     in `RunLogLine` rows; surface them to the user.

To cancel a runaway: `POST .../runs/{run_id}/cancel`. The runner
SIGTERMs the subprocess and updates status.

## What this skill does NOT cover

- The internal Pydantic field names beyond what's needed to build
  request bodies. For full fidelity, read
  `backend/jig_server/models.py`.
- Phase-1 endpoints (settings, wizard, rubric, agent, ablation,
  results) that are PLANNED but NOT YET IMPLEMENTED. Check the
  routers — if a route returns 404, the feature isn't shipped.