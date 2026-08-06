---
name: latency-as-rubric-gate
description: Interactive agent? Latency budget per task on real HW.
---

# Latency as a rubric gate (interactive agents)

For interactive agents — CLI assistants, chat UIs, anything where a
human is waiting on the response — latency is a *first-class rubric
dimension*, not a footnote. A model that takes 30s to answer
"what was Apple's revenue last quarter?" is unusable even if its
accuracy is 95%. The user has moved on.

## The pattern: per-task-class latency budget

Define 3–5 task classes and a p50 / p95 budget for each, in seconds,
on the *target deployment hardware* (not the laptop, not a beefy
cloud box). Capture p50 and p95 for every task class at baseline
and after every fine-tune. **Ship-blockers:** any task class
exceeding its p95 budget.

Worked budget for a CLI financial-research agent on a Jetson Orin
Nano (7.4 GiB unified, 3B model Q4_K_M):

| Task class | p50 | p95 | Why |
|---|---|---|---|
| Grounded fact QA (single fact, single period) | 2s | 5s | Interactive feel |
| Cross-company comparison (2–3 cos, one metric) | 6s | 15s | Multi-retrieval cost |
| Multi-document synthesis (e.g., organic/inorganic) | 10s | 25s | Real model work |
| Projection with user-provided assumptions | 8s | 20s | Calc call adds ~50ms |
| First-time ingestion of a new company | – | – | One-time, no budget |

## Rules of thumb (interactive agents, edge hardware)

  - **Fact QA on a 3B Q4_K_M on edge:** p50 ~1–3s, p95 ~3–8s.
  - **Cross-company compare on the same:** p50 ~5–10s, p95 ~10–20s.
    The bottleneck is multi-retrieval, not the model.
  - **Multi-doc synthesis on the same:** p50 ~8–15s, p95 ~20–30s.
    Real model work; the prompt is large.
  - **Anything over 30s p95 for an interactive agent** is broken,
    regardless of accuracy. The user has tabbed away.

These numbers are *starting points*, not laws. The right budget
depends on the user's workflow. Ask: "if this takes 8 seconds,
would you keep waiting or switch tasks?" Tune to the answer.

## When latency is NOT a rubric gate

  - Batch pipelines (overnight eval, offline labeling) — only
    throughput matters, not p95
  - Long-running research tasks where the user explicitly accepts
    minutes (e.g., "build me a 50-page report") — gate on throughput
    and total wall-clock, not per-response latency
  - Real-time streaming (autocomplete, voice) — gate on
    time-to-first-token, not total latency

## Optimization order (when a fine-tune breaks the budget)

Likely fixes, in order of impact:

  1. **Cache retrieval results.** Repeated questions on the same
     evidence are common; cache for 5–15 min.
  2. **Switch quant.** Q4_K_M → Q4_K_S, or even Q4_0. Loses
     ~1pp quality, gains ~30% speed.
  3. **Reduce context length.** RAG stuffing is the biggest
     contributor; cut top-k from 10 to 5.
  4. **Parallelize tool calls.** Two parallel retrievals save the
     sum of two serial retrievals.
  5. **Smaller model.** Last resort. A 1.5B at p95 = 3s may be
     better than a 3B at p95 = 8s for an interactive tool.

## The hardware-mismatch trap

Don't measure latency on the wrong hardware. A 3B QLoRA training
rig with an H100 gives 10s/token for *training* and ~10ms/token
for *inference*; the same model on a Jetson Orin Nano gives
~50–100ms/token for inference. If the deployment target is edge,
measure on edge. Cloud-only latency numbers are fiction for edge
products.

## Display policy (the trust surface for interactive agents)

For an agent that issues cited claims, the UI must show:

  - The answer
  - The cited evidence (clickable in a terminal: `open <id>`)
  - A confidence tag (high/medium/low), derived from retrieval
    score + verifier pass
  - The reason for any refusal, not just "I don't know"
  - The formula and operands for any projection, not just the
    number (the user must be able to audit)
  - A clean one-liner refusal for out-of-scope questions
    ("should I buy X?" → "I can analyze X; I don't make buy/sell
    calls. Try `/analyze X` for fundamentals.")

## Origin

This pattern came up in a session designing a CLI financial-research
agent. The user explicitly preferred a CLI over a web UI, which
made latency a hard constraint. The "no latency budget" pitfall
in `eval-rubric-design` (which says to "capture latency") wasn't
enough — capture is not enough; you need a budget per task class
on the deployment hardware.
