---
name: training-studio-workflow
description: 16-step training studio wizard workflow map.
---

# Training studio — workflow + 16-step wizard

A **training studio** (`$TRAINING_STUDIO_ROOT`) is a local, reproducible
app for fine-tuning small LLMs on constrained hardware. It dogfoods an
example project at `$PROJECT_ROOT`.

Stack (typical): FastAPI + SQLite backend, React + Vite frontend, SSE for
run progress, subprocess workers in `scripts/`.

**Plans:** `~/.hermes/plans/reproducible-training-studio.md` (engineering
contract), optional researcher reframing plan alongside it.

**State:** query via REST API — not raw SQLite (`$TRAINING_STUDIO_STATE_DB`).

## Where to run what

| Host | Role |
|------|------|
| Dev laptop | Code edits, dataset gen, UI at `http://localhost:8001` |
| `$EDGE_HOST` | Training, export, GPU eval via SSH |

## 16-step wizard (summary)

1. Setup → 2. Eval rubric → 3. Domain → 4. Specs + freeze →
5. Dataset → 6. Validate + scrub → 7. Contamination → 8. Splits →
9. Baseline eval → 10. Recipe → 11. Pilot → 12. Train →
13. Export & serve → 14. Final eval → 15. Ablations → 16. Writeup

Done bar: **fine-tuned model + reproducible eval + writeup artifact**.

See `reproducible-training-studio` skill for invariants and open decisions.

## How to work on the studio

1. **Studio code** — `backend/server/`, frontend under `frontend/` or `ui/`.
2. **Advance a project** — REST API (`training-studio-api` skill) or run
   `scripts/` directly on `$EDGE_HOST`.
3. **In-app agent** — see `training-studio-trainer` skill.

## Sister skills

- `training-studio-api` — REST surface
- `training-studio-recipes` — QLoRA recipes + fit_check
- `training-studio-trainer` — autonomous in-app agent
- `edge-qlora-pipeline` — edge-box command sequence without the UI
