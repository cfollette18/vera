---
name: finetune-recipe-picker
description: Pick a fine-tune recipe from hardware + model + data.
---

# finetune-recipe-picker — meta-skill (project-agnostic)

Choosing a fine-tuning recipe is mostly a VRAM + wall-clock + quality
trade-off. Use this skill when asked to: pick a recipe for a new
project, downshift a recipe that OOM'd, compare two recipes, or set
up a small hyperparameter sweep.

## The decision tree

```
Step 1: What hardware are we on?
  Edge box (≤ 8 GB unified, e.g. Jetson Orin Nano)
    → QLoRA (4-bit NF4 base) is the only feasible path for 1.5B+ models.
      QDoRA (4-bit base + DoRA) if you can afford ~5-10% more wall-clock
      and want best quality/MB.
  Laptop with 12-16 GB GPU
    → LoRA bf16 (full-precision base, LoRA adapters) fits 1.5-3B comfortably;
      QLoRA still works but quality is similar.
    → DoRA bf16 if hyperparams allow; better quality-per-rank.
  Desktop with 24 GB GPU (RTX 3090/4090)
    → LoRA / DoRA bf16 up to 13B.
    → Full bf16 SFT for ≤ 3B (laptop-class data sizes).
  Multi-GPU / cloud
    → Full SFT, distributed; QLoRA only if you specifically want it.

Step 2: What model size?
  0.5-1.5B → all recipes fit on edge.
  3-7B     → QLoRA on edge; LoRA bf16 on laptop.
  13B+     → QLoRA on desktop; full FT on cloud.

Step 3: How much data?
  < 1k examples → recipe matters less than hyperparams; do a small hparam sweep.
  1k-10k       → standard recipe works; lr ∈ {1e-4, 2e-4} is the band.
  10k-100k     → QLoRA with packing + neftune often wins.
  100k+        → consider curriculum learning or per-task mixing.

Step 4: What's the wall-clock budget?
  ≤ 1 hour    → pilot on 1-5% data, abort if not converging.
  1-8 hours   → full single-seed run.
  8+ hours    → multi-seed (≥ 3) + at least one ablation.
```

## Recipes (reference dataclass shapes)

```
qlora_nf4   base=4bit NF4, peft=LoRA r=16  alpha=32, dropout=0.05
            optim=paged_adamw_8bit, grad_ckpt=True, bf16 compute
            VRAM (1.5B): ~1.5 GB + adapter  →  fits 8 GB easily
            wall:        slowest of the LoRA family on edge

lora_bf16   base=bf16,   peft=LoRA r=16  alpha=32, dropout=0.05
            optim=adamw_torch_fused (no 8-bit on bf16)
            VRAM (1.5B): ~3 GB + adapter  →  tight on 8 GB

dora_bf16   base=bf16,   peft=DoRA r=16 (magnitude/direction decomp)
            VRAM similar to lora_bf16, ~5-10% slower per step
            Quality/rank is generally higher than LoRA at the same rank.

qdora_nf4   base=4bit NF4, peft=DoRA r=16
            Best quality/MB on edge. Slower than qlora_nf4.

full_bf16   base=bf16, no peft, full SFT
            VRAM (1.5B): ~12 GB → laptop only, NEVER on edge.
```

## Hparam sweep (when the recipe isn't the bottleneck)

If the recipe is settled but the result is mediocre, sweep on a 10%
data subset first:

```
lr ∈ {5e-5, 1e-4, 2e-4, 5e-4}    # 4 values
lora_r ∈ {8, 16, 32}             # 3 values
warmup_ratio ∈ {0.03, 0.1}       # 2 values
                                 # = 24 runs total; on a 1.5B on edge, ~5 min each
                                 # = 2 hours for the sweep
```

Pick the best by validation loss. THEN full-train with the best
hparams across ≥ 3 seeds.

## Common pitfalls

- **Higher rank ≠ better.** r=64 on 100 examples overfits; r=8 on
  100k examples underfits. Match rank to data size, not to model
  size.
- **LR too high on small data.** 5e-4 with 100 examples diverges
  fast. 5e-5 to 2e-4 is the safe band; if loss spikes in the first
  100 steps, halve LR and rerun.
- **Paged optimizer on bf16 doesn't help.** paged_adamw_8bit only
  matters when optimizer states dominate VRAM (i.e., 4-bit base).
  For bf16 base, use adamw_torch_fused.
- **Gradient checkpointing on small models.** For ≤ 1B, grad_ckpt
  adds 20-30% wall-clock with no real VRAM benefit. Only enable when
  the run OOMs.
- **Mixing DoRA + QLoRA is QDora, not DoRA.** Don't assume
  dora_bf16 hyperparams transfer to qdora_nf4.

## Verification before committing to a long run

1. **fit_check**: load base + recipe, run 1 forward+backward on 1
   example. If OOM, downgrade recipe (full → lora → qlora) and retry,
   max 3 downgrades. Exit 0 = fits.
2. **Pilot**: 1-5% data, ~100 steps. Abort on NaN, loss spike
   > 5x initial, or VRAM > 90% of box limit. Time the wall-clock
   per step; extrapolate to full run.

## When to load other skills

- For the actual training code: `axolotl` (YAML configs, broad
  recipe coverage), `unsloth` (2-5x faster on edge, single-GPU),
  `fine-tuning-with-trl` (TRL SFT/DPO/GRPO, programmatic).
- For DPO/GRPO alignment after SFT: same TRL skill.
- For project-specific recipes: load the project skill (e.g.
  `training-studio-recipes` for the training studio's 5 dataclasses).