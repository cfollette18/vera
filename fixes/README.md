# fixes/ — known failure-mode corrections

Plain `.md` files loaded **additively**. Each fix documents a symptom,
root cause, and the ordered recovery steps.

| File | Topic |
|------|-------|
| `01-credential-leak-risk.md` | Don't persist session-pasted keys to disk |

## When to add a fix

- You burned time on the same mistake twice
- The failure mode is subtle (credential leaks, eval leakage, VRAM OOM)
- Steps are actionable, not narrative

Keep incident timestamps and private hostnames out of portable fixes.
