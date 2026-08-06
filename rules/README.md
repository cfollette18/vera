# rules/ — persistent agent rules

`.mdc` files loaded into the Hermes system prompt for this profile.
Zero-padded prefixes control load order.

| File | Purpose |
|------|---------|
| `00-ai-research-workflow.mdc` | Researcher workflow defaults |
| `01-no-secrets-or-personal-paths.mdc` | Never hardcode secrets, keys, or machine paths |
| `02-portable-agnostic-harness.mdc` | Skills and artifacts must work for any clone |

## Format

```markdown
---
description: one-line trigger
appliesTo: vera-profile
alwaysApply: true
---

# Title

Action-shaped body.
```

Project-local rules can also live in `<cwd>/.hermes/rules/` and merge
with profile rules per Hermes loader priority.
