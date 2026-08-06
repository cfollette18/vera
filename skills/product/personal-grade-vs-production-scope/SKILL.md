---
name: personal-grade-vs-production-scope
description: User says personal? Ask scope questions before scaffolding.
---

# Personal-grade vs production scope

When a user opens a project, the first 60 seconds of scope
questioning determine whether you spend the next 9 weeks building
something they wanted or something they didn't. The two ends of
the spectrum are:

- **Production grade:** multi-tenant, 3+ seeds, full ablations,
  W&B, audit trail, auth, CI/CD, threat model, runbooks,
  monitoring, alerting, release process. ~10x the work.
- **Personal grade:** single user, best-effort rigor, drop the
  ceremony that exists to defend against *other* users and
  *other* time, keep the rigor that defends against *your own
  mistakes*.

Most "fine-tune a 3B model" projects are personal. Most plans
written without asking end up production-shaped, because the
defaults in `eval-rubric-design` and `finetune-recipe-picker`
lean that way (3 seeds, W&B, full ablations, "ship to HF"). That
mismatch is a 2–3x time tax on a solo project.

## Ask these 5 questions before scaffolding

1. **Who is the user?** Just you, or other people on the same
   box? Multi-tenant? Public? Tailscale-only?
2. **What's the deployment target?** Edge (Jetson, phone), laptop
   CPU/GPU, single cloud box, or production serverless?
3. **What's the interface?** CLI, web UI, Telegram bot, API
   only, embedded in another app?
4. **What's the failure cost?** "I'm curious" → personal grade.
   "This signs legal documents" → production grade. Anything
   in between: ask.
5. **How long can you wait?** Overnight run is fine → you can
   do 3 seeds + ablations. Need results in 2 hours → 1 seed
   or none, and the eval has to be fast.

If the answer to 1 is "just me," 3 is "CLI" or "API only," and
4 is "I'm curious" or "I lose an hour if it's wrong," you are
in personal-grade territory. Default the plan accordingly.

## What personal-grade drops (keep what's cheap)

| Drop | Why it's safe to drop |
|---|---|
| W&B / experiment tracking | A local `runs/` dir with loss curves is enough |
| 3rd seed | 2 seeds give ~2pp resolution; sufficient for 10–20pp deltas |
| Production monitoring / alerting | The user runs the system; they'll see errors |
| Auth, rate limiting, multi-tenant | Single user on a private network |
| CI/CD, formal release process | `make publish` is enough |
| Threat model, prompt-injection defense | Personal use, Tailscale-only |
| Full W&B ablation matrix | Keep the 4 mandatory ablations; drop the rest |
| Statistical-significance bar | Report measured numbers; flag the pre-reg; don't over-claim p-values |
| Web UI / mobile / chat surface | CLI is fine for the user |
| SOC2 / audit trail | Not applicable |
| 99.9% uptime SLO | No users to be down for |

## What personal-grade KEEPS (these are cheap and matter)

| Keep | Why it stays even at personal scale |
|---|---|
| Frozen test set with SHA256 + contamination check | Cheap, catches data leaks |
| Pre-registered success criteria | The whole point of the eval |
| Verifier rules in the serving path | The model is still fallible; the user trusts citations |
| Reproducibility manifest (one-command rerun) | The user will want to rebuild in 6 months |
| All four mandatory ablations | They isolate whether the fine-tune even helped |
| The "what we learned" report at the end | The user is the only stakeholder; this is for them |
| Multi-seed run, dropped to 2 | Still gives noise floor for the gains we care about |

## The trap

The trap is conflating "best-effort" with "no rigor." Personal
projects still need:
  - Frozen test sets (otherwise you can't tell if the fine-tune
    helped)
  - Citation faithfulness in the serving path (otherwise the
    user trusts hallucinations)
  - Pre-registered success criteria (otherwise "it works" is
    anecdote, not result)

The discipline that's *optional* is the discipline that exists
to defend against scale, users, and time. Keep the discipline
that defends against your own future confusion.

## Origin

Came up in a session designing a financial-research agent. The
user explicitly said "personal project not production, best you
can, CLI, Tailscale-only." The plan I'd written was 70% production
shape; re-framing as personal-grade dropped 1 seed, the W&B
ceremony, the threat model, the auth plan, and the web UI,
without losing any of the eval discipline or the verifier.

## When to load this skill

  - At the start of any new project, when the user has not
    explicitly named a deployment context
  - When the plan is starting to feel like it has more
    ceremony than the user asked for
  - When the user says words like "personal," "pet project,"
    "for me," "best effort," "fast," "just want to see if it
    works"

## When NOT to load

  - The user has explicitly said "production," "ship to
    customers," "this will be in front of users," "needs to be
    SOC2 compliant"
  - The system handles money, health, safety, or anything with
    regulatory exposure
  - There are already other people in the loop whose work
    depends on this system
