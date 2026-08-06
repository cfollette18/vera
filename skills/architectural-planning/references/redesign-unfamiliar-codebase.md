# Architectural Planning Over Unfamiliar Multi-Repo Codebases

Use this reference when the user asks for a plan or "happy path" for a system **they want redesigned or turned into a product**, but no code is being written yet. This is the design-phase sibling of bite-sized TDD planning — the deliverable is an architectural blueprint plus open decisions, not a numbered task list with copy-pasteable code.

## When to reach for this reference

The request sounds like one of these:
  - "plan out the full end-to-end workflow and happy path for X"
  - "make Y a replicable / shippable / turnkey application"
  - "what do we need to make Z work for a real user?"
  - "audit X and tell me how it should evolve"
  - "I have these repos (A, B, C) — figure out the architecture for the goal I'm about to describe"

Signals it's **this** and not bite-sized implementation planning:
  - No feature ticket with acceptance criteria yet
  - Multiple repos / projects interact (front-end + back-end + worker scripts + external service like a teacher LLM)
  - The user is choosing the destination (settings UI shape, agent-in-the-loop, etc.) — not you
  - Constraints are hardware, cost, or operational ("8 GB VRAM", "must run on Jetson", "no cloud budget") rather than functional

If those signals are present, **do NOT proceed straight to the bite-sized TDD structure** — it will push you toward 2–5 minute tasks and you'll burn the plan on micro-implementation detail before the design is decided.

## Exploration sequence (read-only, before writing)

This is the order that consistently produces plans that survive contact with reality. Parallelize aggressively where the calls are independent.

1. **Locate the repos.** A single `find ~ -maxdepth 4 -type d -name <repo>` plus two `search_files(pattern=<repo>, target=files)` calls usually nails it in one batch.
2. **READMEs first, both repos.** They encode the author's mental model and the curated pitch. Skim the stack table, the architecture diagram, the quick-start, the CLI fallback section.
3. **Tree dump with extensions filter.** One `terminal find` with `\( -name "*.py" -o -name "*.tsx" ... \)` and `-not -path "*/node_modules/*"`. This tells you the actual shape (FastAPI? Django? Express?) before you read a single source file.
4. **Entry points.** Backend `main.py`, frontend `App.tsx` + router + sidebar, every `scripts/*.py`. These are the highest-information-per-line files in the project.
5. **The data model.** Pydantic models / SQLAlchemy / DB schema / Pydantic-Settings. Where state lives, what types flow through.
6. **The "interesting" scripts.** In the training studio's case: `dataset.py` (the validation philosophy), `qlora.py` (the optimization choices), `eval.py` (what counts as success), `export_gguf.sh` (how the artifact is born).
7. **The UI surface.** One representative route per category (data table, form, monitor). Don't read all routes — pick a `Dataset.tsx`-style one and an `Eval.tsx`-style one to learn the conventions.

**Stop reading once you can answer these four questions:**
  - What does a successful end-to-end user run look like today?
  - What is the gap between today and the user's stated goal?
  - What is the smallest piece of new code that would close the largest gap?
  - What decision is the user going to need to make, where my default would be wrong?

If you can't answer (4), keep exploring or ask.

## Plan structure that worked here

For architectural / redesign plans, use this outline (not the bite-sized TDD structure):

1. **What the system is today** — concrete inventory of existing pieces, one paragraph per major area. Naming real files. This is your contract with the reader: I read these things, here's what I found.
2. **The happy path** — numbered user-facing steps from "cold start" to "goal achieved." The user's words ("replicable application", "train my own models") translated into concrete actions with file paths and commands.
3. **UX shape** — wireframe in prose. "Top-aligned horizontal stepper; locked tabs are CSS-blurred with a lock badge and tooltip showing the unlock criterion." The user can argue with prose; they can't argue with a Figma they can't see.
4. **Critical guarantee sections** — anything the user emphasized ("pristine and strict JSON schema", "constrained hardware", "autonomous agent"). Give each its own subsection that names the mechanism, not just the desire. "JSON-schema enforcement at the teacher via `response_format=json_schema`, not just post-hoc Python validation" beats "the dataset will be clean."
5. **New + changed files** — split into Backend / Frontend / Scripts / Skills. Each entry: NEW or CHANGE, absolute path, one-line purpose. This is the artifact the implementer actually scans for.
6. **Phased implementation plan** — 3–5 phases, each independently shippable, with a time estimate. This converts the architectural plan into something a project manager can put on a calendar.
7. **Verification** — what "done" means in measurable terms. For dataset: a `validate_dataset.py --strict` command returns 0. For the wizard: complete step 1, confirm steps 2–10 are blurred. For the agent sandbox: try `os.system("rm -rf /")`, confirm the allowlist blocks it.
8. **Open decisions** — questions you can't answer without the user. **Always end with this.** Phrased so the user can answer in one line each. Without this section you ship a plan that secretly encodes your guesses.

## Anti-patterns to avoid

  - **Don't write code in the plan.** Not even "obvious" snippets. Once there's code in the plan, every implementer rewrites it to their taste anyway, and the plan becomes a half-finished implementation. The one exception is JSON-schema examples for external contracts — those are documentation, not implementation.
  - **Don't propose a rewrite.** If the user has a working pipeline, the plan should be additive. Name every NEW file and every CHANGE to existing files. If you're tempted to say "replace the runner," you're probably wrong.
  - **Don't hide open decisions.** If the encryption passphrase path, the quarantine threshold, and the agent default-on-or-off are real choices, list them. Pretending you know the answer is how plans fail in week three.
  - **Don't skip the "what is today" inventory.** It's tempting to jump to the redesign. But the user (and you, when context grows) need a baseline to measure the gap against.
  - **Don't bundle independent decisions into one phase.** "Phase 1: settings store + recipe picker + wizard UI + agent" is not a phase, it's a quarter. Each phase should be independently shippable and reversible.

## Output conventions

  - Save under `.hermes/plans/<YYYY-MM-DD_HHMMSS>-<slug>.md`. Use a descriptive slug, not the project name — `reproducible-training-studio.md` not `jig.md`. Slugs let multiple plans coexist in the same dir.
  - In the chat reply, lead with the path to the saved plan, then a short "what's in it" summary (5–10 bullets max), then the questions / next-action options. The user came for the plan, not the chat.
  - Don't quote large sections of the plan in chat. They have it on disk.

## Decision record template (use inside the plan)

For each architectural choice where multiple defensible options exist, include a one-paragraph "Decision" subsection with:
  - **Options considered** (2–4, named)
  - **Default I'm picking and why**
  - **Why the user might override**

Example:
> **Decision: Wizard stepper state persistence.** Options: (a) React component state, (b) localStorage, (c) SQLite JSON column. **Default: (c)** because state must survive device switching and is already adjacent to the run registry. User might override to (b) for offline-only deployments.
