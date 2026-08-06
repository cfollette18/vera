---
name: trackio
description: trackio live experiment-tracking dashboard for SFT / QLoRA / dataset gen.
---

# trackio — local-first experiment tracking + live UI

`trackio` (https://github.com/gradio-app/trackio, MIT, HF-built) is a
wandb-compatible experiment tracker. Stores runs in a local SQLite database
under `~/.cache/huggingface/trackio/<project>.db`; the Gradio dashboard reads
that DB and exposes a web UI for live loss curves, hyperparameter comparison,
run management.

Use this skill when the user asks for:
- "live training loss dashboard"
- "wandb alternative that runs on my Jetson"
- "log metrics to a local SQLite DB and view them in a browser"
- "compare two QLoRA runs side by side"

## Install (already done in orzo venv)

```
source /home/cfollette18/orzo/.venv/bin/activate
uv pip install trackio   # 0.34.0 at writing
```

## Minimal pattern (Python API)

```python
import trackio

trackio.init(
    project="orzo-qlora",
    config={"base": "Qwen2.5-Coder-1.5B", "lr": 2e-4, "bs": 4, "rank": 16},
)
for step, batch in enumerate(dataloader):
    loss = model(batch).loss
    trackio.log({"loss": loss.item(), "step": step})
    if step % 50 == 0:
        trackio.log({"lr": scheduler.get_last_lr()[0]})
trackio.finish()
```

## Reserve-key warning

`step` is reserved by trackio. Calling `log({"step": step})` raises a
UserWarning and renames the key to `__step`. Use `trackio.log({"step": step})`
**only** to pass the x-axis; if you don't, trackio auto-counts.

## HF Trainer integration (drop-in)

```python
from transformers import TrainingArguments, Trainer
from trackio.integration.huggingface import TrackioCallback  # confirms import path

args = TrainingArguments(
    output_dir="runs/orzo-qwen25-coder-1.5b",
    report_to="trackio",   # HF auto-detects
    ...
)
trainer = Trainer(args=args, model=model, ...)
trainer.train()
```

If the auto-callback path doesn't import, fall back to a manual
`TrackioCallback` (subclass `TrainerCallback`) that calls `trackio.log`
on `on_log`.

## Dataset-gen logging

For orzo's `gen_dataset.py`, log per-spec results into a `orzo-dataset-gen`
project so you can watch validator pass rate live:

```python
import trackio
trackio.init(project="orzo-dataset-gen",
             config={"task": "harness_scaffold", "teacher": "MiniMax-M3"})
# per spec:
trackio.log({
    "pass": int(validate_ok),
    "tokens": completion.usage.completion_tokens if hasattr(...,"usage") else 0,
    "step": spec_idx,
})
# periodic aggregate:
trackio.log({"rolling_pass_rate": sum(window)/len(window)})
```

## Launch the dashboard

```
trackio show --project <name>            # binds 127.0.0.1:7860 by default
trackio show --project <name> --host 0.0.0.0    # LAN-accessible for Jetson
```

Programmatic:
```python
import trackio
trackio.show(project="orzo-qlora", host="0.0.0.0")  # blocks; call from a worker
```

CLI subcommands worth knowing:
- `trackio list projects|runs|metrics` — what's been logged
- `trackio query "<sql>"` — read-only SQL against the project DB
- `trackio status` — local + HF Spaces sync state
- `trackio sync --project <p> --space <hf-user>/<space>` — push to HF
  Spaces (free public hosting; uses HF token)
- `trackio freeze` — one-shot static snapshot of a project
- `trackio skills` — generate AI-coding-assistant skill files for a project

## Multi-project baseline comparison

To compare base-model vs orzo-model loss curves on the same plot: log both
runs to the same project (different `run_name`):

```python
trackio.init(project="orzo-experiments", name="base-qwen25-coder-1.5b", config={...})
# ... train base ...
trackio.finish()

trackio.init(project="orzo-experiments", name="orzo-v1-qwen25-coder-1.5b", config={...})
# ... train orzo ...
```

Note: trackio.init does NOT take `name=`. Instead, pass it via config or
use a fresh call without init sharing — see "Pitfalls" below.

## Pitfalls

- **`trackio.init()` does NOT take `name=`** for the run; the run name is
  auto-generated (e.g. `dainty-sunset-0`). To set a custom name, pass
  `name=` — actually double-check the installed version's API by
  `trackio.init --help` or `inspect.signature(trackio.init)`.
- **Project names are canonically normalized**: trackio's
  `canonical_project_name()` strips everything not `[A-Za-z0-9_-]`, so
  `orzo-dataset-gen/rules` becomes `orzo-dataset-genrules` and **collides**
  with a project literally named `orzo-dataset-genrules`. Use `-` as a
  namespace separator if you want sub-projects (e.g.
  `orzo-dataset-gen-rules`, `orzo-dataset-gen-guardrails`).
- **No `--port` flag on `trackio show`** — Gradios picks 7860 by default.
  To move it: edit the Gradio launch or run behind a reverse proxy.
- **Buffers metrics in DB only**. There is NO real-time websocket push to
  the UI; the Gradio dashboard polls the SQLite DB. So step-to-step lag is
  bounded by SQLite write latency (<50ms typically) and the polling
  interval. Fine for SFT, NOT good for sub-second agent harness tracing —
  use a separate tool (Phoenix, LangSmith) for that.
- **Multiple processes logging concurrently to the same SQLite DB** can
  cause `database is locked` errors. Use a single orchestrator process for
  parallel logging, or run separate projects per worker.
- **Reserved keys** (`step`, `epoch`, `_step`, `_epoch`, `_runtime`, `_timestamp`)
  get renamed to `__step`, `__epoch`, etc. with a UserWarning. Avoid those
  metric names unless you specifically want them on the x-axis.
- **HF Spaces sync** requires a write token (`huggingface-cli login` first)
  and creates a PUBLIC space. Don't sync experiment logs that contain
  business-sensitive data without thinking about it.
- **Don't confuse with `tensorboard`**: trackio is for run metadata +
  scalar metrics; tensorboard eats event files for graph/embedding
  visualization. For Orin Nano class hardware, trackio wins on simplicity.

## Where it lives (orzo-specific)

- venv: `/home/cfollette18/orzo/.venv` (uv-installed, version 0.34.0).
- DB dir: `/home/cfollette18/.cache/huggingface/trackio/`.
- Launcher: `/home/cfollette18/orzo/scripts/trackio_dashboard.sh` —
  sources PATH, picks an unused port, nohups the dashboard, returns the URL.

## Recommended orzo wiring

1. In `train/train_qlora.py` (or wherever the HF `Trainer` lives on
   heater): set `report_to="trackio"` and `project=` (or wrap with a
   manual `TrackioCallback`).
2. In `data/gen_dataset.py`: add a per-task trackio logger writing to
   project `orzo-dataset-gen-<task>` so each task gets its own dashboard.
3. On heater, expose port 7860 over Tailscale so the dev laptop can view.
4. The dev-laptop dashboard (port 8000) becomes a thin index that links
   to the per-task Trackio dashboards + the live JSONL counters.
