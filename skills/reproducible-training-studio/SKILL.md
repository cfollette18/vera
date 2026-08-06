---
name: reproducible-training-studio
description: Pattern for a reproducible edge-LLM training studio app.
---

# Reproducible training studio — project pattern

A **training studio** is a FastAPI + React (or similar) app for fine-tuning
small LLMs on constrained hardware. User bar: **fine-tuned model +
reproducible eval + writeup-ready artifact**.

Set `$TRAINING_STUDIO_ROOT` to the app repo and keep an **example project**
at `$PROJECT_ROOT` (e.g. a 1.5B agent-harness generator) to dogfood the wizard.

## Locked decisions (template)

1. Writeup = auto-generated `RESULTS.md` + export bundle
2. Contamination check = n-gram MinHash overlap (embedding similarity opt-in)
3. Multi-seed = configurable per run, default `[42]`; publication-ready badge
   requires ≥3 seeds
4. Done = model + reproducible eval + writeup (research-grade, not happy-path)

## Pipeline (16 wizard steps)

1. Setup → 2. Eval rubric → 3. Domain → 4. Specs + freeze → 5. Dataset →
6. Validate + scrub → 7. Contamination → 8. Splits → 9. **Baseline eval** →
10. Recipe → 11. **Pilot** → 12. Train (multi-seed) → 13. Export & Serve →
14. Final eval → 15. Ablations → 16. Writeup & publish

## Key invariants

- Held-out test specs SHA256-frozen at step 4; baseline eval at step 9 **before** training
- JSON-schema enforcement at the teacher (`response_format=json_schema`)
- MinHash dedup at gen-time + post-gen
- Multi-seed training produces N checkpoint dirs
- `reproducibility.json` records dataset, splits, train_config, checkpoint hashes
- `RESULTS.md` auto-generated from app state (SQLite or equivalent)

## Plans and state

- Master plan: `~/.hermes/plans/<studio-name>.md` or
  `$TRAINING_STUDIO_ROOT/docs/PLAN.md`
- Project-specific decisions: `$PROJECT_ROOT/.hermes/` or `AGENTS.md`
- App state: query via API, not raw DB files, when a backend exists

## Open decisions (ask user before coding)

1. Settings encryption: first-launch password vs OS keyring
2. Prose vs schema-wrapped outputs for markdown tasks
3. Quarantine threshold strictness
4. HF publish default: public vs private
5. Standard benchmark hook: shipped vs opt-in
