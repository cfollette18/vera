---
name: orzo-pipeline
description: orzo QLoRA agent-harness gen trained on 'heater' edge box.
---

# orzo pipeline — what every session needs to know

orzo (`/home/cfollette18/orzo`) is a small agent-harness-generator LoRA. The
generic training/dataset/eval tooling lives in `jig` (sister repo). orzo
itself is model + dataset + results only. Full background: `orzo/README.md`
and `orzo/data/README.md`.

## Where things live

- Dev laptop (this machine): x86_64, no GPU, no torch. **Cannot train.**
  Used for: dataset generation, dashboard, watching progress.
- Edge device: Tailscale hostname `heater` (`100.66.219.78`), aarch64, NV
  Power Mode MAXN_SUPER, 7.4 GiB unified, `~/orzo` + `~/orzo-venv` present.
  This is where `bash scripts/run_qlora.sh ...` runs.

## Teacher convention (2026-08-05 update): MiniMax-M3 ONLY

Per user direction (out-of-band message), **all dataset generation uses
MiniMax-M3 only**. No DeepSeek fallback. Implications:

- Endpoint: `https://api.minimax.io/v1` — creds in `~/.orzo.env` (mode 600).
- Model: `MiniMax-M3` — emits `  ` blocks; the script's
  `strip_thinking()` already handles that, but reasoning eats 50–90% of
  the budget, so expect low throughput on long-output tasks.
- Per-task viability with M3:
  | task              | M3 viable? | notes                                              |
  |-------------------|------------|----------------------------------------------------|
  | tool_schema       | yes        | JSON validation, fast (~6/min on M3 with workers=4) |
  | react_trace       | yes (slow) | requires long JSON array with finish action; ~7/min |
  | rules             | yes        | markdown sections; near-zero failure rate          |
  | guardrails        | yes        | Python code; 26% validators-pass first-shot, OK    |
  | hooks             | yes        | Python code; ~0% failure                           |
  | skills            | yes        | Python code; ~5% failure                           |
  | harness_scaffold  | viable but VERY slow | requires 13-symbol full harness + vector memory |
- DeepSeek credentials (`sk-d9d8...`) were hardcoded in the original launch
  command. The launch script `scripts/launch_m3_workers_v2.sh` uses
  whatever's in `~/.orzo.env` instead. The DeepSeek key is NOT in any
  committed file.

## Live observability

- **Orzo stdlib dashboard**: `python dashboard/serve.py --root . --port 8000`.
  Already running on the dev laptop at `http://localhost:8000/`. Polls
  `/api/status` every 2s. Targets: 2400 / 1200 / 900 / 600 / 300 / 300 / 300.
- **Hermes cron watchdog**: job `a60e90811bd1` ("orzo gen watchdog"),
  `no_agent=True`, script at `~/.hermes/scripts/orzo_gen_watchdog.sh`,
  fires every 15 min, `deliver=local`. Tracks per-task JSONL mtime to
  detect stuck workers (no append in 45 min = ALERT).
- **Per-task heartbeat**: `pgrep -af 'gen_dataset.py --task'` +
  `for f in data/generated/*.jsonl; do echo "$(wc -l < $f) $f"; done`.
  Note: progress text in `data/generated/<task>.m3.log` is buffered; check
  row counts on the JSONL itself, not the log.

## To advance the pipeline (canonical sequence)

1. **Wait** for all 7 tasks to hit their targets (watchdog alerts; row
   count on each JSONL ≥ target).
2. **Dedupe**: `.venv/bin/python scripts/dedupe_dataset.py` — strips any
   race-condition duplicates (happens when a stale + new worker both
   append to the same JSONL). Safe to run while workers are writing.
3. **Validate**: `.venv/bin/python data/validate_dataset.py` — schema +
   re-runs each task's VALIDATORS on every row. Fails = 0 expected
   (gen script only writes passing rows).
4. **Re-split** on the dev laptop: `.venv/bin/python scripts/split_dataset.py`.
   The split is **by `spec_id`, not by row** — necessary because all
   7 tasks draw from the same 21,728 specs, and splitting by row would
   let the same spec land in train + valid + test (eval leakage). Default
   5/5 split, seed=42, writes `train.jsonl`/`valid.jsonl`/`test.jsonl`
   in `data/generated/`. Safe to run while a single worker is finishing
   remaining rows; unsafe if MULTIPLE workers write concurrently to
   different task JSONLs at the moment you run split.

5. **Push** data to heater: `rsync -av data/ heater:~/orzo/data/` (or
   git push if the dataset is in the repo; current convention is
   gitignored — `data/generated/` is in `.gitignore`).
6. **Train on heater**: `ssh heater` then
   `bash scripts/run_qlora.sh --data data/generated/train.jsonl --valid
   data/generated/valid.jsonl --output checkpoints/orzo-qwen25-coder-1.5b
   --epochs 2 --lr 2e-4`. Wrapper sets `LD_LIBRARY_PATH` for Jetson
   cuDSS libs and enables `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`.
7. **Log tegrastats alongside**: `bash scripts/tegrastats_log.sh
   runs/orzo-$(date +%Y%m%d-%H%M).tegrastats.log &` in a separate shell.
8. **Export**: `bash export/export_gguf.sh checkpoints/orzo-qwen25-coder-1.5b
   Qwen/Qwen2.5-Coder-1.5B-Instruct` — merges LoRA, converts to GGUF
   Q4_K_M, registers with Ollama.
9. **Eval**: `python eval/run_eval.py --model orzo --specs
   data/generated/test.jsonl --task harness_scaffold --smoke --out
   eval/orzo.jsonl` (and the same for the base model). Compares
   dispatch correctness.

## Pitfalls

- Do NOT run training on the dev laptop — `run_qlora.sh` needs Jetson
  cuDSS libs and there's no CUDA.
- Do NOT launch workers with `&` inside a foreground terminal command —
  Hermes rejects commands that combine shell-level backgrounding
  (`nohup/.../&/disown`) with foreground execution. Use `terminal(
  background=true)` for new server processes, or write a launcher
  script like `scripts/launch_m3_workers_v2.sh` and bash it directly.
- The M3 teacher has a `fail silently` mode: when a worker process dies,
  no log line is emitted until the next 25-success tick. Always check
  `pgrep -af` to confirm a worker is alive, not the log file.
- The `range(2)` retry loop in `gen_dataset.py` (`for attempt in
  range(2)`) gives one retry with 2× token budget. For memory-bearing
  harness_scaffold specs, M3 first-shot success is roughly 1-in-10 even
  with retries — so expect ~1000 spec IDs to never produce a passing
  harness_scaffold row. That's a known dataset ceiling, not a bug.
- The dev laptop dashboard shows N/A for VRAM/temp because this isn't
  Jetson; only the heater dashboard has real vitals.
- After re-splitting, the JSONL splits overwrite in-place — back up
  before re-running if you want a checkpoint of an old split.
