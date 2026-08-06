---
name: base-model-selection
description: Pick a base model for a fine-tune from HF model cards.
---

# base-model-selection — meta-skill (project-agnostic)

Picking a base model is upstream of recipe picking and downstream of
nothing. The wrong base wastes the entire fine-tune, the data
pipeline, the eval rubric, and the training time. This skill
covers the selection process only — the recipe (QLoRA vs LoRA,
rank, optimizer) is in `finetune-recipe-picker`; the training
execution is in `axolotl` / `unsloth` / `fine-tuning-with-trl`.

## Core principle

Match the **benchmark to the task**, not the benchmark to the
leaderboard. Then verify on your own stack before committing.

## Step 1: Identify the task profile

Before looking at models, list the *behaviors* the fine-tune needs
to support. Behaviors map to specific benchmarks:

| Task behavior | Benchmark to optimize for | Why not MMLU |
|---|---|---|
| Tool / function calling | **BFCL** (Berkeley Function Calling Leaderboard) | MMLU doesn't test tool syntax |
| Math / ratio / trend reasoning | **AIME25** or **GSM-Plus** | MMLU is multiple-choice factual recall, not step-by-step math |
| Code generation | **LiveCodeBench**, **HumanEval+** | MMLU doesn't test code generation |
| Long-context QA (RAG, multi-doc) | **RULER**, **LongBench**, **InfiniteBench** | MMLU fits in 1k context |
| Citation discipline | **IFEval** + manual probe | No public benchmark for citation |
| Instruction following (general) | **IFEval** | Acceptable proxy |
| Reasoning (general) | **GPQA-Diamond** | More discriminating than MMLU |
| Factual recall | **MMLU** (only here it's the right score) | This is what MMLU actually tests |

Multiple behaviors? Pick the top two benchmarks and weight them. A
fine-tune is usually trying to do 2–3 things well, not 8.

## Step 2: Pull candidates (working pattern)

Don't pick from a stale list. Use the HF API to enumerate, then
read the actual model cards to confirm.

```bash
# Most-downloaded in a size class (community support proxy)
curl -sS -m 15 \
  "https://huggingface.co/api/models?pipeline_tag=text-generation&sort=downloads&direction=-1&limit=100" \
  -o /tmp/hf_top.json

# Recently updated (freshness signal)
curl -sS -m 15 \
  "https://huggingface.co/api/models?pipeline_tag=text-generation&search=instruct&sort=lastModified&direction=-1&limit=200" \
  -o /tmp/hf_recent.json
```

Note: the API accepts `sort=downloads` and `sort=lastModified`, but
**NOT** `sort=created` (returns 400). To filter by release date,
sort by `lastModified` and cross-reference with the per-model
`createdAt` field from the metadata endpoint.

Filter the JSON for your size class (e.g. substring match on
`-3b`, `-1.5b`, `-mini`, `-nano`) and exclude derivative quants
(`-GGUF`, `-AWQ`, `-bnb-4bit`, etc.).

## Step 3: Read the actual model cards

For each surviving candidate, fetch the metadata + the raw README:

```bash
# Metadata: dates, downloads, likes, tags, library, license
curl -sS -m 15 "https://huggingface.co/api/models/<ORG>/<MODEL>" \
  -o /tmp/m.json

# The benchmark table is in the model card markdown
curl -sS -m 15 "https://huggingface.co/<ORG>/<MODEL>/raw/main/README.md" \
  -o /tmp/m.md

# Find the comparison table (each model card has slightly different layout)
grep -B2 -A30 "MMLU\|IFEval\|GPQA\|AIME\|BFCL\|HumanEval\|LiveCodeBench" /tmp/m.md | head -60
```

Read the column headers carefully. Many cards present
"GPT-4.1-nano vs Qwen3-30B vs Qwen3-4B vs Qwen3-4B-Instruct-2507"
and the column you care about is one of several. The `<u>` tags
in some Mistral-style cards mean "underlined = best in row."

## Step 4: Build the comparison table

For each candidate, capture at minimum:

| Field | Why |
|---|---|
| Release date | Recency signal (use `createdAt`) |
| Params | Size class |
| AIME25 / GSM-Plus | Math (if math task) |
| BFCL | Tool calling (if agent task) |
| IFEval | Instruction following (sanity) |
| GPQA-D | Reasoning (sanity) |
| Context length | Task fit |
| License | Redistribution |
| HF downloads | Community support proxy |
| HF likes | Quality sentiment (likes/dl > 1‰ = unusually positive) |

A worked example from Aug 2026 (1.5–3B, financial-reasoning
agent with tool use):

| Model | Released | Params | AIME25 | GPQA-D | IFEval | BFCL | Ctx | License | dl |
|---|---|---|---|---|---|---|---|---|---|
| mistralai/Ministral-3-3B-Instruct-2512 | 2025-10 | 3B | **72.1** | 53.4 | – | strong | 256k | Apache-2.0 | 477k |
| Qwen/Qwen3-4B-Instruct-2507 | 2025-08 | 4B | 47.4 | 62.0 | 83.4 | 61.9 | 256k | Apache-2.0 | 3.24M |
| HuggingFaceTB/SmolLM3-3B | 2025-07 | 3B | 17.1 | 44.4 | 76.7 | **95.0** | 128k | Apache-2.0 | 872k |
| Qwen/Qwen2.5-3B-Instruct | 2024-09 | 3B | – | – | – | – | 32k | Apache-2.0 | 5.95M |

Notice: download count is *inversely* correlated with recency here.
The most-downloaded model is from 2024. Don't pick by popularity.

## Step 5: Apply weighted criteria and write the rationale

Order your weighted criteria (from Step 1) and rank candidates:

For a financial-reasoning RAG agent in Aug 2026:
  1. **AIME25 (weight 5)** — math reasoning is the #1 failure mode
     for ratio/trend tasks. Ministral wins (72.1 vs 47.4 vs 17.1).
  2. **BFCL (weight 4)** — tool calling is essential. SmolLM3
     leads (95.0), Qwen3-4B is 61.9, Ministral strong (not published
     but model card demonstrates it).
  3. **Context length (weight 3)** — RAG + cross-company compare
     needs ≥32k. Ministral and Qwen3-4B both have 256k.
  4. **Recency (weight 2)** — freshest wins on the other metrics.
  5. **License (weight 2)** — Apache-2.0 for upload-back-to-HF.

Pick: **Ministral-3-3B-Instruct-2512** (AIME25 dominates, 3B
parameters, 256k context, Apache-2.0, recent). SmolLM3-3B as
fallback for better tool-calling.

Lock the pick with a one-paragraph written rationale. Save it in
the plan or in `runs/<project>/model_selection.md`. The user
will ask "why are we using this model?" three months from now
and the answer is in this paragraph.

## Step 6: Document the fallback chain (mandatory)

The fit probe may fail; the model card may lie; the tool-call
syntax may be a mess. Always document:

```
primary:  best by criteria
fallback 1: best by criteria + smaller or larger size
fallback 2: best tool-calling or math, whichever you need more
last resort: most-downloaded model in the size class
            (community support will save you when the others don't fit)
```

For the example above:
  - primary:  Ministral-3-3B-Instruct-2512
  - fallback 1: Qwen3-4B-Instruct-2507 (larger, if user accepts 4B)
  - fallback 2: SmolLM3-3B (better tool-calling, weaker math)
  - last resort: Qwen2.5-1.5B-Instruct (always fits, 14M dl support)

## Step 7: Laptop smoke test (BEFORE generating data)

Before committing a multi-day data pipeline, run a 5-prompt smoke
battery on each candidate on the x86 laptop (no GPU, CPU
inference is fine):

  1. **Tool-call syntax.** "Use the `<tool>` tool to look up X."
     Verify parseable call in the documented format.
  2. **Citation discipline.** "Cite the source of any number."
     Verify cited output in the documented format.
  3. **Arithmetic from a table.** Feed a 3-row table, ask for a
     ratio. Verify correct computation + shown work.
  4. **Refusal.** Ask for a fact that doesn't exist. Verify clean
     refusal, not invention.
  5. **Multi-turn tool follow-up.** After a tool result, verify
     the model consumes it and synthesizes a cited answer.

If the primary fails ≥2 of 5 (especially tool-call or refusal),
drop to the next fallback. Cost: ~30 min. Catches: model card
lying, chat template breakage, tool-call syntax bugs. Skipping
this costs 8+ hours of training against a wrong model.

Record results in `runs/<project>/phase_0_5_smoke/results.md` and
in `reproducibility.json` (locked model ID, SHA of chat template,
SHA of smoke-test prompts).

## Common pitfalls

- **Picking by download count.** Qwen2.5-1.5B is the most-downloaded
  1.5–3B model in 2026 and was released Sept 2024. It is robust,
  not frontier. Downloads = community support, not quality.
- **Optimizing for MMLU on a tool-calling task.** MMLU tests
  multiple-choice factual recall. It tells you almost nothing
  about whether the model can emit a parseable tool call. Pick
  the benchmark that matches the behavior.
- **Picking a model with no fallback chain.** The fit probe may
  fail. Document the fallback BEFORE you start, not after the OOM.
- **Trusting the model card.** Cards overstate. Tool-call syntax
  varies by model. The 30-min smoke test catches what the card
  hides.
- **Picking a model with a restrictive license when you need to
  upload.** Llama license has acceptable use + 700M MAU threshold.
  Granite is enterprise. Qwen/Mistral/SmolLM are Apache-2.0 and
  re-upload-safe.
- **Picking a 2024 model because the community loves it.** Frontier
  moves. A 2024 leader is a 2026 also-ran. Re-survey every 6 months.
- **Picking without checking the chat template exists.** Some
  newer fine-tunes ship without `chat_template.jinja` or with
  broken `tokenizer_config.json`. Verify before training.

## When to load other skills

- For the recipe (QLoRA vs LoRA, rank, optimizer, VRAM estimates):
  `finetune-recipe-picker`.
- For HF API + uploads: `huggingface-hub` (the `hf` CLI).
- For the actual training run: `axolotl`, `unsloth`, or
  `fine-tuning-with-trl`.
- For eval and rubric design: `eval-rubric-design`.
