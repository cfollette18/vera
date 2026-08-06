---
name: jig-reproducible-app
description: Use when editing /home/cfollette18/jig. Read plan first.
---

# jig — Reproducible Edge-LLM Training Studio (project memory)

## What the project is
A FastAPI + React app for fine-tuning small LLMs on constrained hardware (Jetson Orin Nano, 8 GB unified memory). User bar is "fine-tuned model + reproducible eval + writeup-ready artifact."

## Repos
- `/home/cfollette18/jig` — the app (backend, frontend, scripts)
- `/home/cfollette18/orzo` — the canonical example project (1.5B agent-harness generator)
- `/home/cfollette18/.hermes/plans/jig-reproducible-app.md` — the master plan, read this first when resuming

## Locked decisions (from the user)
1. Writeup = auto-generated `RESULTS.md` template + export bundle (both available)
2. Contamination check = n-gram MinHash overlap, default fast path (embedding similarity is opt-in strict mode)
3. Multi-seed = configurable per-run, default `[42]`, "publication-ready" badge requires ≥3 seeds
4. Done = fine-tuned model + reproducible eval + writeup-ready artifact (research-grade, not happy-path)

## Pipeline (16 wizard steps)
1. Setup → 2. Eval rubric → 3. Domain → 4. Specs + freeze → 5. Dataset → 6. Validate + scrub → 7. Contamination → 8. Splits → 9. **Baseline eval** → 10. Recipe → 11. **Pilot** → 12. Train (multi-seed) → 13. Export & Serve → 14. Final eval → 15. Ablations → 16. Writeup & publish

## Key invariants the code must preserve
- Held-out test specs SHA256-frozen at step 4; contamination check at step 7; baseline eval at step 9 BEFORE training
- JSON-schema enforcement at the teacher (`response_format=json_schema`), code tasks wrapped in `{ "code": "..." }`
- MinHash dedup at gen-time + post-gen (n=5, 128 perms, threshold 0.85)
- Quality scoring per task (heuristic), PII regex scrub
- Multi-seed training loop produces N checkpoint dirs
- `reproducibility.json` records all hashes (dataset, splits, train_config, checkpoints)
- `RESULTS.md` auto-generated from SQLite state

## Open decisions to ask user about before coding
1. Settings encryption: first-launch password vs libsecret (lean password, headless edge)
2. `rules` task: schema-wrap with `{ "markdown": "..." }` or stay prose
3. Quarantine threshold: 0 strict or >X% lenient
4. HF publish default: public or private
5. Standard benchmark hook: ship enabled or opt-in
