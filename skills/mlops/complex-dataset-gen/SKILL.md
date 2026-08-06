---
name: complex-dataset-gen
description: Finish multi-task SFT data gen. Fanout, dedupe, split.
---

# complex-dataset-gen — multi-task parallel SFT dataset generation playbook

Use this when generating a synthetic SFT dataset across multiple task
types (e.g. the project's 7 task types: tool_schema, react_trace, rules,
hooks, guardrails, skills, harness_scaffold) and need to finish a
long-running generation job efficiently. Covers parallel fanout, token
budget tuning, race-condition handling, dedupe, validation, and the
train/valid/test split.

## Trigger conditions

- "finish the dataset for X"
- "we're at N/target rows on task Y, get it to target"
- "how do I speed up dataset generation"
- "kill the stale workers and restart cleanly"
- Any multi-task parallel synthetic data gen run with 5+ tasks and
  per-task targets.

## The 7-step playbook (in order — never skip ahead)

### 1. State check — never guess what's already done

```
for f in data/generated/*.jsonl; do echo "$(wc -l < $f) $f"; done
pgrep -af 'gen_dataset.py --task'
ls -la data/generated/*.log
```

Record per-task `(current_rows, target, gap)`. Decide which tasks still
need work. Tasks over target are fine — the dedupe at step 6 normalizes.

### 2. Worker census — kill stale, identify fresh

Multiple launch rounds leave zombie workers. Symptom: `pgrep -af` shows
more workers than expected, all writing to the same JSONL.

```
pgrep -af 'gen_dataset.py --task'
```

**Rule**: keep only the FRESHEST round of workers per task. Kill all
earlier rounds — they double-write rows because their `done` set was
loaded before the new rows landed. Let the fresh round continue.

### 3. Backup before restart

Always snapshot the current JSONL before restarting any worker that
touches it. `data/generated/<task>.jsonl.pre-cleanup.bak` is a fine
naming convention.

```
cp data/generated/harness_scaffold.jsonl \
   data/generated/harness_scaffold.jsonl.pre-cleanup.bak
```

### 4. Fanout launch — slice the spec pool, parallel-process

For a long generation job, the highest-throughput pattern is to slice
the spec pool into N non-overlapping chunks (one per worker process)
and run `--workers K` per process for N×K effective threads.

For example, the existing pattern is `_specs_slice_{1..4}.jsonl` × 4
processes × `--workers 4` = 16 effective threads, each with their own
`done` set loaded at startup, all appending to the same output JSONL.

**Race-condition caveat**: when multiple processes append to one file,
the OS doesn't corrupt line writes (each `json.dumps(...) + "\n"` is
typically <4 KB), but the dedupe at step 6 must run.

```
cd $PROJECT_ROOT && source $PROJECT_ENV_FILE
for i in 1 2 3 4; do
  nohup env PROJECT_TRACKIO=1 PATH="$PWD/.venv/bin:$PATH" \
    .venv/bin/python data/gen_dataset.py \
      --task harness_scaffold \
      --specs data/generated/_specs_slice_${i}.jsonl \
      --out data/generated/harness_scaffold.jsonl \
      --max-tokens 49184 \
      --workers 4 \
    > data/generated/harness_scaffold.fanout_${i}.log 2>&1 &
  echo "slice $i pid=$!"
done
```

### 5. Token budget tuning — match budget to task difficulty

The `gen_dataset.py` default is `--max-tokens 32768`, with retry doubling
to 65536. For reasoning teachers (MiniMax-M3, DeepSeek), 50–90% of the
budget goes to `<think>...</think>` before the actual answer.

Per-task budget recommendations (with M3 as teacher):

| task              | --max-tokens | retry budget | reasoning share | notes
|-------------------|--------------|--------------|-----------------|------
| tool_schema       | 16384        | 32768        | ~40%            | JSON array, fast
| react_trace       | 24576        | 49152        | ~60%            | JSON array, up to 12 steps
| rules             | 12288        | 24576        | ~30%            | markdown, near-zero fail
| hooks             | 16384        | 32768        | ~30%            | Python code
| guardrails        | 16384        | 32768        | ~40%            | Python code
| skills            | 16384        | 32768        | ~30%            | Python code
| harness_scaffold  | 49184        | 98368        | ~70%            | 13-symbol full harness + memory

Bump `--max-tokens` for tasks where the teacher regularly truncates
mid-answer (look for `finish_reason == "length"` in the response).

### 6. Dedupe + validate + split (the hygiene pass)

After all workers finish (or you decide to stop early):

```
# dedupe — strips race-condition duplicates from concurrent workers
.venv/bin/python scripts/dedupe_dataset.py

# validate — schema + per-task validator re-run; expected fails=0
.venv/bin/python data/validate_dataset.py

# split — by spec_id, NOT by row (eval leakage if by row)
.venv/bin/python scripts/split_dataset.py
```

The split MUST be by `spec_id` (not by row) when multiple task JSONLs
draw from the same spec pool. Splitting by row lets the same spec
land in train + valid + test simultaneously.

### 7. Push + verify

```
rsync -av data/ $EDGE_HOST:$PROJECT_ROOT/data/
ssh $EDGE_HOST "wc -l $PROJECT_ROOT/data/generated/{train,valid,test}.jsonl"
```

Pre-train sanity: each task type should be roughly represented in
train.jsonl in proportion to its target share (e.g. harness_scaffold
should be ~35% of train rows for the target target distribution).

## Pitfalls

- **Don't run with `--workers 4` × 4 slices × multiple launch rounds**.
  Each additional round races the prior one. Kill prior rounds before
  launching new ones.
- **Don't change `--max-tokens` mid-run**. Workers started with budget
  X can't share a JSONL safely with workers using budget Y.
- **Don't trust the JSONL row count as "rows written".** A row counts
  only if the validator passed AND the line was flushed. Check
  trackio for `pass=1` counts instead — that's the source of truth.
- **Don't dedupe during a write.** `dedupe_dataset.py` is idempotent
  but its count keeps changing if you run it mid-stream. Wait for
  workers to exit.
- **Don't split during a write.** `split_dataset.py` reads the JSONL
  into memory and re-writes train/valid/test; running it while
  workers write causes inconsistent spec counts across splits.
- **Reasoning models eat your budget.** If pass rate is low AND
  completion looks truncated, bump `--max-tokens`. If pass rate is
  low AND completion is short, the prompt is the problem — not the
  budget.

## Worked example: finishing the project's harness_scaffold (1562 → 2400)

State:
- 1562/2400 rows (5 dupes from prior concurrent runs)
- 16 stale workers from 4 prior launch rounds still alive
- Target: 2400, gap: ~838
- Time budget: ~45 min wall

Steps taken:
1. State check → 1562, 16 stale workers
2. Kill all 16 workers (kept none — easier than deciding which is freshest)
3. Backup: `cp harness_scaffold.jsonl harness_scaffold.jsonl.pre-cleanup.bak`
4. Launch 4 fresh slice workers, `--max-tokens 49184` (was 32768), `--workers 4`
5. Trackio initialized for `dataset-gen-harness_scaffold`
6. Wait for workers to reach 2400 (or stop early at user's call)
7. Dedupe → 2400 (or current count)
8. Validate → 0 fails expected
9. Split → train/valid/test by spec_id
10. rsync to $EDGE_HOST

## When to load other skills

- `edge-qlora-pipeline` — downstream train, export, eval steps
- `teacher-providers` — for which teacher endpoint to use and why
- `dataset-design` — meta-skill for designing the validator + spec generator
- `trackio` — for the live dashboard pattern
