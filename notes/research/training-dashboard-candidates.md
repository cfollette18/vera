# Dashboard candidates for ML training projects

Comparison of local-first experiment trackers suitable for edge QLoRA and
dataset-generation workflows.

## Trackio (Hugging Face)

- **Why it fits:** Built for fine-tuning. Local SQLite/Parquet store,
  wandb-compatible API (`import trackio as wandb`), Gradio dashboard,
  optional HF Spaces hosting. HF Trainer json logs work via `log({...})`.
  Pure pip install.
- Fit: ★★★★★ — drop-in for HF ecosystem; pairs with a simple
  `dashboard/serve.py`.

## MLflow

- **Why it fits:** Mature artifact + metric store; local server mode.
  Heavier than Trackio but good if you already use MLflow elsewhere.
- Fit: ★★★☆☆ — more setup for a solo edge box.

## Weights & Biases (offline / local)

- **Why it fits:** Industry standard charts; local mode or self-hosted.
- Fit: ★★★☆☆ — account + sync assumptions unless self-hosted.

## TensorBoard

- **Why it fits:** Zero new deps if already on PyTorch; scalar curves only.
- Fit: ★★☆☆☆ — no dataset-gen or eval tables without custom writers.

## Optuna Dashboard

- **Why it fits:** Live UI for HPO sweeps; HF Trainer callbacks exist.
- Fit: ★★★☆☆ — when hyperparameter search is the focus.

## Recommendation

For a portable harness profile, document **Trackio** in `mlops/trackio` and
wire projects via env vars (`TRACKIO_PROJECT`, `TRACKIO_HOST`). Keep
project-specific dashboard URLs in `$PROJECT_ROOT/.env`, not in skills.
