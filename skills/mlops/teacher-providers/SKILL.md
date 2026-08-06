---
name: teacher-providers
description: PROJECT_TEACHER_* env vars and OpenAI-compatible teacher quirks.
---

# Teacher providers + credentials for dataset generation

`data/gen_dataset.py` (or equivalent) calls any OpenAI-compatible
chat-completions endpoint configured via:

- `PROJECT_TEACHER_BASE_URL`
- `PROJECT_TEACHER_API_KEY`
- `PROJECT_TEACHER_MODEL`

Store values in `$PROJECT_ENV_FILE` (mode 600), never in the repo.

## Credential handling

1. **Never hardcode** API keys in scripts, skills, or probe files.
2. Source env before launch: `set -a; source "$PROJECT_ENV_FILE"; set +a`
3. One-shot probes → `/tmp/probe_<random>.py`, read from `os.environ`, delete after.
4. After probing, scan for leaks:
   ```bash
   git grep -E 'sk-[A-Za-z0-9]{10,}|ghp_' -- "$PROJECT_ROOT"
   ```

## Reasoning-model gotcha

Some teacher models emit a thinking block before the answer, e.g.
`<think>...</think>` in `message.content`. If
validators run on raw content, every row fails.

**Fixes (pick one):**
1. Strip the think block before validation in the gen script.
2. Pass provider-specific `extra_body` to disable thinking (if supported).
3. Use a non-reasoning model variant for structured JSON generation.

## Provider discovery pattern

When the user has a key but no base URL:

1. Probe candidate bases with a tiny PONG prompt.
2. Loop bases × models; print status + first 200 chars.
3. Record the working triple in `$PROJECT_ENV_FILE`, not in skills.

Example placeholder (never commit real values):

```bash
export PROJECT_TEACHER_BASE_URL="https://api.example-provider.com/v1"
export PROJECT_TEACHER_API_KEY="sk-xxxxxxxxxxxx"
export PROJECT_TEACHER_MODEL="example-model"
chmod 600 "$PROJECT_ENV_FILE"
```

## Pipeline state is always stale

Any skill table of per-task row counts is a snapshot. Before trusting
counts, re-check live:

```bash
pgrep -af 'gen_dataset.py --task'
for f in "$PROJECT_ROOT"/data/generated/*.log; do
  echo "== $(basename "$f" .log) =="; tail -2 "$f"
done
```

## Sister skills

- `edge-qlora-pipeline` — downstream train/export/eval
- `complex-dataset-gen` — parallel worker patterns
- `security/credential-tokens-never-in-config`
