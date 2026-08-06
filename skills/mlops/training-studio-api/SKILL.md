---
name: training-studio-api
description: FastAPI backend REST surface for training studio projects.
---

# Training studio — backend REST surface

Base URL: `http://localhost:8001` on the dev laptop. Over Tailscale:
`http://$EDGE_HOST:8001`. Paths below assume a global `/api` prefix
(check `backend/server/main.py`).

Health: `GET /api/health` → `{"status": "ok"}`
SSE: `GET /api/stream` — global run-event fan-out.

## Endpoints (typical routers)

**Projects** (`/projects`):
- `GET /projects` — list
- `POST /projects` — create `{name, path}` where `path` is the on-disk
  project root (e.g. `$PROJECT_ROOT`); does not create the directory
- `GET /projects/{id}` — get
- `GET /projects/{id}/status` — aggregate status

**Runs, specs, dataset, splits, training, eval, export, system** — each
under `/projects/{id}/...`. See the app's OpenAPI docs at `/docs` when
running locally.

## Common body shapes

`start_dataset_task(task: str)` — task name is project-defined (e.g.
`harness_scaffold`, `tool_schema`, `rules`).

`start_training()` — QLoRA params: `{recipe?, base_model?, data, valid?,
output, epochs?, lr?, seed_list?, seq_len?, batch?, grad_accum?, lora_r?,
lora_alpha?}`.

`start_eval()` — `{model, specs, task, smoke?, out}` where `model` is an
Ollama name or HF id.

## Sister skills

- `training-studio-workflow` — wizard map
- `training-studio-recipes` — recipe dataclasses
- `edge-qlora-pipeline` — CLI-only path on the edge box
