---
name: credential-tokens-never-in-config
description: Token pasted in chat? Never save to config; use CLI cache.
---

# Credential tokens: never save to config, always use the CLI cache

Pattern that came up when a user pasted a HuggingFace write token
into chat. Reflex: write it to a config file. Wrong move. The
correct move is to *not save it anywhere persistent in this session*
and tell the user how to wire it in the right place once.

## Why not save to a config file

  - Config files are readable by every process on the box. A token
    with write scope is a credential, not a setting.
  - If the file ever gets committed to git, the token is leaked,
    and HF auto-revokes it. The user has to re-issue, and we
    burned their trust.
  - There's a *correct* place for it already; using a worse place
    is just asking for the leak.

## The correct place, per service

Each official CLI writes the token to a well-known, mode-600 file
when you run its login command. No code in your repo should ever
see the literal token.

| Service | Login command | Token file | Permissions |
|---|---|---|---|
| HuggingFace (legacy) | `huggingface-cli login` | `~/.huggingface/token` | 600 |
| HuggingFace (new `hf` CLI) | `hf auth login` | `~/.cache/huggingface/token` | 600 |
| GitHub | `gh auth login` | `~/.config/gh/hosts.yml` | 600 |
| OpenAI | `openai login` (if available) | `~/.openai/credentials` | 600 |
| Anthropic | `claude login` | varies | 600 |
| AWS | `aws configure` | `~/.aws/credentials` | 600 |

**Always tell the user the exact command for their service** and
which file it lands in. Then scripts in your repo can authenticate
without ever holding the token:

```python
# HuggingFace: HfApi() reads the cached token automatically
from huggingface_hub import HfApi
api = HfApi()  # no token arg needed
api.create_repo("user/repo", repo_type="model")

# GitHub: gh CLI handles its own auth
import subprocess
subprocess.run(["gh", "repo", "create", "user/repo", "--public"])
```

If a script ever needs the literal token (e.g., to pass to a
non-cooperative tool that doesn't read the cache), prefer
environment variables over hardcoded values:

```bash
# In the user's shell rc, not in the repo
export HF_TOKEN="hf_xxxxxx"

# In the script
import os
api = HfApi(token=os.environ["HF_TOKEN"])
```

## What to do when a user pastes a token in chat

1. **Do not write it to any file in this session.** Not to
   `~/.bashrc`, not to a `.env`, not to a config.yaml, not to
   memory. Memory is for user preferences and stable facts, not
   credentials.
2. **Acknowledge the token and explain the right way to wire it.**
   In one short message: the correct CLI login command, the
   expected file location, and a one-line Python example using
   `HfApi()` (or equivalent) that proves no code needs the
   literal.
3. **Don't proceed with operations that need auth.** The first
   real HF API call that requires write scope will fail with
   401, which is the right signal: that's the moment the user
   runs `huggingface-cli login` (or `hf auth login`).
4. **Do not echo the token back** in subsequent messages, even
   partially. Once the user has it in their head, you don't need
   to remind them; echoing risks shoulder-surfing and screen
   capture leaks.

## What to write to the plan or skills

In the project plan, document the auth setup *procedure* (the
CLI command, the file path, the Python pattern), not the token
itself. The plan should look like:

> ### HuggingFace authentication
> The user provided an HF write token in chat. **I did not save
> it to any config file.** The right place is
> `~/.huggingface/token` (mode 600) on the box that will do
> uploads. Run `huggingface-cli login` once on that box.
> Scripts authenticate via `HfApi()` which reads the cached
> token automatically.

That's enough. No token in the plan, no token in memory, no
token in any file under the repo.

## Common pitfalls

- **Writing the token to a `.env` file in the repo.** Even with
  `.env` in `.gitignore`, a `git add -A` mistake, a backup copy,
  or a tarball inclusion leaks it. The CLI cache is mode-600
  and outside the repo.
- **Saving the token to memory.** Memory is injected into every
  future turn. A leaked token in memory is worse than a leaked
  token in a file — the agent will keep using it.
- **Echoing the token in `python -c` or shell examples.** Even
  in a sandbox, an `echo $HF_TOKEN` in a debug session ends up
  in scrollback. Don't.
- **Assuming `huggingface-cli` vs `hf` — they write to different
  files.** Both are currently valid; `huggingface-cli` is the
  legacy installer and writes to `~/.huggingface/token`, while
  the new `hf` CLI (installed via `curl -LsSf https://hf.co/cli/install.sh | bash`)
  writes to `~/.cache/huggingface/token`. `HfApi()` reads both.
  Tell the user which is installed on their box.
- **Asking the user to re-paste the token in a future session.**
  Once they have it in their CLI cache, you never need to see
  it again. If you find yourself asking, you forgot the cache.
