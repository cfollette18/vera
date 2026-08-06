---
name: dataset-design
description: SFT dataset design workflow from spec to pristine JSONL.
---

# dataset-design — meta-skill (project-agnostic)

Designing a fine-tuning dataset well is the highest-leverage thing you
do in a training project. The training recipe, the eval rubric, and
the model choice are downstream of the data. Use this skill when
asked to: design a dataset for a new domain, design synthetic-data
prompts for a teacher, validate an existing JSONL, debug a generation
pipeline that's producing low yield, or set up a reproducible
data-generation workflow.

## The dataset-design loop

1. **Define the input distribution** — what's the space of inputs
   the model will see at inference? A *spec generator* (deterministic,
   seeded, combinatorial) is the cleanest source. Output: a
   `specs.jsonl` with `{id, domain, input_text, tools, constraints,
   persona, ...}` records.
2. **Define the target output distribution** — per task type, what
   does a good answer look like? Encode as: (a) a JSON schema for
   structural validation, (b) a Python validator for content
   validation (ast.parse + import allowlist + identifier presence),
   (c) a quality scorer for "valid but bad" detection. Three layers
   are defense in depth; one is fragile.
3. **Freeze a held-out test set BEFORE generating any training data.**
   SHA256 the test specs. Any spec that ends up in `train.jsonl`
   invalidates the eval. Re-freezing is acceptable; silent test-set
   drift is not.
4. **Generate the dataset.** Teacher is any OpenAI-compatible API.
   `response_format={"type": "json_schema", ...}` per task where the
   teacher supports it. Per-call retry on validation failure (1
   retry with corrective follow-up is usually enough; 3+ retries
   indicate a structural problem with the prompt). Resumable: spec_id
   set in the output JSONL is the source of truth.
5. **Hygiene pass.** Run, in order:
   a. **Schema validation** (rejects structurally wrong outputs)
   b. **Content validation** (Python validators)
   c. **Quality scoring** (length bounds, identifier presence, code
      structure) → drops below threshold go to `_quality_drop/`
   d. **Dedup** (MinHash n-gram, 128 perms, threshold 0.85 Jaccard)
      → drops go to `_dedup_drop/`
   e. **PII / safety scrub** (regex for emails/phones/AWS keys/GitHub
      PATs, keyword list for toxicity) → drops go to `_pii_drop/`
   f. **Contamination check** (MinHash n-gram, threshold 0.5 Jaccard
      between train examples and test specs) → contaminated test
      specs moved to `_contaminated/`, replaced from held-out pool
6. **Stratify the test set** by difficulty (easy / medium / hard —
   by teacher perplexity at generation time, or by a separate
   difficulty rubric). Without stratification, post-training reports
   hide the "fine-tune helps easy, hurts hard" failure mode.
7. **Split** into train/valid/test. Record SHA256 of each split.
8. **Pre-register the expected outcome** per task. Hash + lock.
   This is the artifact that makes the eval meaningful later.

## Common failure modes

- **Validation too lenient.** A 95%+ pass rate means the validator
  isn't testing what you think. Add a held-out 50-example
  manually-labeled set, run the validator against it, measure
  precision/recall. If you can't build that set in 30 min, your
  validator is checking the wrong thing.
- **Validator + teacher in a race.** Validator gets more strict →
  teacher looks worse → teacher gets retuned → validator gets more
  strict. End state: every example passes but coverage drops. Pull
  back the validator to essentials; track coverage separately.
- **Dedup before quality score.** Dedup is cheap; quality score is
  cheap; the order doesn't matter much, but if you dedup first you
  may keep duplicates of high-quality examples and drop unique
  low-quality ones. Quality-score first.
- **Contamination on common boilerplate.** Test specs often share
  boilerplate ("You are orzo, a generator of..."). Strip the system
  prompt + user prompt headers before MinHash; otherwise the
  contamination rate is artificially inflated.
- **Generation race with retries.** If a single spec hits the retry
  cap, the *cost* is the same as the success case (you still paid
  for 2-3 teacher calls). Set a per-spec token-budget cap, not just
  a per-run cap.
- **PII scrub on the wrong axis.** Scrubbing user-visible PII in
  outputs is necessary; scrubbing PII in the input spec is usually
  *wrong* (the model needs to learn how to handle real PII it might
  see, or you're teaching it that PII doesn't exist).
- **Stratification by teacher perplexity at gen-time ≠
  stratification by task difficulty.** Teacher perplexity measures
  how hard the *teacher* found the input, not how hard the
  *student* will. Use a separate difficulty rubric if you care
  about the latter.

## Reference: real examples in this profile

- **orzo** (`/home/cfollette18/orzo`) — 7 task types, spec generator
  in `data/gen_specs.py`, validators in `data/gen_dataset.py`,
  splits in `scripts/split_dataset.py`. The README
  (`orzo/data/README.md`) is a worked example of the validator +
  per-task share design.
- **jig** — generic version of the same pipeline + dedup +
  contamination check + JSON-schema enforcement at the teacher.
  See `/home/cfollette18/.hermes/plans/jig-researcher-reframing.md`
  for the failure-mode taxonomy overlay.

## When to load other skills

- For the actual training step: `axolotl`, `unsloth`, or
  `fine-tuning-with-trl` depending on the recipe.
- For publishing the dataset: `huggingface-hub`.
- For project-specific tooling (jig's wizard, orzo's dashboard):
  load the project-specific skill instead.