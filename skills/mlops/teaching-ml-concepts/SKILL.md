---
name: teaching-ml-concepts
description: Explain ML concepts when the user asks "what does X do."
---

# teaching-ml-concepts

Use this when the user asks a conceptual question about ML/AI — "what
does QLoRA do?", "how is DPO different from RLHF?", "explain attention
heads", etc. The goal is to teach, not just answer: connect theory to
their project, use concrete examples, and surface the implications
they'd care about.

## When to use

- "What does X do?"
- "How does X differ from Y?"
- "Explain X"
- "I don't understand X"
- "Why does X matter?"
- Any conceptual question that's not "run X for me" or "fix this bug"

## The teaching pattern

### 1. Lead with the one-sentence intuition

Before any jargon, give the answer in plain language.

❌ "QLoRA applies low-rank adapters to a 4-bit-normal-float-quantized
   base model with paged optimizer states."

✅ "QLoRA is a way to fine-tune a big model on small hardware: freeze
   the original weights, compress them to 4-bit to save memory, and
   train a tiny adapter on top."

If the user wants more depth, they ask. The first answer should always
be one sentence that lands the concept.

### 2. Concrete example, not abstract definition

Always pair the intuition with a worked example. Numbers beat
adjectives.

❌ "QLoRA is more memory-efficient than LoRA."

✅ "QLoRA lets you fine-tune a 7B model in ~5 GB of VRAM; LoRA needs
   ~16 GB. The cost is ~5-10% slower per training step."

### 3. Compare side-by-side when the user asks "differ from"

If the question is "how does X differ from Y," make a comparison
table. Don't prose-compare; users skim.

Example for QLoRA vs LoRA:

| Aspect          | LoRA                    | QLoRA                       |
|-----------------|-------------------------|-----------------------------|
| Base weights    | bf16 (~12 GB for 7B)    | 4-bit NF4 (~4 GB for 7B)    |
| Adapter         | Low-rank trainable      | Low-rank trainable           |
| VRAM (7B)       | ~16 GB                  | ~5–6 GB                     |
| Quality         | Baseline                | Within 1% on most tasks     |
| Speed           | Baseline                | 5-10% slower per step       |
| Edge-feasible 7B| No (won't fit)          | Yes (Orin Nano fits)        |

### 4. Connect to the user's project (the implicit ask)

After explaining, bridge to "what this means for what you're
building." One sentence is usually enough.

"For your 3B-on-Orin setup, QLoRA is the only option — full LoRA
needs more VRAM than the Orin has."

### 5. Cite the source (when it adds authority)

For non-obvious claims, point to the paper or doc:

"QLoRA was introduced in Dettmers et al. 2023
(https://arxiv.org/abs/2305.14314). The 'NF4' is a 4-bit data type
designed to be information-theoretically optimal for
normally-distributed weights."

Don't cite every sentence — only when it adds trust or context.

### 6. Offer the deep-dive (but don't force it)

End with an offer to go deeper:

"Want me to dig into why NF4 specifically, or how the LoRA adapters
attach to the quantized base?"

This respects the user's time and lets them pull on the thread they're
actually interested in.

## Patterns for common question shapes

### "What does X do?"

1. One-sentence intuition (the "elevator pitch")
2. Concrete example with numbers
3. Where it fits in the pipeline
4. Source citation if non-obvious
5. Offer to deep-dive

### "How does X differ from Y?"

1. One-sentence summary of the key difference
2. Side-by-side comparison table
3. Concrete example showing the difference in numbers
4. When you'd pick X vs Y
5. Offer to deep-dive

### "Explain X" / "I don't understand X"

1. Start with the simplest possible mental model
2. Build up: one layer of detail at a time
3. Use a concrete example end-to-end
4. Connect to a related concept they probably already know
5. Offer to go deeper

### "Why does X matter?"

1. The practical consequence (what breaks without it)
2. The typical scenario where it shows up
3. The cost of not caring (what you lose)
4. When it doesn't matter (the "don't worry about it" case)

### "Should I use X or Y?"

This is a decision question, not a pure teaching question. Use
`eval-rubric-design` and `finetune-recipe-picker` patterns:

1. State the decision criterion (VRAM? latency? quality?)
2. List the relevant differences on that criterion
3. Apply to the user's specific setup
4. Give a default + a fallback

## Anti-patterns

- **Definitions without intuition.** "QLoRA is a parameter-efficient
  fine-tuning method that uses 4-bit NormalFloat quantization." True
  but useless — doesn't tell the user why they'd care.
- **Jargon stacking.** "NF4 with double quantization and paged
  optimizers via bitsandbytes enables low-rank adaptation on sub-6GB
  VRAM footprints." Four concepts at once. Pick one to teach, then
  layer.
- **Academic-only framing.** "The information-theoretic motivation
  for NF4 is that quantized neural network weights follow a normal
  distribution..." Cool, but the user wanted to know when to use it.
- **Skipping the bridge to their project.** A perfectly correct
  explanation of LoRA that doesn't mention "for your Orin Nano setup,
  this is the only way" misses the implicit ask.
- **Going too long.** If the user asked a one-sentence question, a
  one-paragraph answer is right. If they ask for depth, then expand.
- **Made-up analogies.** "It's like a librarian who..." — only use
  analogies you're confident actually help. Bad analogies are worse
  than no analogy.
- **Inventing paper citations or numbers.** If you don't know the
  exact number or paper, say "I don't have the exact figure handy"
  or run an arxiv search. Don't make up stats.
- **Hedging everything.** "It kind of depends and there are trade-offs
  and..." teaches nothing. State the default, name the trade-off,
  move on.

## Style notes

- Plain text renderable in a terminal. The user is on CLI; markdown
  is fine but don't over-format.
- No emojis as bullet markers. Just `-` or `1.` for lists.
- Don't over-link. One paper link per concept max.
- Mirror the user's vocabulary. If they said "fine-tune" not "SFT",
  use "fine-tune".
- One concept at a time. If the question requires three concepts
  to answer, say "this touches three things: A, B, C. Let's start
  with A."

## When to load other skills

The teaching skill is meta. When teaching, draw from the domain
skills:

- `finetune-recipe-picker` — for "QLoRA vs LoRA vs full FT" questions
- `base-model-selection` — for "which model should I pick" questions
- `dataset-design` — for data pipeline questions
- `eval-rubric-design` — for "how do I measure if my fine-tune worked" questions
- `latency-as-rubric-gate` — for "why is my model slow" questions
- `training-loss-reading` — for "what does my loss curve mean" questions
- `quantize-and-validate` — for "why quantize" or "what quant" questions

If the user asks a question that needs deeper source material than the
skills provide, run an `arxiv` search to ground the answer. Don't
make up paper citations or stats from memory.

## Sources to ground answers in

- Dettmers et al. 2023 — QLoRA: https://arxiv.org/abs/2305.14314
- Hu et al. 2021 — LoRA: https://arxiv.org/abs/2106.09685
- Rasley et al. 2020 — DeepSpeed ZeRO: https://arxiv.org/abs/1910.02054
- Rafailov et al. 2023 — DPO: https://arxiv.org/abs/2305.18290
- Schulman et al. 2017 — PPO: https://arxiv.org/abs/1707.06347
- Guo et al. 2024 — DeepSeekMath (GRPO): https://arxiv.org/abs/2402.03300
- llama.cpp GitHub: https://github.com/ggml-org/llama.cpp
- HuggingFace PEFT docs: https://huggingface.co/docs/peft
- HuggingFace TRL docs: https://huggingface.co/docs/trl

## Verification: am I teaching well?

After writing the answer, check:

- [ ] Did I lead with one-sentence intuition?
- [ ] Did I include a concrete example with numbers?
- [ ] Did I bridge to the user's project?
- [ ] Did I cite a source for non-obvious claims?
- [ ] Did I offer the deep-dive (without forcing it)?
- [ ] Did I avoid jargon stacking and made-up analogies?
- [ ] Did I avoid invented citations or stats?

If any answer is no, revise before sending.
