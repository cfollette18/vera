# hooks/ — gateway hook scaffolds

Hermes **gateway hooks** are Python handlers under `hooks/<name>/`:

```
hooks/<name>/
├── HOOK.yaml
└── handler.py
```

This profile ships the directory layout; add hook implementations as needed.
For blocking terminal commands and audit logging, use **shell hooks**
configured in `hooks-config-snippet.yaml` (copied into your live
`config.yaml` under `~/.hermes/profiles/<name>/`).

See the Hermes docs for gateway vs shell vs plugin hook differences.
