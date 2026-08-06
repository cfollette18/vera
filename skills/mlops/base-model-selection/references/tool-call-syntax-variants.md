# Tool-call syntax across model families (verified Aug 2026)

If your fine-tune needs function/tool calling, **each model family
emits tool calls with different delimiters in the response stream**.
The serving framework (llama.cpp, vLLM) has to match the model's
format, or the call is silently dropped. This file is the
quick-reference for the variants verified during the
financial-research-analyst planning session (Aug 2026).

## The matrix

| Model family | Tool-call delimiters in response text |
|---|---|
| Mistral (Ministral-3, Mistral-Nemo, etc.) | `[TOOL_CALLS][{"name": ..., "arguments": {...}}][/TOOL_CALLS]` |
| Qwen (Qwen2.5, Qwen3) | `<tool_call>{"name": ..., "arguments": {...}}</tool_call>` (XML-ish tags) |
| Llama 3.x | `[function_call]` header + JSON; less consistent across fine-tunes |
| SmolLM / HuggingFaceTB | varies; typically JSON block within ```json fenced code |
| Hermes-style (NousResearch) | `<tool_call>` XML tags (same as Qwen) |

OpenAI's native format (the most common teacher output) is
`{"tool_calls": [{"function": {"name": ..., "arguments": ...}}]}` —
which is **none of the above**. You will be transcribing between
formats routinely.

## Why this matters

  - The 5-prompt smoke test (Step 7 of the main skill) must include
    a "use the X tool" prompt. Verify the model emits *parseable*
    JSON inside the model's documented delimiters. If the model
    emits JSON without the delimiters, or uses the wrong
    delimiters, the serving framework won't extract the call —
    and the failure mode is silent, not an error.
  - **llama.cpp's `--chat-template` flag matters more than the
    model card implies.** When you quantize a model and serve it
    with llama-server, the chat template bundled with the GGUF
    may not match the upstream model's tool-call format. If tool
    calls silently fail at serve time, the fix is usually
    pointing `--chat-template` at the model's actual
    `chat_template.jinja` from the HF repo, not the one in the
    GGUF.
  - **vLLM is more forgiving** than llama.cpp because it parses
    a wider range of formats and routes them through a tool-call
    parser. If tool-call reliability is critical and VRAM
    permits, prefer vLLM on the desktop/cloud box; use llama.cpp
    only on the edge where vLLM won't fit.

## Fine-tune data format footgun

Fine-tuning a model on a different tool-call syntax than it was
pretrained with is a real footgun. If you generate training data
in OpenAI's `{"tool_calls": [...]}` JSON format (the most common
teacher output) but the base model uses Mistral's `[TOOL_CALLS]`
format, you'll need to either:

  - post-process the training data into the model's format, OR
  - add chat-template handling to the data pipeline

Do this *before* running the 5,000-example teacher call, not
after. Re-running the teacher is $200+ and 6+ hours.

## Concrete regex patterns (for the data pipeline)

If you're post-processing teacher output into the model's
expected format, the transforms look like:

```python
# OpenAI teacher -> Mistral target
openai_format = '{"tool_calls": [{"function": {"name": "query_facts", "arguments": "{\\"company\\": \\"AAPL\\"}"}}]}'
mistral_format = f'[TOOL_CALLS]{openai_format.replace(chr(34) + "tool_calls" + chr(34), "tool_calls")}[/TOOL_CALLS]'
# (this is illustrative; the real transform is more careful)

# OpenAI teacher -> Qwen target
qwen_format = f'<tool_call>{openai_format}</tool_call>'
```

The exact serialization matters less than the choice to make it
*consistent* between training and serving. If you train on
format A and serve with chat template B, the model emits format
A at inference and the framework can't parse it.

## Origin

Verified during the Aug 2026 financial-research-analyst planning
session, while reading the Mistral-3-3B-Instruct-2512 and
Qwen3-4B-Instruct-2507 model cards. The cards show a calculator
example (Mistral) and a function-calling example (Qwen), but
neither warns you that the *other* model uses a different
syntax. The lesson is: the smoke test (Step 7) is the only
thing that catches this, and "model card looks right" is not a
substitute.
