# Fix: credential handling for AI-research projects

## Symptom

The agent pastes an API key (HF token, teacher endpoint, OpenAI) into a
probe script under `data/`, `runs/`, or any non-`/tmp` directory, then
runs the script. The literal key sits on disk in a path the user might
`grep` later or accidentally commit. Auto-leak revokers (HF in
particular) will revoke the token.

## Root cause

The model was given the key in chat for the session. It treats the
inline value as a string it can persist; the path-of-least-resistance
is a one-liner probe script, not a config file in the right home.

## Fix (apply in this order)

1. **HF tokens** → `~/.huggingface/token` (mode 600) + `huggingface-cli
   login`. Never save to repo.
2. **All other API keys** → `$PROJECT_ENV_FILE` (or per-project
   `.env`, mode 600) sourced at launch via
   `set -a; source "$PROJECT_ENV_FILE"; set +a`.
   Add `*secrets*/` and `.env` to the repo's `.gitignore`.
3. **One-shot probe scripts** → write to `/tmp/probe_<random>.py`,
   source the env file inside the script, run, then `shred -u -z`.
4. **When the user pastes a secret in chat**: do NOT save it to any
   file. State the rule, offer the right CLI command, let them run it
   themselves. The session-pasted value is one-shot reading input,
   not file content to persist.

## Project-level hygiene

- `data/generated/` and `*.jsonl` should be gitignored in ML projects.
  Don't fight the layout; respect it.
- `runs/<date>/` is for config + logs, not for env values.
- Keep `~/.config` / keyring as the golden source. Repo files never.

## Verification

After any probe that touches a key:

```bash
git grep -l "MINIMAX_API_KEY\|OPENAI_API_KEY\|HF_TOKEN\|PROJECT_TEACHER_API_KEY" -- :^*.md :^*.example
ls /tmp/probe_*.sh /tmp/probe_*.py 2>/dev/null | xargs -r shred -u -z
test -f "$PROJECT_ENV_FILE" && stat -c '%a %n' "$PROJECT_ENV_FILE"
```
