---
name: training-loss-reading
description: Read SFT loss curves, spot divergence, compare seeds.
---

# training-loss-reading — meta-skill (project-agnostic)

Reading a loss curve well is the difference between "I trained for 8
hours and got something" and "I trained for 8 hours and got the best
result the data and recipe could produce." Use this skill when asked
to: debug a stuck or diverging training run, decide whether to stop
early, compare two runs, interpret the loss curve from a WandB
dashboard, or write up the training section of a RESULTS.md.

## What's actually in `trainer_state.json`

When you read loss data, you'll see (per logging step):

```
loss                  # the actual training loss (mean over grad_accum micro-batches)
grad_norm             # L2 norm of gradients (pre-clip)
learning_rate         # current lr (per scheduler step)
epoch                 # fractional epoch
eval_loss / eval_...  # eval-set metrics (if eval_strategy='steps')
_step                 # global step
```

TRL/Transformers logs these every `logging_steps` (default 5 in
training studio). The curve you see in the dashboard is `loss` (or `train_loss`)
plotted against `_step` (or `epoch`).

## What a "good" SFT loss curve looks like

For a healthy LoRA/QLoRA SFT run on a small LLM (1-3B) with ~2k-10k
examples:

```
   loss
  3.0 |  *  ← initial loss (depends on init + data; 2-4 is normal)
      | * *
  2.0 |     * * *
      |           * * *
  1.0 |                   * * * * *
      |                                  * * * * * * * * ← plateau
  0.5 |                                                  ___________
      +----------------------------------------------- steps -->
        0   100  200  500  1000  2000  5000
```

Three phases:

1. **Warmup (0 to ~3% of total steps):** loss may rise slightly or
   jitter. The lr scheduler is climbing from 0 to peak. Noise here is
   not informative.
2. **Fast descent (3% to ~30%):** loss drops steeply. This is where
   the model learns the easy parts of the data distribution.
3. **Fine-grained descent + plateau (30%+):** loss drops slowly,
   then plateaus. The plateau value tells you the irreducible loss
   for this data + recipe + model.

**Healthy end state**: smooth plateau with small per-step jitter
(±0.05 is normal on 2k examples; ±0.2 is normal on 200 examples).
The curve should look like a noisy asymptote, not a perfect line.

## What a "bad" SFT loss curve looks like (anomaly catalog)

Each entry: pattern → likely cause → fix.

### 1. NaN at any point

**Pattern:** loss goes from finite to NaN at a specific step.
**Cause:** exploding gradient; usually fp16 underflow or a single
bad example with extreme logits. Less common with bf16.
**Fix:** lower lr by 2x; switch to bf16 if you're on fp16; enable
`max_grad_norm=1.0` (usually default). If it persists, bisect the
data — the problem is one or a few examples. Check token IDs for
out-of-vocab that might explode the embedding gradient.

### 2. Loss spike (>5x the running mean) followed by recovery

**Pattern:** single-step spike, then returns to baseline.
**Cause:** usually one bad micro-batch (an outlier example, or
accumulation order effect when grad_accum > 1).
**Fix:** if it recovers within 50 steps and doesn't recur, ignore.
If it happens 3+ times in 1000 steps, your data has noise — review
the high-loss examples for content issues.

### 3. Persistent oscillation (loss doesn't converge)

**Pattern:** loss swings ±0.5 around a mean for hundreds of steps
without settling.
**Cause:** lr too high for this data; or grad_accum too low
(effective batch too small); or data is too noisy.
**Fix:** halve lr; double grad_accum; check `learning_rate` plot —
if it's not decreasing per the cosine schedule, the scheduler
isn't being applied (bug in the recipe dataclass).

### 4. Loss plateaus too early

**Pattern:** loss stops improving well before the data is
saturated; flat line from step N onwards.
**Cause:** lr too low; model capacity insufficient; data has a
ceiling the recipe can't break.
**Fix:** check the lr schedule — is the peak lr actually being
reached? If yes, the issue is data or capacity. Try lr × 3, or
lora_r × 2. If neither helps, the data may be unlearnable for
this base model.

### 5. Loss decreases on train but eval_loss increases (overfitting)

**Pattern:** classic overfitting curve. Train loss keeps dropping,
eval loss bottoms out then rises.
**Cause:** too many epochs, or lora_r too high for the dataset
size, or eval set too small to be a reliable signal.
**Fix:** stop at the eval_loss minimum (save the early checkpoint,
not the final one). If this happens within 1 epoch, the data is
too small for the recipe — reduce lora_r (e.g. r=16 → r=8) or
add regularization (dropout 0.1, neftune_noise_alpha 5).

### 6. Loss decreases on train AND eval but task perf doesn't

**Pattern:** "the loss says we're learning but the eval says we
aren't."
**Cause:** the loss is going down on tokens that don't matter for
the task (boilerplate, format tokens, common words). The model
has memorized the data distribution but not the behavior.
**Fix:** this is a **rubric / data problem**, not a training
problem. The loss will not diagnose it. Spot-check 20 generated
outputs from intermediate checkpoints. The eval rubric design is
likely misaligned with what the loss optimizes — see
`eval-rubric-design`.

### 7. Loss curve has a "step" (sudden drop or rise)

**Pattern:** loss is smooth, then takes a discrete step at one
specific step number.
**Cause:** usually a learning-rate schedule change (cosine
warmup boundary, or `lr_scheduler_type` change mid-run). Or: a
checkpoint was loaded and resumed from.
**Fix:** none, usually expected. Confirm by overlaying `lr` plot
on the loss plot — the step should align with an lr transition.

### 8. Loss is constant from step 0

**Pattern:** loss is a flat line at the initial value.
**Cause:** learning rate is effectively zero (typo, scheduler not
applied, or `requires_grad=False` on all params).
**Fix:** check that LoRA params have `requires_grad=True`
(`model.print_trainable_parameters()`); check `learning_rate`
in trainer_state — if it's 0, the scheduler is broken.

## Reading across seeds (the part most people skip)

Single-seed loss curves lie. The pattern researchers want:

1. Run ≥ 3 seeds with the same data + recipe.
2. Plot all loss curves on the same axes, with low alpha (e.g. 0.3).
3. Look for:
   - **Convergence**: do all seeds end at roughly the same loss
     value? Std > 0.1 across seeds at the end = unstable recipe.
   - **Trajectory shape**: are the descent phases similarly shaped
     or wildly different? Wildly different = data has high-variance
     examples that some seeds latch onto.
   - **Late divergence**: do seeds agree at step 1000 but diverge
     at step 5000? = overfitting is happening in some seeds and not
     others; the "best seed" is probably the one that didn't
     overfit, not the one with the lowest loss.

**Averaging loss across seeds is wrong.** Compute the *mean ± std*
of the final eval_loss, and the *median trajectory* with the
confidence band. Outliers are signal, not noise.

## Reading tegrastats alongside loss

On edge hardware (`$EDGE_HOST`), pair the loss plot with tegrastats:

```
tegrastats columns: RAM, CPU, GPU, POM_5V, thermal
```

What to look for:
- **RAM creeping up over the run** = memory leak or accumulating
  optimizer states. Restart before OOM. Most common cause: `lora_r`
  set too high (adapter states don't fit).
- **GPU utilization < 50% sustained** = data loader is the
  bottleneck. Increase `dataloader_num_workers` (default 0 in TRL).
- **Power dropping mid-run** = thermal throttling. Either reduce
  `max_steps`, lower `seq_len`, or improve cooling.
- **CPU pinned at 100%** = the teacher thread (if running) is
  starving the trainer. Move dataset gen to a separate box.

## Reading W&B alongside loss

If using Weights & Biases (`--wandb`):

- Group runs by `seed`, `lr`, `lora_r`, `data_version` for sane
  filtering. Tags > names for this.
- Always log: `train/loss`, `train/grad_norm`, `train/lr`,
  `eval/loss` (if eval_strategy='steps'), `eval/perplexity`
  (exp(loss) — easier to interpret than raw loss for a non-ML
  audience), `system/peak_vram_mb`, `system/peak_swap_mb`.
- Useful plots: `loss vs step` grouped by seed (facet), `lr vs
  step` (debugging the schedule), `grad_norm histogram` (per-step
  distribution tells you if you're clipping too much).

## What "good" looks like by recipe class

Different recipes have different healthy loss ranges. Don't compare
absolute loss values across recipes — compare trajectory shapes and
relative improvements.

| Recipe | Initial loss range | Final plateau range | Notes |
|--------|--------------------|---------------------|-------|
| `full_bf16` (≤3B) | 2.0-3.5 | 0.4-0.9 | Lower plateaus; risks overfit |
| `lora_bf16` | 2.5-4.0 | 0.7-1.2 | Higher plateau than full FT |
| `qlora_nf4` | 2.5-4.0 | 0.8-1.4 | Higher still; quantization noise |
| `dora_bf16` | 2.5-4.0 | 0.6-1.1 | Lower plateau than LoRA at same rank |
| `qdora_nf4` | 2.5-4.0 | 0.7-1.2 | Best 4-bit plateau |

If your plateau is way above these ranges, the data is hard OR the
recipe is wrong for the model. If way below, you may be overfitting
or the data has a degenerate structure (e.g. one repeated pattern).

## Decision tree: should I stop this run?

```
Is loss NaN?
  → Yes: stop. Fix lr/recipe/batch. Don't try to "recover" NaN.
  → No: continue.

Is the run still in the warmup phase (first 3% of steps)?
  → Yes: don't read anything into the loss yet.
  → No: continue.

Is loss still descending?
  → Yes: let it run. Don't kill a working run.
  → No: are we past 70% of total budget?
    → Yes: stop, save checkpoint, evaluate.
    → No: give it 200 more steps; if still flat, stop and check
          the rubric (could be a data problem, not a training one).

Has grad_norm exploded (>10× the running median)?
  → Yes: lower lr by 2x and restart from a saved checkpoint.
  → No: continue.

Is eval_loss > 1.1× its minimum so far?
  → Yes: this is overfitting. Save the minimum-eval checkpoint,
         stop training, evaluate that one.
  → No: continue.
```

## When to load other skills

- For writing up the training section of RESULTS.md: load
  `eval-rubric-design` for the framing; this skill for the loss-curve
  narrative.
- For project-specific training tooling (the training studio's training router):
  load the project skill.
- For ablations: a separate skill may be warranted if it doesn't
  exist yet; for now, see `eval-rubric-design` §"Failure-mode
  taxonomy" + this skill §"Reading across seeds".

## Reference: real examples

- The current training run (`$PROJECT_ROOT/`) — once training
  starts on `$EDGE_HOST`, `trainer_state.json` will be at
  `$PROJECT_ROOT/checkpoints/qwen-finetune25-coder-1.5b/`. Read it with this
  skill in mind.
- `tegrastats_log.sh` output for the training run — `runs/training-*.log`
  on `$EDGE_HOST`. The `training-loss-reading` checklist above is the
  way to interpret that log alongside `trainer_state.json`.