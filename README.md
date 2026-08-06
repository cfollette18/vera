# vera

An open-source agent profile built on the [Hermes](https://github.com/cfollette18/agentic-teams) framework, specialized for **AI research, dataset design, fine-tuning, and evaluation on constrained hardware** (Jetson Orin Nano / edge-GPU class).

`vera` is a *profile*, not a runtime. It is the portable, human-readable part of a Hermes agent — the persona, the operating principles, and a curated library of skills — with all secrets, session state, and personal memory stripped out. Drop it into a Hermes install and you get a reproducible research-and-finetuning agent; read it on GitHub and you get a concrete picture of how I build ML pipelines.

## What's in here

| Path | What it is |
|------|-----------|
| `SOUL.md` | The agent persona + operating principles (reproducibility-first, plan-before-train, measure-before-claim, VRAM-aware, researcher-not-engineer). |
| `profile.yaml` | Profile metadata / description. |
| `skills/` | 27 categorized skill packs (~550 files, 12 MB) — the reusable knowledge base. |
| `notes/` | Research notes and design memos that informed the skills. |

### Skill categories

`apple` · `architectural-planning` · `autonomous-ai-agents` · `creative` · `data-science` · `diagramming` · `domain` · `email` · `gaming` · `gifs` · `github` · `inference-sh` · `jig-reproducible-app` · `mcp` · `media` · `mlops` · `mlops-inference` · `note-taking` · `planning` · `product` · `productivity` · `red-teaming` · `research` · `security` · `smart-home` · `social-media` · `software-development`

The research-relevant cores are `research/` (arxiv, blogwatcher, grounded-citations, llm-wiki, research-paper-writing), `mlops/` (training, inference, datasets, eval), and `mlops-inference/`.

### Skill format

Each skill is a directory with a `SKILL.md` carrying YAML frontmatter (`name`, `description`, `version`, `license`, `platforms`, `metadata.hermes.tags` / `related_skills`) plus a markdown body, optionally with `scripts/`, `references/`, and `templates/`. Every skill is individually MIT-licensed.

## What is deliberately NOT in here

This repo contains **only the open-sourceable definition layer**. A live Hermes profile also holds runtime state and secrets — none of that is committed:

- `auth.json`, `.env`, `config.yaml` (API keys, provider credentials)
- `state.db`, `response_store.db`, `verification_evidence.db` (agent state, response logs, evidence)
- `sessions/` (request dumps), `memories/` (personal `USER.md` / `MEMORY.md`), `checkpoints/`
- `cache/`, `logs/`, `audio_cache/`, `image_cache/`, `models_dev_cache.json`, provider-model caches
- `gateway.*`, `pairing/`, `sandboxes/`, `cron/`, `processes.json`, `*.lock`, `.hermes_history`
- `bin/tirith` (compiled Hermes runtime binary)

See `.gitignore` for the full exclusion list — it's written so that copying a live profile over this repo and running `git add -A` will **not** stage secrets.

## Using vera

```bash
# In a Hermes install
hermes profile import /path/to/vera   # or symlink skills/ into ~/.hermes/profiles/<name>/skills
```

Credentials are expected in the environment (`MINIMAX_API_KEY`, `HF_TOKEN`, `ORZO_TEACHER_*`, etc.), never in the repo.

## License

MIT — see `LICENSE`. Individual skills declare their own license in frontmatter (all MIT in this tree).

## Why this exists

`vera` is also a portfolio piece: it shows, in concrete and runnable form, how I think about dataset design, recipe selection, eval rigor, and constrained-hardware fine-tuning — the meta-level habits that make an ML agent effective across projects rather than hardcoded to one.
