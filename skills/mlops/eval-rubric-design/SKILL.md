---
name: eval-rubric-design
description: Design rubric, scorer, pre-registered success criterion.
---

# eval-rubric-design — meta-skill (project-agnostic)

The rubric is the most important artifact in a fine-tuning project.
Everything else exists to optimize against it. Use this skill when
asked to: design an eval for a new task, pick a scorer type, debug
an eval that's giving misleading numbers, set up a pre-registered
success criterion, or add a human-eval hook.

## The rubric has 4 jobs

1. **Measure the right thing.** A good rubric captures the
   behavior you actually want; a bad rubric captures something
   correlated with it but not the same.
2. **Be deterministic enough to compare.** Two fine-tune runs on
   the same data must produce comparable rubric scores, modulo
   known non-determinism.
3. **Be discriminative.** It must separate good outputs from bad
   ones — and confident-correct from confident-wrong. If everything
   passes, the rubric is too lenient. If everything fails, the
   task is out of scope.
4. **Be cheap enough to run at scale.** You'll run it on base, on
   every seed, on every ablation, on every OOD slice. A 30-second
   per-example scorer caps you at ~3000 examples per hour.

## Scorer types (cheapest first)

| Type | Speed | Determinism | Use when |
|------|-------|-------------|----------|
| Regex match | ms | perfect | Output must contain specific tokens / patterns |
| AST parse + structural check | ms | perfect | Output is code; check it parses + has required symbols |
| JSON schema validation | ms | perfect | Output is structured; check shape |
| Reference comparison (BLEU, exact match) | ms-s | perfect | Output is short and exact-form |
| LLM-as-judge with structured prompt | 1-5 s | noisy | Output is open-ended; reference answer unknown |
| Human annotation | minutes | subjective | Final gold standard; small N; reference set |

**Default rule**: deterministic scorers first, LLM-as-judge second,
human annotation third. Most fine-tunes can be evaluated end-to-end
with regex/AST/JSON-schema. Reserve LLM-judge for tasks where the
output is genuinely open-ended (creative writing, ambiguous
correctness).

## The rubric sanity check (before training)

Before you trust any post-training number:

1. Run the rubric against 20-30 *base model* outputs.
2. **Manually annotate** whether the rubric's pass/fail matches
   your judgment.
3. If disagreements > 20%, fix the rubric, not the model.
   Researchers spend 30 min on this; engineers skip it; the
   cost of skipping is "we trained for 8 hours and the rubric
   was wrong the whole time."

This is the step most easily skipped and most expensive when
skipped.

## Pre-register the success criterion

After the base eval, write down (in a free-form document or
structured YAML):

```
task: harness_scaffold
baseline_pass_rate: 0.05        # measured
baseline_latency_p95_ms: 4200   # measured
expected_finetune_pass_rate: 0.40   # hypothesis
expected_delta: +0.35
uncertainty: ±0.10              # ±1 std across seeds, expected
failure_mode_hypothesis: capability_gap | format_gap | robustness_gap | generalization_gap
```

Hash this and lock it. After fine-tune, compare measured vs expected.
**Flag surprises in both directions** — fine-tune *exceeding*
expectation is as interesting as under-performing (it suggests the
data was easier than you thought, which may be a generalization
concern for harder unseen inputs).

## Failure-mode taxonomy

Use this to bucket regressions and design remediation:

| Bucket | Symptom | Likely cause | Remediation |
|--------|---------|--------------|-------------|
| Capability gap | Base = 0% on task | OOD for base | Reconsider scope OR pick stronger base |
| Format gap | Base has skill, won't follow format | Format unfamiliar | Few-shot examples in rubric; more epochs |
| Robustness gap | Easy fine, hard fails | Distribution mismatch | Stratified training; upweight hard |
| Generalization gap | Train-domain ↑, OOD ↓ | Memorization | More diverse data; smaller LoRA r; regularization |

Bucket failures by combining the baseline pass-rate per input
difficulty stratum with a quick LLM-judge pass over 20 outputs.

## Common pitfalls

- **LLM-judge as the only scorer.** It works for open-ended tasks but
  has 5-15% noise floor; you can't measure a 3% improvement against
  it. Always have a deterministic sub-scorer for the binary structural
  parts.
- **Perplexity as the eval metric.** Perplexity on held-out data is
  not task performance. Always have a functional eval.
- **Same rubric for train and eval.** If the rubric is used to
  filter training data, then to score the trained model, the model
  is being graded on the same metric it was trained against. Use
  the rubric for training-time filtering only if it's not the eval
  rubric; use a different one for eval (or use train-domain data
  for filtering and test-domain data for eval).
- **No latency budget.** A fine-tune that improves pass rate by
  10% but doubles latency may be a regression. Capture latency
  during baseline eval and report it alongside pass rate.
- **Few-shot in rubric but not in fine-tune.** If the rubric
  includes 3-shot examples to coax the base model, the fine-tune
  may not learn that prompt structure. Either include the same
  shots in the fine-tune data, or remove them from the rubric.
- **Single-pass eval.** The base model output for a spec has
  temperature variance. Run 3 samples per spec with temp 0.7; report
  pass@k (any of 3 passes) AND pass@1 (greedy). The gap tells you
  about the model's confidence vs competence.

## Verification: pre-registration discipline

A researcher's checklist before committing to a multi-hour training
run:

- [ ] Rubric scorers all pass the manual 20-example sanity check
- [ ] Base model scored on full test set; per-task + per-stratum
- [ ] Expected pass rate + delta + uncertainty written down
- [ ] Hash of expected-outcome document recorded in
      `reproducibility.json`
- [ ] OOD test set identified (if any); same SHA256-frozen discipline
- [ ] Statistical power: "for ±3% delta at p<0.05, need ≥N specs per
      task" — verified the test set is large enough
- [ ] Latency + token cost captured at baseline; budget for
      fine-tune eval

If any of these is missing, surface it to the user before proceeding.

## Reference: real examples in this profile

- **jig's eval router** (`/home/cfollette18/jig/backend/jig_server/
  routers/eval.py`) — functional eval against a served Ollama
  model; uses deterministic scorers in `scripts/eval.py`.
- **orzo's eval** (`/home/cfollette18/orzo/eval/run_eval.py`) —
  compiles the generated harness, runs it against mock tools,
  scores on dispatch correctness.
- The researcher-flavored wizard refactoring plan
  (`/home/cfollette18/.hermes/plans/jig-researcher-reframing.md`)
  captures the gaps a researcher would add to the current eval step.

## When to load other skills

- For lm-eval-harness (standard benchmarks like IFEval, MMLU,
  HumanEval): `evaluating-llms-harness`.
- For W&B experiment tracking the rubric + baseline + fine-tune:
  `weights-and-biases`.
- For the actual eval routing in a project: load the project skill.