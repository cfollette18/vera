# RAG + Fine-tune Architecture for Vertical Expert Systems

When the task is "expert system over a private corpus" — financial
analyst over SEC filings, legal Q&A over case law, medical
literature reasoning over PubMed, scientific paper QA — the
fine-tuned model is one piece of a larger system, not the whole
thing. This reference describes the architecture and the
sector-agnosticism discipline.

## The six components

```
User question
     │
     ▼
┌────────────────────────┐
│ Query planner (model)  │  classifies task: fact | ratio | compare |
└───────────┬────────────┘  reasoning | projection | refusal
            │
   ┌────────┼────────┬─────────────┐
   ▼        ▼        ▼             ▼
┌──────┐ ┌──────┐ ┌──────┐   ┌──────────┐
│Fact  │ │RAG   │ │Web   │   │Calculator│
│store │ │index │ │search│   │+ forecast│
│XBRL  │ │FTS5  │ │live  │   │ engine   │
│+norm │ │+vec  │ │fallb │   │          │
└──┬───┘ └──┬───┘ └──┬───┘   └─────┬────┘
   │        │        │             │
   └────────┴────────┴─────────────┘
                    │
                    ▼
         ┌────────────────────────┐
         │ Fine-tuned 3B model    │
         │ (QLoRA → GGUF Q4_K_M)  │
         │ synthesizes, cites,    │
         │ abstains, explains     │
         └───────────┬────────────┘
                     │
                     ▼
         ┌────────────────────────┐
         │ Verifier (rules + LLM) │
         │ claim↔evidence,        │
         │ arithmetic replay,     │
         │ citation faithfulness  │
         └────────────────────────┘
```

A 3B model with 4–8k context cannot, by itself:
  - Read a 200-page 10-K and synthesize a 5-paragraph summary
  - Compute a multi-step valuation
  - Compare 10 companies on a normalized basis
  - Distinguish "what was reported" from "what was derived"

It can:
  - Orchestrate the right tool calls
  - Read retrieved chunks and extract structured claims
  - Compute simple ratios from a small table
  - Write a citation-faithful explanation
  - Refuse cleanly when evidence is missing
  - Distinguish organic from inorganic growth *when the evidence
    is in the chunk*

The other components carry the load. The model is the *interface*
and the *explainer*, not the calculator, the database, or the
search engine.

## Sector-agnosticism: what's actually agnostic at each layer

A common design failure: pick a sector (e.g., "US SaaS"), train on
filings from that sector, ship something that silently fails on
unfamiliar sectors. The model has memorized vocabulary it doesn't
understand and ignores concepts it doesn't recognize ("net
interest margin" in a bank, "book-to-bill" in semis, "combined
ratio" in insurance).

Sector-agnosticism must be designed in at four layers, each with
different constraints:

**Layer 1 — Model: fully agnostic.** The fine-tune learns the
*behavior* (ground, cite, compute, distinguish organic/inorganic,
refuse). It never sees a hardcoded sector in its system prompt.
The dataset generator must include training examples from ≥4
distinct sectors so the model doesn't over-fit to one industry's
vocabulary. The test set must include a held-out OOD sector to
verify "ground in evidence, refuse when insufficient" works when
the vocabulary is unfamiliar.

**Layer 2 — Retrieval + parsing: agnostic by construction.** The
retrieval index doesn't care about sector. Parsing PDFs into
text+tables is generic. Table-as-structured-record is generic
(row, column, period, unit, footnote) — it doesn't know it's an
income statement, it just preserves structure. The chunker is
generic (split on section, ~512 tokens, with metadata).

**Layer 3 — Normalized fact store: agnostic by schema, specific
by extension.**
  - For SEC filers, XBRL has 10,000+ us-gaap concepts covering
    ~80% of what any filer reports. Ingest `companyfacts` directly.
    This is fully sector-agnostic for US public companies.
  - For company-defined KPIs (the long tail — NRR, RPO, ARR, SSS,
    book-to-bill, NIM, CET1, etc.), there is no universal taxonomy.
    Build a *generic KPI extraction pipeline*: parse the filing,
    identify non-XBRL tables and MD&A disclosures, extract
    `metric_name: value (period, unit)` with the surrounding
    sentence as the `definition_text`. Store as a
    `company_defined_kpi` row.
  - The model never assumes what a KPI means. It always cites
    the definition in the filing that defined it.

**Layer 4 — Calculator / projection engine: agnostic by
operation, specific by template.** Ratio, sum, delta, CAGR,
weighted average, sensitivity are sector-agnostic. Projection
templates (e.g., SaaS: revenue growth × NRR → ARR → revenue;
bank: NIM × earning assets → NII → revenue) are sector-specific.
Ship 1 template per sector as *data*, not as code. The model
identifies sector → looks up template → asks for missing drivers
→ invokes calculator → explains result. Sector is metadata;
template is data.

## International filings (designed in, not implemented in v1)

The parser must be jurisdiction-agnostic (PDF + XBRL + HTML
filing formats all in scope). A taxonomy table maps us-gaap →
ifrs-full → uk-gaap. The fact store stores the original taxonomy
tag; retrieval does the cross-walk at query time. No code path
hardcodes "SEC == US." This is the v0.4 of the project, not v1.

## The dataset mix that prevents OOD failure

If the training set is single-sector, the model will over-fit to
that sector's vocabulary. A model trained only on SaaS filings
will not recognize "net interest margin" or "provision for credit
losses" as concepts that exist. It will either ignore them or
hallucinate. The fix is at the dataset layer:
  - Phase 0 corpus: ≥4 sectors × 2–3 companies each
  - Held-out OOD sector: 1–2 companies the model never trains on
  - Test set must include the OOD sector as a refusal/
    generalization slice, with the target being refusal precision,
    not competence
  - The model never sees the sector label as a feature; sectors
    exist only in metadata and in the held-out test composition

## Concrete phase order (with gates)

  1. **Phase 0 — Scope lock + frozen eval** (week 1)
     - Pick the multi-sector company pool
     - Freeze 300–500 question test set with OOD slice
     - SHA256 the test JSONL
     - Pre-register expected pass rates per task type
  2. **Phase 0.5 — Laptop smoke test** (2–3 days)
     - 5-prompt battery on each candidate model
     - Lock the model + chat template + tool-call format
  3. **Phase 1 — Evidence infrastructure** (weeks 2–3)
     - Acquire raw documents
     - Parse + chunk + index
     - Ingest XBRL companyfacts
     - Build the calculator + projection templates
  4. **Phase 2 — Baseline system + baseline eval** (week 4)
     - Wire up RAG + tools + served base model
     - Run frozen test set
     - Manual rubric sanity check (20–30 examples)
     - Lock baseline numbers
  5. **Phase 3 — Dataset design + generation** (weeks 5–6)
     - Spec generator driven by baseline failures
     - Multi-sector spec mix
     - Programmatic arithmetic, not teacher arithmetic
     - Distractor + contradictory + OOD evidence
     - Hand-labeled organic/inorganic subset
  6. **Phase 4 — Training on edge** (week 7)
     - 1-step fit probe → pilot → hparam sweep → 3 seeds
     - Mandatory ablations: RAG-only, no-calculator,
       clean-contexts, base vs fine-tuned
  7. **Phase 5 — Calculator + projection** (week 8, parallel to 4)
     - Sector-specific projection templates as data
     - Tornado sensitivity, scenario reproduction tests
  8. **Phase 6 — Export + serve** (week 9)
     - Merge LoRA → GGUF on x86 laptop
     - llama.cpp + thin API + verifier
     - Upload merged GGUF + adapter + dataset to HF

Building the dataset before the baseline is the order that makes
"we trained for 8 hours and the rubric was wrong" happen.

## Reference example

`/home/cfollette18/.hermes/plans/financial-research-analyst.md` is
a worked example of this architecture applied to a financial
research analyst. The §1.5 "Sector-agnosticism" section is the
canonical treatment of the four-layer design.
