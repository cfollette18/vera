---
name: unsloth
description: "Unsloth: 2-5x faster LoRA/QLoRA fine-tuning, less VRAM."
version: 1.0.0
author: Orchestra Research
license: MIT
dependencies: [unsloth, torch, transformers, trl, datasets, peft]
metadata:
  hermes:
    tags: [Fine-Tuning, Unsloth, Fast Training, LoRA, QLoRA, Memory-Efficient, Optimization, Llama, Mistral, Gemma, Qwen]

---

# Unsloth Skill

Comprehensive assistance with unsloth development, generated from official documentation.

## When to Use This Skill

This skill should be triggered when:
- Working with unsloth
- Asking about unsloth features or APIs
- Implementing unsloth solutions
- Debugging unsloth code
- Learning unsloth best practices

## Quick Reference

### Common Patterns

*Quick reference patterns will be added as you use the skill.*

## Reference Files

Slim local index in `references/llms.md`. For full, up-to-date documentation use the official site:

- **https://docs.unsloth.ai**

Use `view` to read `references/llms.md` when a local index is enough; follow links to the official docs for depth.

## Working with This Skill

### For Beginners
Start with [Unsloth getting started](https://docs.unsloth.ai) on the official site.

### For Specific Features
Use the appropriate section on https://docs.unsloth.ai for detailed information.

### For Code Examples
The quick reference section above contains common patterns extracted from the official docs.

## Resources

### references/
- **llms.md** — compact local index (large vendor dumps are not vendored in-repo)

### scripts/
Add helper scripts here for common automation tasks.

### assets/
Add templates, boilerplate, or example projects here.

## Notes

- Prefer https://docs.unsloth.ai over stale vendored copies
- Reference files preserve structure where kept locally
- Quick reference patterns are extracted from common usage examples in the docs

## Updating

To refresh local reference material, re-scrape or update `references/llms.md` from https://docs.unsloth.ai — do not re-add full `llms-full` dumps to the repo.

<!-- Trigger re-upload 1763621536 -->
