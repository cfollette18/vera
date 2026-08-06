---
name: edge-qlora-pipeline
description: End-to-end QLoRA agent-harness pipeline on dev laptop + edge GPU.
---

# Edge QLoRA pipeline — dev laptop + edge box

Typical layout for fine-tuning a small agent-harness LoRA on constrained
hardware:

- **`$PROJECT_ROOT`** — model, dataset, eval, and results (your ML project)
- **`$TRAINING_STUDIO_ROOT`** (optional) — generic training-studio app if you
  use one for wizard UI / API
- **`$DEV_HOST`** — x86 laptop: dataset generation, dashboards, no CUDA training
- **`$EDGE_HOST`** — SSH/Tailscale hostname for the edge GPU (Jetson-class,
  ~8 GB unified). Training, export, and GPU eval run here.

Document paths and env vars in the project's `README.md` and `.env.example` —
never hardcode usernames or home directories in skills.

## Environment variables

Teacher dataset generation (OpenAI-compatible API):

```bash
export PROJECT_TEACHER_BASE_URL="https://api.example-provider.com/v1"
export PROJECT_TEACHER_API_KEY="..."   # in $PROJECT_ENV_FILE, mode 600
export PROJECT_TEACHER_MODEL="your-teacher-model"
```

Load with `set -a; source "$PROJECT_ENV_FILE"; set +a` before launching workers.

## Canonical sequence

1. **Generate** — run `data/gen_dataset.py` (or equivalent) per task type until
   row targets are met. Monitor JSONL row counts, not buffered log files.
2. **Dedupe** — run project dedupe script if parallel workers can race on the
   same JSONL.
3. **Validate** — schema + task validators; expect zero failures if the gen
   script only writes passing rows.
4. **Split** — by `spec_id`, not by row, to avoid eval leakage across
   train/valid/test. Record split hashes in `reproducibility.json`.
5. **Sync to edge** — `rsync -av data/ $EDGE_HOST:$PROJECT_ROOT/data/` (or your
   project's sync convention).
6. **Train on edge** — `bash scripts/run_qlora.sh` with train/valid paths,
   output dir, epochs, lr. Set `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
   on Jetson-class hardware when needed.
7. **Log hardware** — tegrastats or equivalent alongside training.
8. **Export** — merge LoRA → GGUF (or your serving format) on edge or dev.
9. **Eval** — run functional eval against base vs fine-tuned; compare pass rates.

## Pitfalls

- Do **not** run CUDA training on `$DEV_HOST` if it has no GPU.
- Do **not** launch long-running workers with shell `&` inside a single
  foreground terminal command — use `terminal(background=true)` or a launcher
  script.
- After re-splitting, back up old splits if you need to compare runs.
- Reasoning teacher models may emit thinking blocks before JSON — strip before
  validation (see `teacher-providers` skill).

## Observability

- Local dashboard: project-specific `dashboard/serve.py` or Trackio (see
  `mlops/trackio`).
- Optional: training observatory events (see `training-observatory`).
