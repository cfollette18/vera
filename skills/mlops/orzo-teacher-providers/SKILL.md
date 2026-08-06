---
name: orzo-teacher-providers
description: ORZO_TEACHER_* env vars + MiniMax/DeepSeek quirks + creds.
---

# Teacher providers + credentials for orzo dataset gen

`data/gen_dataset.py` calls any OpenAI-compatible chat-completions endpoint
configured via three env vars: `ORZO_TEACHER_BASE_URL`, `ORZO_TEACHER_API_KEY`,
`ORZO_TEACHER_MODEL`. This file records what actually worked (and what didn't)
against each provider tested on this profile. Update after each new test —
don't rely on memory.

---

## MiniMax (api.minimax.io) — verified 2026-08-05

**Working config:**
- `ORZO_TEACHER_BASE_URL=https://api.minimax.io/v1`
- `ORZO_TEACHER_MODEL=MiniMax-Text-01` (clean) or `MiniMax-M3` (reasoning)
- API key shape: `sk-cp-...` (subscription-key style); treated as Bearer token.

**Model family tested:**
| model            | reasoning? | notes                                                  |
|------------------|------------|--------------------------------------------------------|
| MiniMax-Text-01  | no         | clean content, no `<think>` block — safest for gen    |
| MiniMax-M2       | yes        | emits `<think>...</think>` — validator sees raw think |
| MiniMax-M1       | yes        | emits `<think>...</think>` — same issue               |
| MiniMax-M3       | yes        | emits `<think>...</think>`; reasoning_tokens charged  |

**Reasoning model gotcha (M1/M2/M3):**
- The response's `choices[0].message.content` contains `<think>...</think>`
  followed by the actual answer. `gen_dataset.py` reads `.content` directly
  and runs it through `valid_harness()` / `valid_tool_schema()` / etc.
- Result: every output starts with the think block, every validator fails,
  every spec gets counted as `failed`.

**Possible fixes** (pick one before launching reasoning models as teacher):
1. Strip `<think>...</think>` from `resp.choices[0].message.content` before
   validation. Surgical patch to `gen_dataset.py:285`.
2. Send `extra_body={"thinking": {"type": "disabled"}}` — already in the
   existing code for DeepSeek (line 283). MiniMax may or may not honor it;
   needs verification on each new model revision.
3. Use `MiniMax-Text-01` instead — no reasoning, no think block.

**Billing/headers:**
- Standard `usage` field present (`prompt_tokens`, `completion_tokens`,
  `completion_tokens_details.reasoning_tokens`, `total_tokens`).
- No rate-limit headers exposed.
- `/account/billing` → 403 (endpoint exists, separate auth).
- Per-token rates not exposed via API — check the provider dashboard.
- Cost probe before scaling: ~50 tokens is enough to verify a model works.

**Endpoint discovery methodology:**
- User pasted a subscription key with no endpoint URL.
- Probed candidate bases (`api.minimax.io`, `api.minimaxi.com`,
  `api.minimax.chat`, `api.MiniMax.com`) with a 1-shot PONG prompt — first
  hit was `https://api.minimax.io/v1`. Other bases returned 401 (key
  doesn't apply) or DNS NXDOMAIN.
- Probe pattern is reusable: write a small `urllib`-based script that
  loops candidate bases × candidate models, prints status + first 200
  chars of body, and reports the first 200 + `PONG`-containing response.

---

## DeepSeek (api.deepseek.com) — current live workers as of 2026-08-05

**Working config (live workers pid 28383, 60854):**
- `ORZO_TEACHER_BASE_URL=https://api.deepseek.com/v1`
- `ORZO_TEACHER_MODEL=deepseek-chat`
- API key: in the launcher's bash wrapper env (not in repo).

**Known failure rate on harness_scaffold:** ~86% of validation attempts fail.
Per the orzo-pipeline skill's pitfalls section, retry loop is `range(2)`;
bumping to `range(5)` is the documented quick fix.

---

## Credential handling (CRITICAL — user-corrected 2026-08-05)

**Rule: NEVER hardcode API keys in scripts, even throwaway probes.**

- `data/` is mostly gitignored (`data/generated/`, `*.jsonl`) but PROBE
  scripts would NOT be — risk of accidental commit. User caught me doing
  this in three probe scripts; `shred -u -z` was used to wipe them.
- Correct pattern: store creds in `~/.orzo.env` (mode 600), source in shell
  before launch, scripts read from `os.environ` only.
- Probe scripts in `data/` should be prefixed with `_` (e.g.
  `_probe_real_spec.py`) so they're obviously scratch and easy to spot +
  clean up; never commit them.
- After any probe that touches a real endpoint, run
  `grep -rl "<key-prefix>" /home/cfollette18/orzo/` to confirm nothing
  leaked to disk.

**`~/.orzo.env` template:**
```
export ORZO_TEACHER_BASE_URL="https://api.minimax.io/v1"
export ORZO_TEACHER_API_KEY="sk-cp-..."
export ORZO_TEACHER_MODEL="MiniMax-M3"
```
- `chmod 600 ~/.orzo.env`
- Source with `set -a; source ~/.orzo.env; set +a` so all `export` lines
  become shell vars.
- Env vars only enter a process when explicitly sourced — they don't leak
  to other processes or to `/proc/<pid>/environ` of unrelated workers.

---

## Pipeline state table is STALE — always re-check

The orzo-pipeline skill ships a "Pipeline state and YYYY-MM-DD recovery"
table that freezes per-task written-counts at a moment in time. **This
table drifts every generation cycle.** Future sessions MUST re-run:
```
pgrep -af 'gen_dataset.py --task'
for f in /home/cfollette18/orzo/data/generated/*.log; do
  echo "== $(basename $f .log) =="; tail -2 "$f"
done
```
before trusting any per-task count. As of 2026-08-05 the skill's table
said `harness_scaffold ~375` but actual was 500/2400.