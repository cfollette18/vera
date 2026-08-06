# Orzo Dashboard Candidates — GitHub Research

**Context:** orzo runs QLoRA agent-harness dataset generation + LoRA training on a Jetson Orin Nano (7.4 GiB) + x86 dev laptop. HF Trainer writes standard json logs. User wants to view live progress from anywhere. Constraints: MIT/Apache/BSD, self-hostable, active (commits ≤12 months), OpenAI-compatible API support.

## Top picks (ranked for orzo's needs)

### 1. gradio-app/trackio — **STRONGLY RECOMMENDED**
- URL: https://github.com/gradio-app/trackio
- License: MIT
- Stars: 1,621 / Pushed: 2026-08-05 (very active, by Hugging Face team)
- **Why it fits orzo:** Built by HF for fine-tuning use cases. Local-first SQLite/Parquet store, wandb-compatible API (`import trackio as wandb`), Gradio dashboard, optional free cloud hosting on HF Spaces. CLI/SQL query surface (LLM-friendly). HF Trainer json logs work via the wandb-style `log({...})` API. Pure pip install, no infra.
- Fit: ★★★★★ — drop-in compatible with HF ecosystem, trivial to integrate with the existing orzo/dashboard/serve.py.

### 2. aimhubio/aim
- URL: https://github.com/aimhubio/aim
- License: Apache-2.0
- Stars: 6,218 / Pushed: 2026-08-04
- **Why it fits:** Self-hosted experiment tracker with native HF Transformers/Hugging Face Trainer callback (`AimCallback`). Robust comparison UI for many runs. Self-host via `aim up` (single command). Active. Heavyweight — backend is a separate process.
- Fit: ★★★★ — best for comparing many runs; more setup than trackio.

### 3. mlflow/mlflow
- URL: https://github.com/mlflow/mlflow
- License: Apache-2.0
- Stars: 27,376 / Pushed: 2026-08-05
- **Why it fits:** Industry standard. MLflow has `mlflow.transformers.autolog()` for HF Trainer, plus native LLM tracing + an OpenAI-compatible gateway (`mlflow.openai.autolog()`). `uvx mlflow server` runs the UI locally.
- Fit: ★★★★ — broad and battle-tested; more overhead than trackio; strong if user wants the AI Gateway for OpenAI-compatible routing too.

### 4. tensorflow/tensorboard
- URL: https://github.com/tensorflow/tensorboard
- License: Apache-2.0
- Stars: 7,198 / Pushed: 2026-07-30
- **Why it fits:** `tensorboard --logdir ...` ingests HF Trainer event files directly (Trainer auto-writes to `runs/`). Self-host = one command. Zero-dependency, but UI is spartan and no native dataset-viewer.
- Fit: ★★★ — easiest possible fit for the loss-curve half; weak for the dataset/harness half.

### 5. optuna/optuna
- URL: https://github.com/optuna/optuna
- License: MIT
- Stars: 14,610 / Pushed: 2026-08-05
- **Why it fits:** `optuna-dashboard` package gives a live web UI for HPO sweeps. `MLflowCallback` and direct HF Trainer integration. Useful for orzo if hyperparameter sweeps become a focus.
- Fit: ★★★ — best for HPO, not a general training dashboard.

### 6. clearml/clearml
- URL: https://github.com/clearml/clearml
- License: Apache-2.0
- Stars: 6,812 / Pushed: 2026-07-27
- **Why it fits:** Self-hostable MLOps platform with full pipeline + dataset + experiment tracking. Heavier infra (server + web UI + agents).
- Fit: ★★ — overkill for a single Jetson + laptop setup.

### 7. huggingface/dataset-viewer
- URL: https://github.com/huggingface/dataset-viewer
- License: Apache-2.0
- Stars: 895 / Pushed: 2026-08-04
- **Why it fits:** Specifically the *dataset* half (not training). Powers the HF Hub dataset preview; can be self-hosted for viewing parquet/jsonl shards.
- Fit: ★★ — pairs with #1–#4 for the dataset-viewing half.

## How they handle the user's constraints
- **MIT/Apache/BSD**: all 7 above ✓
- **Self-hostable**: all 7 ✓ (trackio via `trackio show`, aim via `aim up`, mlflow via `mlflow server`, tensorboard via `tensorboard --logdir`)
- **Active (≤12 months)**: all 7 ✓ — every top pick had a commit within the last 2 weeks as of 2026-08-05
- **HF Trainer json log compat**: trackio (wandb-style API) and aim (`AimCallback`) read Trainer metrics directly; mlflow via `mlflow.transformers.autolog()`; tensorboard ingests event files. HF dataset-viewer reads parquet/jsonl directly.
- **OpenAI-compatible APIs**: mlflow has the AI Gateway that fronts any OpenAI-compatible endpoint (relevant for orzo's teacher model routing). trackio has HF Spaces as optional backend.

## Notable exclusions
- **neptune-ai/neptune-client** (Apache-2.0) — **ARCHIVED** 2026-03, dropping support.
- **wandb/wandb** (MIT, 11k stars) — strong but cloud-first; self-host requires the closed-source enterprise tier, so it fails the "self-host" constraint.
- **salujayatharth/wa-finetune**, **deep-diver/llm-finetuning-dashboard**, **zijieguo2003/offline-training-dashboard** — too small/stale, single-author hobby projects with <5 stars.

## Recommendation for orzo
Start with **trackio** — MIT, HF-built, local-first, drop-in wandb-style API, supports custom frontends and HF Spaces if user later wants remote viewing. Add **tensorboard** if richer loss-curve UX is needed (zero-cost coexistence). Consider **mlflow** later if OpenAI-compatible gateway + agent tracing become priorities.
