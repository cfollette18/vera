---
name: jig-workflow
description: jig training studio 16-step wizard workflow.
---

# jig — workflow + 16-step wizard map

jig (`/home/cfollette18/jig`) is a local, reproducible training studio
for fine-tuning small LLMs on constrained hardware. Used to build
orzo (`/home/cfollette18/orzo`). Backend = FastAPI+SQLite. Frontend =
React 19 + Vite + Tailwind 4 + Recharts. Real-time = SSE. Worker
scripts in `scripts/` are called via subprocess and their stdout is
streamed to the browser.

When applying the researcher lens to jig, see the companion plan at
`/home/cfollette18/.hermes/plans/jig-researcher-reframing.md` — it
records the gaps a researcher would add to the current 16-step
wizard. The main plan at
`/home/cfollette18/.hermes/plans/jig-reproducible-app.md` is the
engineering contract.

**Source of truth for design decisions:**
`/home/cfollette18/.hermes/plans/jig-reproducible-app.md`. Read that
plan first when resuming a non-trivial jig change. Memory has a
compressed summary; the plan is the canonical doc.

**Source of truth for project state:** SQLite at
`~/.local/share/jig/jig.db`. Always query via the API, not the DB
directly.

**Where to run what:**
- Dev laptop (`/home/cfollette18/jig`, x86_64, no GPU, no CUDA):
  code edits, dataset generation, dashboard at
  `http://localhost:8001/` (or 5173 dev mode), REST API at
  `http://localhost:8001`.
- Edge box (`heater`, Tailscale, aarch64, 8 GB, MAXN_SUPER): all
  training, export, and eval. SSH: `ssh heater`. Repo lives at
  `~/orzo` on heater; `~/orzo-venv` is the venv.

## The 16-step wizard (plan §2, reframed as a researcher workflow)

The "done" bar is: **fine-tuned model + reproducible eval + writeup-ready
artifact.** Every step has an unlock criterion; the wizard surfaces
locked steps as blurred with a tooltip. Steps in **bold** are new in
the plan (don't yet exist as pages).

### Current plan order

| # | Step | Today's page / new | Unlocks when |
|---|------|--------------------|--------------|
| 1 | Setup | Dashboard | Settings saved + teacher test-connection green |
| 2 | **Eval rubric** | new | Rubric saved + ≥1 task defined |
| 3 | **Domain** | Specs (config) | Domain chosen + tool pools + constraints valid |
| 4 | Specs + freeze | Specs | ≥N specs; train/test spec SHA256 recorded |
| 5 | Dataset | Dataset | All tasks ≥90% target; quarantine + dedup_drop + quality_drop + pii_drop = 0 |
| 6 | **Validate + scrub** | new | validate_dataset.py exits 0 |
| 7 | **Contamination** | new | check_contamination.py: 0 pairs above threshold |
| 8 | Splits | (merged into 4) | train/valid/test exist with hashes |
| 9 | **Baseline eval** | new | Base model eval done; baseline pass rate captured |
| 10 | Recipe | Training (config) | Recipe picked + seeds set + fit_check passes |
| 11 | **Pilot** | new | Pilot run completes without divergence |
| 12 | Train | Training | All N seed checkpoints have adapter_config.json |
| 13 | Export & Serve | Export | .gguf exists AND ollama reports model loaded |
| 14 | Final eval | Eval | Fine-tuned eval done; per-seed mean ± std captured |
| 15 | **Ablations** | new | ≥1 ablation done (or user explicitly skips) |
| 16 | **Writeup & publish** | new | RESULTS.md rendered; export.zip downloadable; HF publish optional |

### Researcher reframing (what an AI researcher would change)

The plan's order is mostly right but encodes three subtle bugs of
framing that make the artifact less defensible. Researchers think in
terms of: **instrument → data → baseline → train → eval → artifact**,
with **pre-registration** as the load-bearing concept. Recommended
regrouping (same 16 steps, reordered by research cognitive sequence):

**Pre-wizard — pre-registration**
- P.0 Project intent — one-paragraph statement of capability, why this
  base model, smallest result that would count as progress. Free-text;
  locked once written (audit notes only).
- P.1 Success criterion — per-task expected baseline range + expected
  delta after fine-tune, with uncertainty. **Hashed and locked**; any
  edit requires an audit note ("changed expected delta for harness_scaffold
  from +5% to +8% because pilot revealed faster convergence than expected").

**Phase A — Instrument (the rubric is the most important artifact)**
- 1. Setup — teacher creds, cost cap, hardware inventory.
- 2. Eval rubric — define tasks + scorers; rubric **discriminative-power
  check** (does it separate confident-correct from confident-wrong?).
- 3. **Rubric sanity** (NEW in plan terms) — manually annotate 20 random
  base-model outputs against the rubric; fix disagreements. **30 min
  that saves days of confused debugging.** Without this, post-training
  numbers may be measuring the wrong thing.

**Phase B — Data**
- 4. Domain + specs + freeze — input distribution; frozen test set
  stratified by difficulty (easy / medium / hard — see below).
- 5. Dataset gen — with live **failure taxonomy**: capability gap /
  format gap / robustness gap / generalization gap. Different
  remediation per bucket. Today the plan collapses these into
  "quarantine / dedup / quality / pii" which is insufficient.
- 6. **Data smoke** (NEW) — train on 50 examples for 50 steps, eyeball
  loss curve + spot-check generated outputs. Catches "data is unlearnable"
  before the user burns a multi-seed budget on it.

**Phase C — Baseline (you must know where you're starting)**
- 7. Baseline eval — base model on stratified test set; per-task +
  per-stratum; latency, token cost, rubric discriminative power.
- 8. Expected outcome — user writes expected delta per task; **hashed
  and locked at this point**. The wizard will compare actual vs
  expected at step 13 and flag surprises (positive OR negative).

**Phase D — Training**
- 9. Recipe + seeds + fit_check.
- 10. Pilot — convergence + capacity check on 1-5% data; abort on
  divergence.
- 11. Train — multi-seed serial loop with **early stopping across
  seeds**: if seed 1 regresses vs baseline on the rubric, the wizard
  prompts before continuing to seed 2. (Today's plan trains all seeds
  unconditionally.)

**Phase E — Evaluation**
- 12. Final eval — fine-tune on stratified test; per-task + per-stratum
  + per-seed mean ± std.
- 13. Comparison — pre-registered expected vs actual; flag surprises.
- 14. Ablations — **mandatory** (≥2, not optional). "Optional but
  recommended" is fine for engineering; for research it's a floor.
  Default pair: remove `rules` / remove `react_trace` (mirrors the
  two largest data-share tasks).

**Phase F — Artifact**
- 15. **Failure analysis** (NEW) — for any task that regressed,
  diagnose which of the 4 failure-mode buckets (capability / format /
  robustness / generalization) and write a hypothesis. Researchers do
  this; engineers often skip it.
- 16. Writeup + publish — auto-generated RESULTS.md + reproducibility
  bundle + HF push.

### Concepts the plan is missing entirely (research hygiene)

These are the items a researcher would add at step 1 of any project,
that the current plan doesn't surface:

- **Statistical power analysis.** With 3 seeds and a 200-spec test set,
  the std on a pass-rate metric is large. Wizard should warn: "for a
  ±3% delta to be detectable at p<0.05 with 3 seeds, you need ≥X test
  specs per task." Shapes both the test-set size and the seed count.
  Today the plan just says "default [42]; ≥3 seeds for publication-ready."
- **Held-out difficulty stratification.** Today's test set is 200 random
  specs. Stratify (easy / medium / hard — by teacher perplexity, or by
  a difficulty rubric) and break down reports per stratum. This is how
  you spot the "fine-tune memorizes train distribution but loses
  hard cases" failure mode.
- **Out-of-distribution (OOD) test set.** 20-50 specs in a different
  domain, same task. Measures true generalization. Today the plan has
  no OOD concept.
- **Compute budget transparency.** User sees cost cap at setup but not
  cumulative spend across dataset gen + train × seeds + eval × seeds +
  ablations. Track from step 1; show on every step's footer.
- **Hyperparameter sensitivity (small sweep).** Recipes are fixed.
  Researcher reality: sweep lr ∈ {1e-4, 2e-4, 5e-4} and lora_r ∈ {8,
  16, 32} on a subset, pick the best, then full-train. The plan
  implicitly assumes hyperparams are fixed; a small sweep fits between
  step 10 (Pilot) and step 11 (Train).
- **Pre-registration hashing.** Steps P.1 + step 8 hash the rubric and
  expected outcomes; later edits require an audit note. Currently the
  plan just records SHAs; doesn't enforce immutability.

### Failure-mode taxonomy (proposed for step 5)

Bucket every drop into one of four a-priori failure modes. Each has a
different remediation:

| Bucket | Symptom | Likely cause | Remediation |
|--------|---------|--------------|-------------|
| Capability gap | Base model scores 0% on the task | Task is out-of-distribution for the base | Reconsider task scope OR pick a stronger base |
| Format gap | Base model scores 0% on format, ~X% on content | Model has the skill, won't follow format | Few-shot examples in the rubric; fine-tune harder |
| Robustness gap | Easy inputs fine, hard inputs fail | Distribution gap in fine-tune data | Stratified training; upweight hard examples |
| Generalization gap | Train-domain improves, OOD test regresses | Memorization, not learning | More diverse data; smaller LoRA rank; regularization |

The plan's quarantine / dedup_drop / quality_drop / pii_drop captures
*content* failures (what was generated) but not *behavior* failures
(how the model behaves on different inputs). The taxonomy above is
complementary; both should ship.

### Negative-result handling

The current jig-trainer playbook (plan §7) has an "eval regression →
revert or retry at smaller LR" decision. Researchers would add:

- **Early-stop across seeds.** If seed 1 regresses on the rubric vs
  baseline, pause and ask the user whether to continue. With 3 seeds,
  you only need 2 of 3 to confirm a real regression; with 1 seed, you
  can't distinguish "bad seed" from "bad fine-tune."
- **Hyperparam sweep as escape hatch.** If all seeds regress, the
  wizard should suggest a small lr/r sweep before declaring the
  project a negative result. Researchers expect negative results and
  want them documented cleanly, not buried.

## How to work on jig

1. **Edit jig itself** — backend FastAPI code in
   `backend/jig_server/`, frontend React code in
   `frontend/src/routes/` + `components/`. Hot-reload via
   `./run.sh` (uvicorn --reload for backend, Vite HMR for frontend).
2. **Advance a registered project** — use the API
   (`POST /projects`, `POST /projects/{id}/dataset`, etc.); the
   `jig-api` skill has the surface. Or run the scripts in `scripts/`
   directly with the env vars set.
3. **Inspect state** — `GET /projects/{id}/runs`,
   `GET /projects/{id}/training/loss-curve`, etc. The DB has it but
   don't read it directly.
4. **Add a new wizard step** — touch 4 files: backend router,
   frontend route under `frontend/src/routes/wizard/`, sidebar/stepper
   component, `models.py` for the Pydantic type. Existing pattern: see
   how `Eval` (router + route) wires up.

## Things you must NEVER do

- **Never run training on the dev laptop.** No GPU, no CUDA, the
  `qlora.py` recipe expects a CUDA device. All training work goes on
  `heater`.
- **Never edit test specs after they've been frozen** (step 4). Their
  SHA256 is recorded; the eval router reads from `test_specs.jsonl`
  only. If a test spec needs to change, add it to the held-out pool
  and re-freeze, never edit in place.
- **Never delete or rewrite `train.jsonl` / `valid.jsonl` /
  `test.jsonl` without recording the prior SHA256 in
  `reproducibility.json`.** Splits are part of the reproducibility
  manifest; losing them means losing the ability to bit-replay a run.
- **Never bypass the validator** when adding examples to a task JSONL.
  The strict Python validators per task + the future JSON-schema
  validator are the contract. If you need new content that doesn't
  fit, extend the schema/validator first.
- **Never `kill 28383`** (the original orzo harness_scaffold worker)
  on the dev laptop without a replacement — it's the only one with a
  deepseek-chat process slot warm and its Python interpreter is in
  memory at `/home/cfollette18/.local/share/uv/python/cpython-3.11.15-
  linux-x86_64-gnu/bin/python3.11` (no venv needed).

## Key files to know

| Path | What |
|------|------|
| `backend/jig_server/main.py` | FastAPI app, includes routers |
| `backend/jig_server/models.py` | Pydantic models: RunKind, RunStatus, Project, Run, RunMetric |
| `backend/jig_server/services/db.py` | SQLite schema + migrations |
| `backend/jig_server/services/runner.py` | Subprocess runner with SSE streaming |
| `backend/jig_server/routers/*.py` | 9 routers: projects, runs, specs, dataset, splits, training, eval, export, system |
| `scripts/specs.py` | Combinatorial spec sampler |
| `scripts/dataset.py` | Teacher-driven dataset gen, strict per-task validators |
| `scripts/qlora.py` | TRL+PEFT+bitsandbytes QLoRA training |
| `scripts/eval.py` | Functional eval against served Ollama |
| `scripts/split_dataset.py` | Train/valid/test split with seed=42 |
| `scripts/export_gguf.sh` | LoRA merge → GGUF → ollama create |
| `frontend/src/App.tsx` | 8-route shell with Sidebar; wizard stepper is the planned replacement |
| `frontend/src/api.ts` | Typed API client |
| `frontend/src/routes/` | Dashboard, Specs, Dataset, Examples, Training, Eval, Export, System |
| `RUN_TRAINING.md` | In orzo, the step-by-step edge-box command sequence |

## When you need details, follow these pointers

- Pipeline workflow details → `jig-workflow` (this skill)
- API surface → `jig-api`
- In-app agent behaviour → `jig-trainer`
- Training recipes + fit_check + pilot → `jig-recipes`
- Real-time edge-box status of the orzo project → `orzo-pipeline` skill
  + the cron watchdog `a60e90811bd1` ("orzo gen watchdog")