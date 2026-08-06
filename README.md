# vera

An open-source agent profile built on the [Hermes](https://github.com/cfollette18/agentic-teams) framework, specialized for **AI research, dataset design, fine-tuning, and evaluation on constrained hardware** (Jetson Orin Nano / edge-GPU class).

`vera` is a *profile*, not a runtime. It is the portable, human-readable part of a Hermes agent — persona, operating principles, skills, rules, learnings, and fixes — with all secrets, session state, and personal memory stripped out.

## What's in here

| Path | What it is |
|------|-----------|
| `SOUL.md` | Agent persona + operating principles |
| `profile.yaml` | Profile metadata |
| `skills/` | Categorized skill packs — the reusable knowledge base |
| `rules/` | Persistent `.mdc` rules (secrets, portability, research workflow) |
| `learnings/` | Teachable patterns loaded additively |
| `fixes/` | Known failure-mode corrections |
| `hooks/` | Gateway hook layout + `hooks-config-snippet.yaml` for shell hooks |
| `notes/` | General research memos (no personal paths or secrets) |

### Skill categories

`apple` · `architectural-planning` · `autonomous-ai-agents` · `creative` · `data-science` · `diagramming` · `domain` · `email` · `gaming` · `gifs` · `github` · `inference-sh` · `reproducible-training-studio` · `mcp` · `media` · `mlops` · `mlops-inference` · `note-taking` · `planning` · `product` · `productivity` · `red-teaming` · `research` · `security` · `smart-home` · `social-media` · `software-development`

Research cores: `research/`, `mlops/`, `mlops-inference/`.

### Portability conventions

Skills and profile artifacts use **placeholders**, not private setup:

| Placeholder | Meaning |
|-------------|---------|
| `$PROJECT_ROOT` | Active ML project repo |
| `$TRAINING_STUDIO_ROOT` | Optional training-studio app |
| `$EDGE_HOST` | SSH/Tailscale name of edge GPU box |
| `$PROJECT_ENV_FILE` | Project credentials file (mode 600) |
| `$DEV_HOST` | Dev laptop hostname |

See `rules/01-no-secrets-or-personal-paths.mdc` and
`rules/02-portable-agnostic-harness.mdc`.

## What is deliberately NOT in here

Runtime state and secrets — none of this is committed:

- `auth.json`, `.env`, `config.yaml` (API keys)
- `state.db`, `sessions/`, `memories/`, `cache/`, `logs/`
- Gateway binaries, cron state, pairing tokens

See `.gitignore` for the full exclusion list.

## Using vera

```bash
# Copy or symlink into a Hermes install
hermes profile import /path/to/vera
# or: ln -s /path/to/vera ~/.hermes/profiles/vera
```

Set credentials via environment variables (`HF_TOKEN`, `PROJECT_TEACHER_*`, etc.) — never in the repo.

## License

MIT — see `LICENSE`. Individual skills declare license in frontmatter.
