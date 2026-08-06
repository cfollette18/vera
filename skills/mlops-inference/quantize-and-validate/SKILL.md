---
name: quantize-and-validate
description: Post-train HF → GGUF quantize and validate on Orin Nano.
---

# quantize-and-validate

Use this after fine-tuning produces a HuggingFace model directory that
needs to ship to edge hardware (Jetson Orin Nano, Apple Silicon, AMD
laptop). The fine-tune gives you weights in safetensors; the deployment
needs GGUF. The quantize step loses quality; the validate step tells
you whether the loss matters.

## When to use

- "Convert my fine-tuned model to GGUF and run it on the Orin"
- "What quant should I pick for this 1.5B / 3B model?"
- "Validate that quantization didn't destroy my fine-tune"
- "Why is my quantized model worse than the bf16 version?"
- "How do I fit a 7B model on 8 GB?"

## The workflow

### Step 1: Decide where to convert

HF → GGUF conversion is architecture-sensitive (you need llama.cpp's
converter to know your model's architecture). The quantize step is
architecture-agnostic (it's just bit-packing).

Decision matrix:
- NVIDIA laptop + Orin: convert on laptop (5-10x faster), transfer GGUF, quantize on either
- Only the Orin available: convert + quantize both on Orin (slower but fine for ≤3B)
- Apple Silicon: convert there (Metal is fast), transfer
- Cloud-only: convert on cloud, transfer down for quant + validate on Orin

For a 1.5–3B model, the GGUF file is 1–3 GB; transfer over LAN or USB
is fast enough.

### Step 2: Pick the quant

Mirror what's in `finetune-recipe-picker` § Recipes:

| Hardware budget | First try | If quality drops | If still too big |
|------------------|-----------|------------------|------------------|
| 8 GB (Orin Nano)  | Q4_K_M | Q5_K_M, Q6_K | Q3_K_M, IQ4_XS |
| 16 GB (laptop RTX) | Q5_K_M | Q6_K, Q8_0 | Q4_K_M |
| 24 GB+ (desktop)   | Q6_K  | Q8_0 | Q5_K_M |
| Apple M1/M2 16 GB  | Q4_K_M | Q5_K_M | Q3_K_M, IQ4_XS |

Rules of thumb:
- **Q4_K_M** is the default. ~4.5 bits per weight average (most are 4-bit, some are 6-bit). Near-bf16 quality for most tasks.
- **Q5_K_M** if you can afford 25% more RAM and care about subtle reasoning.
- **Q6_K** if you're quality-obsessed and have the RAM.
- **Q8_0** is essentially lossless (~1% perplexity gap); use as a sanity ceiling.
- **Q3_K_M and below** noticeably degrade; only for fitting a model you can't otherwise run.
- **IQ4_XS / IQ3_XXS** are importance-matrix quants; often beat Q4_K_M at the same size for chat, worse for code/reasoning.

### Step 3: Build llama.cpp (or use a binary)

On the Orin Nano (ARM64), `brew install` won't work. Build from source:

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_NATIVE=OFF   # Orin's Cortex-A78AE trips native auto-detect
cmake --build build --config Release -j 4
# Binaries land in build/bin/
```

Or grab a prebuilt ARM64 static binary from
https://github.com/ggml-org/llama.cpp/releases if available.

Required binaries:
- `convert_hf_to_gguf.py` (repo root) — HF → GGUF
- `llama-quantize` (build/bin) — GGUF → GGUF-quantized
- `llama-perplexity` (build/bin) — perplexity validation
- `llama-bench` (build/bin) — speed benchmark

### Step 4: Convert HF → GGUF (architecture-sensitive)

```bash
# From llama.cpp repo root
python3 convert_hf_to_gguf.py \
  /path/to/your-finetuned-model \
  --outfile /path/to/output/your-model-f16.gguf \
  --outtype f16
```

Notes:
- Output is F16 GGUF (full precision). This is your "ceiling" — every quant is derived from it.
- For a 1.5B model expect ~3 GB F16 GGUF; 3B ~6 GB.
- If the model is a fine-tune of a known architecture (Llama, Mistral, Qwen, SmolLM), conversion usually just works. If it's a custom architecture, you may need to add a converter under `llama.cpp/models/`.
- Check for "Successfully wrote" + reasonable file size. If it complains about unknown architecture, diff `config.json` against llama.cpp's model converters.

### Step 5: Compute importance matrix (optional but recommended for ≤Q4)

For Q4 and below, an imatrix gives significantly better quality.
Computed once per model on a calibration corpus (usually wikitext-2):

```bash
# Download calibration text (or use any domain-matched corpus)
# wikitext-2-raw: https://huggingface.co/datasets/Salesforce/wikitext/tree/main/wikitext-2-raw
./build/bin/llama-imatrix \
  -m your-model-f16.gguf \
  -f wikitext-2-raw/wiki.test.raw \
  -o your-model.imatrix \
  -ngl 999 \
  --chunks 100
```

The imatrix file is small (tens of MB). Save it; reuse for different
quants of the same model.

### Step 6: Quantize

```bash
# Without imatrix (most K-quants)
./build/bin/llama-quantize \
  your-model-f16.gguf \
  your-model.Q4_K_M.gguf \
  Q4_K_M

# With imatrix (recommended for IQ* and low-bit K-quants)
./build/bin/llama-quantize \
  your-model-f16.gguf \
  your-model.IQ4_XS.gguf \
  IQ4_XS \
  your-model.imatrix
```

Output naming convention: `<model-name>.<QUANT>.gguf`. Stick with it.

Expected file sizes:
- 1.5B at Q4_K_M: ~1.0–1.1 GB
- 1.5B at Q8_0: ~1.6 GB
- 3B at Q4_K_M: ~1.7–1.9 GB
- 3B at Q5_K_M: ~2.1–2.3 GB
- 3B at Q8_0: ~3.2 GB
- 7B at Q4_K_M: ~3.8–4.1 GB

### Step 7: Validate (the part most people skip)

Three validation layers, cheapest first.

#### 7a. Perplexity on wikitext-2

```bash
./build/bin/llama-perplexity \
  -m your-model.Q4_K_M.gguf \
  -f wikitext-2-raw/wiki.test.raw \
  -ngl 999 \
  -c 512 \
  --chunks 100
```

Compare to the same model at F16:
- Q4_K_M vs F16: < 5% perplexity increase is normal
- Q5_K_M vs F16: < 2% perplexity increase is normal
- Q8_0 vs F16: < 0.5% perplexity increase
- Any quant: > 10% perplexity increase = quality damage, up-shift one tier

#### 7b. Functional smoke test (your task, not wikitext)

Perplexity on wikitext measures language modeling, not your task.
Run a few prompts from your eval rubric against the quantized model:

```python
from llama_cpp import Llama

llm = Llama(
    model_path="your-model.Q4_K_M.gguf",
    n_ctx=2048,
    n_gpu_layers=999,   # Full offload on Orin
    verbose=False,
)

# Same prompts you used in base-model-selection smoke test
smoke_prompts = [
    "Use the `<tool>` tool to look up Apple's revenue.",
    # ... etc
]

for p in smoke_prompts:
    out = llm(p, max_tokens=200, temperature=0.7)
    print(out["choices"][0]["text"])
    print("---")
```

Compare outputs side-by-side with the F16 version. If the quantized
model refuses to follow instructions the F16 followed, that's a
quantization regression even if perplexity is fine.

#### 7c. Latency + memory on the actual deployment hardware

```bash
./build/bin/llama-bench \
  -m your-model.Q4_K_M.gguf \
  -ngl 999 \
  -p 512 -n 128 \
  --repetitions 5
```

Report:
- `tg` (token generation): tokens/sec for new tokens
- `pp` (prompt processing): tokens/sec for prompt ingestion
- Memory: `tegrastats --interval 1000` in another terminal

Acceptable for interactive CLI on Orin:
- `tg` > 10 tok/s for 1.5B Q4_K_M
- `tg` > 6 tok/s for 3B Q4_K_M
- `pp` > 50 tok/s for 1.5B Q4_K_M (RAG prompt ingestion)
- Peak RAM < 90% of Orin's 7.4 GiB usable

### Step 8: Decide ship / no-ship

| Check | Pass | Fail | Action |
|-------|------|------|--------|
| Perplexity gap vs F16 | < 5% (Q4) / < 2% (Q5) / < 0.5% (Q8) | > 10% | Up-shift one quant tier |
| Functional smoke test | Matches F16 on task | Differs | Up-shift one quant tier |
| Latency p95 | Within budget from `latency-as-rubric-gate` | Over budget | Down-shift quant OR reduce context OR smaller model |
| Memory | < 90% of usable RAM | > 95% | Down-shift quant OR enable swap OR smaller model |

If all four pass: ship.
Latency fails but memory is fine: down-shift one quant.
Memory fails: down-shift + try IQ variants.
Quality fails and memory is fine: up-shift one quant.

### Step 9: Document and ship

Write `runs/<project>/quantize.md` with:
- Source: HF model SHA256
- Conversion: llama.cpp commit SHA
- Calibration corpus + SHA256 (if imatrix used)
- Quant: Q4_K_M (or whatever you picked)
- Validation: perplexity F16 vs quant, smoke test outputs, latency table
- Target hardware: Jetson Orin Nano 8GB, jetson-clocks on, MAXN power mode

Save the final GGUF + imatrix + the F16 GGUF (for re-quant if you
change your mind).

For shipping to HF:
```bash
huggingface-cli upload <username>/<model-name>-GGUF \
  ./gguf-output/ \
  --include "*.gguf"
```

## Orin Nano specifics

The Orin has quirks that don't show up on x86:

1. **Power mode matters.** Default is 15W. For real inference:
   ```bash
   sudo nvpmodel -m 0   # 15W, default
   sudo nvpmodel -m 1   # 25W
   sudo nvpmodel -m 2   # MAXN (unlimited, ~25W sustained, may throttle)
   ```
   For benchmarking: MAXN + `sudo jetson_clocks` to lock clocks at max.

2. **Unified memory is shared.** CPU + GPU + system all use the same
   7.4 GiB (after OS carveout). A 1.5B Q4_K_M model uses ~1.2 GB at
   load; you have ~6 GB for context + KV cache.

3. **Context length matters more than model size.** A 7B model at
   512 ctx fits; the same model at 8192 ctx may OOM. KV cache scales
   with layers × heads × ctx_len × 2 (K+V).

4. **Swap is your friend.** If model + context barely fits, a swap
   file on NVMe gives breathing room at 5-10x slowdown for the
   swapped portion. Set it before OOMing.

5. **tegrastats is your observability.** `tegrastats --interval 1000`
   shows RAM, CPU, GPU, power, thermals. Watch for:
   - RAM creeping up during long contexts
   - GPU utilization < 30% = data loader or CPU bottleneck
   - Power dropping = thermal throttle

6. **No native arch auto-detection.** Build with
   `-DGGML_NATIVE=OFF` or your binary may SIGILL on Orin's
   Cortex-A78AE.

## Common pitfalls

- **Token ID mismatch after quantization.** If outputs look like
  garbage characters or wrong language, the tokenizer isn't being
  loaded correctly. llama.cpp reads it from the GGUF, not the
  original HF directory. Verify `tokenizer.model` is embedded.
- **Chat template breaks.** If the quantized model stops following
  system prompts, the chat template was lost in conversion. Pass
  `--chat-template-file` or check `tokenizer_config.json` made it
  into the GGUF.
- **First-token latency vs steady-state.** Token generation
  throughput (tg) tells you steady state; first-token latency
  matters more for interactive UX. Measure with `llama-bench -p 1
  -n 1` and compare to `llama-bench -p 512 -n 128`.
- **Imatrix on the wrong corpus.** Imatrix quality depends on the
  calibration text matching your use case. wikitext is fine for
  general chat; for code use a code corpus; for non-English use that
  language's text.
- **Re-quantizing a quantized model.** Never quantize Q4 → Q2; the
  noise compounds. Always re-quantize from F16.
- **Comparing quant perplexity across models.** Perplexity absolute
  values differ per architecture. Compare within the same model at
  different quants, not across architectures.
- **Q4_K_M is "Q4" but not 4 bits.** Q4_K_M uses mixed precision —
  most weights are 4-bit, some are 6-bit (the "_K_M" supersedes are
  higher precision). The actual average bits per weight is closer
  to 4.5. Don't compare to "true" Q4 from academic papers without
  checking the format.

## When to load other skills

- `llama-cpp` — for the actual serving + quant selection heuristics
- `finetune-recipe-picker` — for VRAM estimates and recipe decisions
- `latency-as-rubric-gate` — for the latency budget per task class
- `eval-rubric-design` — for the functional smoke test design
- `base-model-selection` — for the model card + benchmark info that informed your choice

## Reference: real examples

- `~/orzo/` will need this when the trained model goes to the Orin. The recipe is this skill.
- jig's eval router when it serves the trained model locally: same recipe.
