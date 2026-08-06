---
name: architectural-planning
description: Plan redesigns of unfamiliar codebases. Use before TDD.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [planning, architecture, design, redesign, unfamiliar-codebase]
    related_skills: [plan, writing-plans, subagent-driven-development, requesting-code-review]
---

# Architectural Planning

The design-phase sibling of bite-sized TDD planning. The deliverable is an **architectural blueprint + open decisions**, not a numbered task list with copy-pasteable code.

## When this applies

Trigger signals (any one is enough):
  - User asks for "the full end-to-end workflow and happy path"
  - User wants to make a system "replicable", "shippable", "turnkey", or "a real application"
  - User names multiple repos or projects the plan must span
  - Constraints are hardware / cost / operational ("8 GB VRAM", "must run on Jetson", "no cloud budget") rather than functional
  - No feature ticket with acceptance criteria exists yet
  - The user is choosing the destination (UI shape, agent-in-the-loop, etc.) — not you

When **not** to use this:
  - Feature ticket with clear acceptance criteria exists → `plan` / `writing-plans` + TDD
  - User asks "fix bug X" or "add function Y" → `systematic-debugging` or just do it
  - User wants execution, not design → `subagent-driven-development`

## Core method

### 1. Explore (read-only) before you write

Parallelize aggressively. Stop reading once you can answer four questions:
  - What does a successful end-to-end user run look like today?
  - What is the gap between today and the user's stated goal?
  - What is the smallest piece of new code that closes the largest gap?
  - What decision is the user going to need to make, where my default would be wrong?

Recommended read order (each step is a single batch of parallel tool calls):
  1. Locate repos (`find` + `search_files`)
  2. READMEs of every repo involved
  3. Tree dump with extensions filter, excluding `node_modules`, `.venv`, `.git`
  4. Entry points: backend `main.py`, frontend `App.tsx` + router + sidebar, every `scripts/*`
  5. The data model: Pydantic models / DB schema / settings
  6. The "interesting" scripts — whichever scripts carry the system's opinionated choices (validation philosophy, optimization defaults, success criteria)
  7. One representative route per UI category (data table, form, monitor) — not all of them

### 2. Save under `.hermes/plans/<YYYY-MM-DD_HHMMSS>-<slug>.md`

Use a descriptive slug, not the project name. Multiple plans must coexist without collisions.

### 3. Use the architectural-plan outline (NOT bite-sized TDD)

  1. **What the system is today** — concrete inventory, naming real files
  2. **The happy path** — numbered user-facing steps, with file paths and commands
  3. **UX shape** — wireframe in prose
  4. **Critical guarantee sections** — one subsection per user-emphasized requirement, naming the mechanism (e.g. "JSON-schema enforcement at the teacher via `response_format=json_schema`")
  5. **New + changed files** — split into Backend / Frontend / Scripts / Skills, each tagged NEW or CHANGE
  6. **Phased implementation plan** — 3–5 phases, each independently shippable, with time estimates
  7. **Verification** — measurable "done" criteria per major section
  8. **Open decisions** — questions you cannot answer without the user. **Always last.**

## Anti-patterns

  - **Don't include code in the plan.** Exception: JSON-schema examples for external contracts (those are documentation).
  - **Don't propose a rewrite.** Name every NEW file and every CHANGE. If you're tempted to say "replace the runner," you're probably wrong.
  - **Don't hide open decisions.** Without this section you ship a plan that secretly encodes your guesses.
  - **Don't skip the "what is today" inventory.** It's the baseline.
  - **Don't bundle independent decisions into one phase.** Each phase must be independently shippable and reversible.
  - **Don't quote large sections of the plan in chat.** Lead with the path, give a 5–10 bullet summary, then offer next-action options.

## Re-baselining an in-flight plan (audit + continue)

When the user says "audit where X currently is and continue working on it," the project already has a written plan and partial execution. The job is not "follow the plan forward" — it's "compare what the plan said to what the code says, identify the gap, fix what's broken, then resume forward work."

Audit shape that actually helps the user:

  1. **Re-read the plan, the README, the reproducibility manifest, and the recent git log in parallel.** The plan may say "Phase X is done" but the code may not match; the manifest may claim a model lock but the model file may have moved; the test set may be SHA-locked but a new dependency may have shifted its sha256.
  2. **Run the existing test suite.** `make test` (or whatever the project defines) before you trust the codebase. If the suite passes in <2 seconds, do this between every session start.
  3. **Find the project's single high-leverage gate metric** (recall@k before training, dry-run pass-rate before live-model eval, build-time before deploy). Run it. Compare to the recorded number. The gap between "the recorded number" and "what you just measured" tells you whether forward work is safe or whether fix work comes first.
  4. **If git history is empty on an existing repo, commit the baseline before you change anything.** A repo on disk with no git history is one disk failure away from being unrecoverable. Use a pre-commit hook that refuses to modify the frozen test set if its sha256 doesn't match the manifest — catches the most common silent-regression class for ML projects.
  5. **Report state, not narrative.** The audit output lists what's working, what's broken, what's missing, and what you did about each. It does not re-tell the plan or explain the project. The user can re-read the plan; they cannot re-derive what you actually observed.
  6. **Make the gate metric runnable.** If the plan says "if recall@10 < 0.6, fix the retriever before training" but no `make eval-retrieval` target exists, wire one. If the plan says "smoke battery on each base candidate" but no `make smoke` target exists, wire one. Make the gates one command so the next session inherits them.
  7. **End with one decision point for the user.** A long-running multi-phase project has at least 2–3 forward moves (live eval, serving stack, dataset design, training run). Surface them as choices, not as a queue you silently drain. The user picks; you execute.

The highest-leverage move on resume is almost always the **gate metric**, not the next phase. Skipping it means forward work either runs on a broken foundation or redoes work that the gate would have caught.

## References

  - `references/redesign-unfamiliar-codebase.md` — concrete worked example (jig → reproducible edge-LLM training studio), with the actual structure and section sizes that worked for a multi-repo redesign.
  - `references/exploration-checklist.md` — the four gating questions + the read-order shortcut, plus what to do when you can't answer one.
  - `references/open-decisions-template.md` — phrasing conventions for the always-last open-decisions section so users can answer in one line each.

## Related skills

  - `plan` / `writing-plans` — bite-sized TDD planning, use **after** architectural sign-off
  - `subagent-driven-development` — execute the phased plan once approved
  - `requesting-code-review` — review the resulting changes before merge
  - `eval-rubric-design` — when the gate metric is a rubric/eval number (recall@k, pass-rate, latency budget). Its appendix "RAG-specific rubric gates" covers the retrieval-prerequisite pattern that pre-training projects need.
  - `dataset-design` — when forward work is the synthetic-data pipeline. Its "Common failure modes" section has a "frozen test set authored without running it through the full evidence path" pitfall that matches the audit's most common finding.
