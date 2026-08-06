---
name: jig-recipes
description: jig's 5 training recipes + fit_check and pilot mechanics.
---

# jig-recipes — optimization strategies (plan §6, §14 P1.14-17)

Recipes are preset dataclasses in `scripts/recipes.py`. They're
chosen at wizard step 10 (Recipe). The CLI flag is `--recipe NAME`
on `scripts/qlora.py`. Five recipes, ordered heaviest to lightest:

| key | base dtype | peft | VRAM est (1.5B) | Notes |
|-----|------------|------|------------------|-------|
| `full_bf16` | bf16 | none | ~12 GB | **Laptop-only.** Disabled if RAM < 16 GB or VRAM < 12 GB. No LoRA. |
| `lora_bf16` | bf16 | LoRA r=16 | ~3 GB weights + adapter | Fallback when bitsandbytes aarch64 fails. |
| `dora_bf16` | bf16 | DoRA r=16 | ~3 GB + adapter | Magnitude/direction decomposition; better quality/rank. `peft.LoraConfig(use_dora=True)`. |
| `qlora_nf4` | 4-bit NF4 | LoRA r=16 | ~1.5 GB + adapter | **Default.** Paged 8-bit Adam + grad checkpointing + bf16 compute. |
| `qdora_nf4` | 4-bit NF4 | DoRA r=16 | ~1.5 GB + adapter | Best quality/MB on edge. |

Common hyperparams across LoRA recipes: `lora_alpha=32`, `lora_dropout=0.05`, `target_modules="all-linear"`, `optim=paged_adamw_8bit` (or `adamw_torch_fused` for non-quantized), `lr_scheduler=cosine`, `warmup_ratio=0.03`, `bf16=True`, `gradient_checkpointing=True` (when supported), `max_grad_norm=1.0`.

## Advanced knobs (behind "Show advanced")

`seq_len`, `batch`, `grad_accum`, `lr`, `epochs`, `lora_r`, `lora_alpha`, `lora_dropout`, `target_modules`, `packing`, `neftune_noise_alpha`, `max_grad_norm`, `optim`, `lr_scheduler_type`, `eval_steps`, `save_steps`, `warmup_ratio`, `bf16/fp16`.

## fit_check.py (P1.15)

`scripts/fit_check.py` — 10-second OOM probe. Loads the base model
with the chosen recipe, runs a single forward+backward pass on 1
example, measures peak memory.

- Exit 0 = fits, proceed.
- Exit 1 = OOM even after 3 downgrades. Surface to user; they need
  a smaller base model.
- Exit 2 = other error (transformers/peft version mismatch, etc.)

Downgrade order on OOM: `full_bf16 → lora_bf16 → qlora_nf4`. DoRA
recipes downgrade to their non-DoRA equivalent.

## pilot.py (P1.16) / `--pilot` flag

Runs on 1-5% of data for ~100 steps as a smoke test before the full
run. Captures: loss curve, time/step, peak VRAM/SWAP. **Aborts** if:
- NaN loss appears
- Loss spikes > 5x initial value
- VRAM approaches box limit (≥ 90%)

On abort: clear error + agent offer to retry at LR/2. The user can
also choose to downgrade the recipe.

The current `scripts/qlora.py` doesn't have `--pilot` yet — that's
P1.16 work. Until it lands, run a manual pilot:
```
python scripts/qlora.py --data <pilot.jsonl> --output runs/pilot \
    --epochs 0.1 --lr 2e-4 --batch 1 --grad-accum 4
```

## Multi-seed (plan §4.7, ships with multi-seed in Phase 3)

`--seed-list` flag (default `[42]`, "publication-ready" badge
requires ≥ 3). Training loops N times serially with `seed = seed[i]`,
producing N checkpoint dirs `checkpoints/run/seed-{i}/`. Eval reports
per-seed mean ± std.

Storage: 3 seeds × 1.5B base × QLoRA adapter ≈ 3 × 30 MB = 90 MB.
Fine on the 8 GB box. For 7B with bf16 adapters, 3 × 200 MB =
600 MB — flagged in UI before commit.

## Reproducibility knobs

Inside each recipe: `seed`, `deterministic_algorithms: True`,
`torch.use_deterministic_algorithms(True)`. Documented: CUDA
reductions + FlashAttention varlen are non-deterministic even with
deterministic mode on. The `reproducibility.json` manifest records
which knobs were on.

## Choosing a recipe for a new project

Quick heuristic:
- 8 GB unified (Jetson Orin Nano) → `qlora_nf4` or `qdora_nf4`.
- 16-24 GB GPU laptop → `lora_bf16` or `dora_bf16` (faster, slightly
  more VRAM).
- 24 GB+ dedicated GPU → `full_bf16` if data is small (< 10k
  examples), else `dora_bf16`.
- bitsandbytes fails to import on aarch64 (rare now, was common
  pre-2024) → `lora_bf16`.

Always run `fit_check` before committing to a multi-hour training run.
Always run a pilot (or 100-step `qlora.py` smoke) to catch divergence
before burning the full data budget.

## Reference

Full detail in `/home/cfollette18/.hermes/plans/jig-reproducible-
app.md` §6 (optimization strategies), §4.7 (multi-seed), §4.8
(pilot), §4.10 (reproducibility), §14 Group E (P1.14-17 tasks).