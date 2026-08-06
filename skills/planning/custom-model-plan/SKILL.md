---
name: custom-model-plan
description: Plan a custom-trained AI model project from idea through serving. Use when the user wants to train, fine-tune, or otherwise customize a language model for a specific task on constrained or specific hardware.
---

# custom-model-plan

Plan a custom-trained AI model end-to-end. This is a *meta* skill: it teaches
how to scaffold a training project, gather the requirements, and produce a
defensible plan. It does not execute the training itself — for that, load
`axolotl`, `unsloth`, `fine-tuning-with-trl`, or `finetune-recipe-picker`
during the relevant phase.

The pattern in this profile is consistent: **the user knows the goal in
concrete terms but starts with the model, not the behavior**. This skill
flips that around. It forces the planner to specify the ideal-state behavior
and the system that produces it *before* selecting a base model, a recipe,
or a fine-tuning method.

## When to load

Load this skill when the user says any of:

- "I want to train a model to do X"
- "I want to fine-tune [model] for [task]"
- "I want a local [analyst / assistant / agent / tool] that can do X"
- "I have a Jetson / laptop / edge box and want to run a custom model"
- "Plan a fine-tuning project for [domain]"
- "How should I approach building a [domain-specific LLM]?"

Do NOT load this skill for: standard inference on existing models, dataset
design in isolation (load `dataset-design` instead), eval design in
isolation (load `eval-rubric-design` instead), or recipe selection in
isolation (load `finetune-recipe-picker` instead). This skill orchestrates
all four; it does not replace them.

## The principle

A trained model is one piece of a system, usually the smallest. The
deliverable is the *behavior* the user can observe at the terminal,
browser, or API. Plan from the behavior backward to the model, not
from the model forward to the behavior.

This means: do not start with "which base model?" or "what's the
training recipe?" Start with "what does the user see, hear, and trust
when the system works?" Then work backward.

## The plan structure (10 sections, fixed order)

Every custom-model plan written from this skill must have these 10
sections, in this order. The structure is non-negotiable because each
section blocks a real failure mode. Reordering or skipping sections
is a research-discipline violation, not a stylistic choice.

### 0. Scope and non-goals

The hardest section. Forces the planner to draw the line between what
the system does and what it does not do. A common failure is "scope
creep by optimism" — the plan says the system does X, Y, Z, and the
user believes the system does A, B, C, D, E, F. Lock this down first.

Questions to answer:
- What is the system? (One sentence.)
- What is the system NOT? (Multi-sentence list, explicit.)
- Who is the user? (Single user? Team? Public API?)
- What hardware? (Edge box, laptop, desktop, cloud — drives everything.)
- What is the failure mode if this project fails? (Honest answer.)

If the user has not specified the hardware, ask. If the user has not
specified the user persona, ask. If the user has not specified the
non-goals, surface a list and ask them to confirm.

### 1. Architecture

The system as components, not as a model. The model is one box in a
diagram. Common components that appear in most custom-model plans:

- Query planner (often the model itself)
- Retrieval layer (vector DB, BM25, hybrid)
- Structured data layer (SQL, knowledge graph, fact store)
- Tool layer (calculator, web search, code execution)
- Generation layer (the fine-tuned model)
- Verifier layer (rules + LLM-judge for claim↔evidence, arithmetic replay)
- Serving layer (vLLM, llama.cpp, Ollama)
- Application layer (CLI, web UI, API)

Draw the diagram. Identify which components are agnostic to the
domain vs which are domain-specific. The model is usually the most
domain-aware component and the least domain-specific in *training* —
it learns behavior, not facts. Facts live in retrieval and the fact
store.

### 2. Base model selection

Resolve the model choice before writing the rest of the plan. The
plan is wrong if the model is wrong; everything downstream depends on
this decision.

Selection criteria (in order of weight, for behavior-training tasks):

1. **Recency** — preference for the most recent release in the size
   class. Stale models waste your training compute.
2. **The specific capability the task needs** — math reasoning for
   financial ratios, tool-calling for agentic tasks, code generation
   for code tasks, multilingual for non-English tasks. Use the model's
   own published benchmark numbers, not downloads.
3. **License** — must permit fine-tuning, redistribution, and
   upload-back-to-HF. Apache-2.0, MIT, Llama 3.x (with caveats),
   and Qwen are fine. Custom enterprise licenses are usually not.
4. **Edge viability** — must QLoRA-fit on the target hardware at
   realistic context lengths. Verify with a 1-step fit probe at the
   start of Phase 4, not before.
5. **Context length** — must support the longest realistic prompt
   (RAG stuffing takes room; 4k is tight, 32k comfortable,
   128k+ luxurious).
6. **Community support** — GGUF quants, bug reports, edge
   optimizations. Low download count is a risk, not a verdict.

Do NOT use download count as the primary signal. The most-downloaded
model in a size class is rarely the freshest frontier — it's the most
battle-tested. Battle-tested is good for inference, not always good
for fine-tuning.

Read the actual model cards. The numbers in the README are the model
publisher's own claims; treat them as a starting hypothesis to verify
with a smoke test (Phase 0.5 below), not as ground truth.

Document the fallback chain. The primary model is rarely the model
that ships. Common reasons: OOM, license surprise, tool-call syntax
incompatibility. Plan for 2–3 fallbacks and the downshift conditions
that trigger each.

### 3. CLI UX / serving surface

The user-facing interface. The plan is wrong if the user can't
describe what the system looks like at 9am when they want to use it.

Specify:
- Entry point (CLI command, web URL, API endpoint)
- Latency budget per task class (with p50 and p95)
- Display policy (what the user sees for an answer, a citation, a
  refusal, an error)
- The commands the user types (real example session in the plan)
- The failure modes that surface to the user (e.g., "I can analyze
  X; I don't make buy/sell calls")

This section is where personal-grade vs production-grade becomes
visible. Personal tools can have relaxed latency (sub-10s is fine);
production tools need p95 < 2s for the most common task.

### 4. Personal-grade vs production-grade scope

Be explicit. The plan is wrong if it tries to be both.

Personal-grade (default for solo research, hobby, learning):
- One user, Tailscale-only, no auth, no rate limiting
- Local logs, no experiment tracking (or W&B only if user wants it)
- 2 seeds, not 3 (saves ~1/3 training wall-clock)
- Best-effort refinements, not statistical significance
- One-command reproducibility required; CI/CD not required

Production-grade:
- Multi-tenant, auth, rate limiting, audit trail
- Experiment tracking (W&B or MLflow), multi-seed (3+), statistical
  significance reported
- Latency SLAs, error budgets, on-call
- CI/CD for the model, dataset, and serving stack
- Versioned releases, rollback, canary

The two require different plans. Mixing them produces plans that
over-engineer personal tools and under-engineer production tools.
Pick one and commit.

### 5. Mandatory ablations (researcher discipline)

The pre-registration. Before claiming the fine-tune works, run the
mandatory ablations. The standard set:

1. Base model + oracle evidence (perfect context) — upper bound on
   what the model can do.
2. Base model + real retrieval — the deployment scenario baseline.
3. Fine-tuned + oracle evidence — does FT help even with perfect
   context?
4. Fine-tuned + real retrieval — the actual deployment scenario.
5. Fine-tuned + retrieval + domain tools (calculator, fact store) —
   does the system help?
6. Fine-tuned without domain tools — the model has to do all
   arithmetic from tokens.

Skip ablations only by explicit user choice, not by silent omission.
Document the choice and the trade-off.

### 6. Eval rubric

The most important artifact in the project. Everything else exists to
optimize against it. Per `eval-rubric-design` skill:

- Deterministic scorers first (regex, AST, JSON schema, exact match
  with tolerance)
- LLM-judge second (structured prompt, 5–15% noise floor)
- Human annotation third (gold standard for subjective tasks)
- Stratify by difficulty, by domain slice, by retrieval quality
- Pre-register the expected outcome per task type
- Hash + lock the test set BEFORE generating any training data
- The rubric sanity check (manual 20–30 example annotation) is
  mandatory and cheap; skipping it is the most expensive mistake

The test set must be frozen (SHA256, recorded in
`reproducibility.json`) before any training data is generated. Any
spec that ends up in `train.jsonl` invalidates the eval.

### 7. Open questions for the user

A short, numbered list of decisions the planner cannot make alone.
Each question must be concrete enough that the user can answer in
one sentence. Vague questions produce vague answers.

Common open questions in this class of project:
- Default domain scope (sector, language, region)
- Web search provider (Tavily, Serper, Brave, Exa, self-hosted)
- Teacher model and budget for dataset generation
- Test set authorship (hand-authored, teacher-generated,
  hand-reviewed, or borrowed from public benchmark)
- HF account username for upload target
- Hardware availability (the training box is sometimes the blocker)

### 8. Risks and how we detect them early

One bullet per risk, with the detection mechanism. Examples:
- "Test set drift: phase 0 SHA256 lock + contamination check in
  phase 3"
- "Rubric wrong: mandatory 20–30 manual sanity check at end of
  phase 2"
- "Base model doesn't behave as model card claims: Phase 0.5
  smoke-test on the laptop (5 prompts, all candidates)"
- "Hardware unavailable: hard blocker, mitigate by completing
  pre-hardware phases elsewhere first"

A risk without a detection mechanism is a hope. Convert hopes to
risks with detections.

### 9. Deliverables

What gets shipped. List the artifacts and where they live. Common
deliverables for a custom-model project:
- The trained adapter (LoRA) and the merged base
- The quantized GGUF for serving
- The retrieval index
- The fact store / structured data layer
- The training JSONL
- The frozen test JSONL
- The reproducibility manifest (every hash, every seed, every config)
- The eval report (per-task, per-slice, latency, ablations)
- A README that explains what the system does and how to run it

If the user wants HF uploads, list the target repos (model card,
dataset card). If the user wants a CLI, list the entry point and
the README that documents it.

### 11. How-to references (the bridge to execution)

The plan is not the code. This section is the bridge. It tells the
executor of each phase:

- Which skill to load (from the available skills list)
- What the code shape looks like (directory structure with file
  names)
- What the configs look like (concrete YAML, not "configure as needed")
- What the commands look like (real bash the executor will run)

This is the section the executor reads first when they start a
phase. If it's vague, the phase takes 2x as long because the executor
spends the first week making decisions that should have been in the
plan.

## The ideal-state interview (do this before writing the plan)

Before writing sections 0–11, conduct a structured interview. The
goal is to surface what the user actually wants vs what they say
they want. The two are often different. The plan is wrong if it
optimizes for what they said and the user actually wanted the other
thing.

Questions, in order:

1. **What does success look like at 9am on a Tuesday when you're
   using this?** (Forces a concrete user scenario. The answer
   reveals the real interface and the real latency budget.)
2. **What does failure look like?** (Forces the user to think about
   trust, refusal, and the cost of a wrong answer. Reveals the
   real non-goals.)
3. **What would you delete if you had to ship in 2 weeks?** (Forces
   prioritization. Reveals what the user actually values vs what
   they said they want.)
4. **What's the one thing this system must never do?** (Forces the
   user to think about the worst-case output. Often reveals
   safety, IP, or trust constraints they hadn't articulated.)
5. **What existing tool does this replace or augment?** (Reveals
   the user's mental model. The system should feel like a familiar
   tool with a specific delta, not a new paradigm.)
6. **Who else will use this, if anyone?** (Reveals whether the
   system is personal or shared. Drives all the production-grade
   decisions.)
7. **What is your compute budget — wall-clock, dollars, electricity,
   patience?** (Drives the recipe choice and the number of
   ablations.)
8. **What will you measure to know if it worked?** (Forces the user
   to think about evaluation before the system exists. Often
   reveals that they have no measurement plan, which is the
   #1 cause of "I trained for 8 hours and don't know if it worked.")

If the user can't answer any of these, surface the gap and write
the plan with placeholders. The placeholders become open questions
in §7.

## The plan output format

A plan written from this skill:

- Lives in `~/.hermes/plans/<project-name>.md` (or
  the project's own `runs/PLAN.md` if the user prefers)
- Uses the 10-section structure above (§0 through §11, with §10
  reserved for the changelog)
- Has a `Status:` header with version, what's resolved, and the date
- Has an `Out-of-band notes` section at the end as a changelog
  (v0.1, v0.2, v0.3, ...) so future iterations can see what changed
  and why
- Cross-references are explicit (`see §3` not "see above")
- Section numbering is stable across versions — renumbering on
  every edit breaks cross-references
- All configs are concrete, not abstract
- All commands are copy-pasteable, not "run the training command"
- The plan is short enough to read in 10 minutes; details that
  don't fit live in §11 as code/config examples

## Pitfalls

- **Starting with the model.** The user almost always opens with
  "I want to fine-tune X for Y." Resist the urge to pick the model
  first. The ideal-state interview surfaces constraints that change
  the model choice.
- **Skipping the non-goals.** A plan without explicit non-goals is
  not a plan, it's a wish list. The non-goals section is what
  the user reads when they're disappointed that the system can't
  do something; it sets the expectation upfront.
- **Hardcoding the domain.** Many custom-model projects hardcode
  the user's first domain ("this is a financial analyst"). A
  good plan designs the system to be domain-agnostic at the
  appropriate layers, with the user's domain as the first
  populated instance. The model is almost always domain-agnostic;
  the data pipeline and the eval are usually domain-specific.
- **Confusing "more data" with "better data."** A 5,000-example
  pristine dataset beats a 50,000-example noisy one. The dataset
  quality bar is the highest-leverage thing in the project.
- **Skipping the smoke test.** Phase 0.5 (laptop smoke test for the
  base model) is the cheapest pre-registration check available.
  Skipping it is the most expensive mistake you can make that
  doesn't cost wall-clock.
- **Picking W&B or experiment tracking by default.** For personal
  projects, local run logs are enough. Experiment tracking is a
  production tool. Match the tool to the user.
- **Three seeds by reflex.** Two seeds is enough for the
  10–20pp gains most custom-model projects are hunting. Three
  seeds costs ~50% more wall-clock for marginal statistical
  power. The extra seed is a production-grade reflex.
- **Writing the plan in markdown but the configs in prose.** A
  plan where the Axolotl config is "tune learning rate and LoRA
  rank" is not a plan. The config block must be paste-ready.
- **Forgetting the as-of-date semantics.** Most "the system
  says X" questions have a time dimension. The plan must specify
  how the system handles "what would you have said in March?"
  and "what does the latest report say?" without conflating them.
- **Forgetting the per-claim citation discipline.** A model that
  gives the right number without citing the source is
  indistinguishable from a model that gives the wrong number
  with confidence. Citation is a trust surface, not a nice-to-have.

## Worked example

The plan at `~/.hermes/plans/financial-research-analyst.md`
is the canonical example. It has:

- 10 sections (0, 1, 2, 3, 4, 5, 6, 7, 8, 9) plus 11 (how-to) and
  12 (changelog)
- 4 versions of changelog (v0.1, v0.2, v0.3, v0.4, v0.5)
- Concrete configs (Axolotl YAML, sector templates, eval commands)
- Frozen pre-registration with expected deltas per task
- Open questions in §7 with concrete defaults the user can override
- A model selection rationale grounded in published model card
  numbers, not download counts
- Sector-agnosticism at the appropriate layers, with the user's
  first sector as the populated default

When writing a new plan, use this as the template. The structure is
the lesson; the content is the project-specific part.

## Related skills

Load these during the relevant phase, not up front:

- `plan` (writing-plans) — for the first draft of the spec list
- `eval-rubric-design` — for the eval rubric section
- `dataset-design` — for the dataset generation pipeline
- `finetune-recipe-picker` — for the recipe decision at Phase 4
- `axolotl` / `unsloth` / `fine-tuning-with-trl` — for the actual
  training
- `serving-llms-vllm` / `llama-cpp` — for the serving layer
- `huggingface-hub` — for HF uploads
- `evaluating-llms-harness` — for standard benchmark sanity checks
- `training-studio-recipes` / `edge-qlora-pipeline` — for project-specific recipes
  if using a training studio or edge pipeline

Do not load all of these at the start. The plan is a guide, not a
manifest. Load skills when the phase demands them.
