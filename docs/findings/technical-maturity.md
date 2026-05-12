# Technical Maturity Issues

The plan in `docs/ai-integration/` treats several capabilities as production-ready when they are actually at the frontier of what current AI can reliably do. A customer reading the plan could reasonably assume these are solved problems. They are not.

---

## Issue 1: Multi-Step Agents (Tier 3) Are Not Solved

### The problem

The roadmap's Phase 4 prominently features a **Month-End Close Agent** that "orchestrates the close checklist — accruals, reconciliations, intercompany eliminations, period-end adjustments." The plan treats this as a routine build task.

### Why this is overstated

Multi-step agent reliability degrades multiplicatively:

| Per-step accuracy | 5 steps | 10 steps | 20 steps | 50 steps |
|-------------------|---------|----------|----------|----------|
| 99% | 95% | 90% | 82% | 61% |
| 95% | 77% | 60% | 36% | 8% |
| 90% | 59% | 35% | 12% | 0.5% |

A realistic month-end close involves **20–50 decision points**. Even with 99% per-step accuracy (far better than any public benchmark), end-to-end success is 61–82%. For financial close, this is unacceptable: a 20% chance of a bad close each month is business-ending.

Current agentic frameworks (LangGraph, Claude tool use, AutoGen) are production-viable for **narrow, well-scoped workflows** (e.g., "extract entities from this document and look them up"). They are **not** production-viable for broad orchestration of stateful multi-decision processes.

### The specific failure modes

- **Drift**: each step's output subtly shifts the context for the next step; by step 15, the agent is working with increasingly inaccurate state
- **Tool misuse**: the agent invokes a capability with wrong parameters because it misunderstood its own prior output
- **Goal misinterpretation**: the agent pursues a sub-goal that doesn't advance the real objective
- **Infinite loops**: the agent retries unhelpfully, burning cost and time
- **Over-confidence cascades**: each step's output is treated as ground truth by the next step, compounding any error

### Recommended revision

**Reframe Tier 3 as "workflow orchestration with AI-augmented steps."**

- A **deterministic state machine** drives the close process (checklist items, dependencies, states)
- Each checklist item is implemented as a Tier 0/1/2 capability with its own safety gates
- The "agent" is just a workflow engine dispatching well-defined calls — not an LLM making top-level orchestration decisions
- Human operator always sees the full state and can intervene at any step
- The LLM is invoked within steps (e.g., "classify this variance"), never as the orchestrator

This is safer AND more feasible than "true" agents. The plan should sell it as the target architecture — do not promise agent autonomy.

---

## Issue 2: Natural-Language Query Accuracy Overstated

### The problem

Phase 1 success criterion: "Natural-language query answers correctly on 95% of a 100-question golden set."

### Why this is overstated

Natural-language-to-SQL is a research problem, not a solved engineering problem, especially for enterprise schemas:

- **Public benchmarks** (Spider, BIRD, Spider 2.0) show state-of-the-art **execution accuracy of 60–80%** on databases with 5–30 tables
- ERPNext has **571+ DocTypes**, many with 50–100+ fields, complex relationships, versioning tables, multi-company dimensions
- Text-to-SQL accuracy drops materially as schema complexity grows
- A published 2024 evaluation of GPT-4 on Spider 2.0 (which has enterprise-grade schemas) showed execution accuracy under **25%**

### The specific risk: plausible wrong answers

The danger is not that the AI says "I don't know." The danger is that it produces a plausible-looking result that is subtly wrong:

- Query ignores a filter ("all invoices" instead of "submitted invoices") — understates AR
- Query joins incorrectly, duplicating rows — overstates revenue
- Query uses wrong date dimension (creation vs posting_date) — wrong period attribution
- Query mixes currencies without conversion — cross-currency total is meaningless
- Query includes cancelled documents — double-counts

A user who sees a clean-looking report will act on it. They will not audit the generated SQL.

### Recommended revision

**Constrain the Phase 1 NL-query feature to a curated set of approved reports.**

- LLM translates natural language → `(report_name, filter_parameters)`, **not** NL → arbitrary SQL
- ERPNext already has ~100 built-in script reports with documented parameter schemas
- LLM picks from this catalog and fills parameters
- Fallback: if no matching report, return "I cannot answer this; here's how to run it manually"

This is tractable today, auditable (every answer ties to a known report), and safe (cannot fabricate a query). Arbitrary text-to-SQL should be a Phase 3+ research track, if at all.

---

## Issue 3: LLM Confidence Is Poorly Calibrated

### The problem

The safety framework leans heavily on "confidence thresholds gate autonomy" — specifically, Tier 2 requires `min_confidence > 0.95`. The plan treats confidence as a known quantity.

### Why this is overstated

LLMs are **notoriously miscalibrated**:

- A model reporting "95% confidence" is often wrong **15–25% of the time** on novel inputs
- Raw token log-probabilities don't transfer well to task-level confidence
- Chain-of-thought outputs tend to be *over*confident (the model rationalizes)
- Fine-tuned or RLHF'd models are especially prone to confident wrongness on edge cases

The plan mentions calibration (Pattern 8) but understates the operational cost:

- Calibration requires **outcome data at scale** — for a brand-new capability, you have no outcomes
- Calibration must be per-capability (generic calibration doesn't transfer)
- Calibration **drifts** when the model version changes — every upgrade requires recalibration
- Ensemble confidence (running N times and measuring agreement) is more accurate but proportionally more expensive

### The specific risk

If raw confidence is taken as true probability:

- A capability labeled "95% confidence required" allows through actions that are actually 80% reliable
- Dollar threshold gating partially compensates, but only for large transactions
- Lots of small transactions × 20% error rate = material aggregate error

### Recommended revision

**Treat LLM confidence as one signal among many, not the gate.**

- Pair every confidence score with **deterministic checks** wherever possible:
  - Exact amount match (for bank recon, 3-way match)
  - Party identity verified against master data
  - Schema conformance (JSON output validates)
  - Range plausibility (amount is within historical distribution for this party)
- Auto-action only when BOTH confidence is high AND deterministic checks pass
- Use calibration map only for *prioritization* (what to show first) not *gating*
- Budget for ensemble confidence where accuracy matters most (cost multiplier accepted)

Be explicit in the documentation that confidence is imperfect. Customers and auditors will ask.

---

## Issue 4: Hallucination Risk Underplayed

### The problem

The plan mentions hallucination in passing but does not build defenses against it. For a financial system, this is the **dominant risk**.

### Concrete failure modes

LLMs will confidently fabricate:

- **Supplier names**: OCR produces "Acme Corp Ltd" but master data has "Acme Corporation" — LLM either fails silently or creates a new supplier record
- **Account codes**: LLM proposes "6200 - Office Supplies" but the company's Chart of Accounts has "6210 - Office Expenses" under a different parent
- **Invoice numbers**: OCR misreads `INV-2024-0123` as `INV-2024-0128`, matching to wrong original invoice
- **Amounts**: LLM vision misreads `$1,450.00` as `$14,500.00` (factor-of-10 error) on handwritten invoices
- **Dates**: LLM infers a date from context when the document doesn't clearly show one
- **Party confusion**: LLM conflates "Acme Corp" (customer) with "Acme Industries" (supplier)
- **Regulatory citations**: LLM cites a tax code section that doesn't exist
- **Historical precedent**: LLM claims "last year you posted similar transactions to account X" when no such history exists

### Why the plan missed this

The plan treats the LLM output as trustworthy after schema validation. Schema validation catches format errors, not hallucinations. A JSON response can be perfectly formed *and* completely fabricated.

### Recommended revision

**Add a grounding validator layer as core infrastructure.**

Before any AI output is acted upon:

1. **Entity verification**: every named entity (supplier, customer, item, account, cost center) in the output must be matched to an existing master record with 100% exact match. Fuzzy matches route to human review; they are never auto-accepted.
2. **Numeric sanity**: amounts must be within a configurable band (historical distribution for this party or type). Outliers route to human.
3. **Date validity**: dates must parse cleanly and fall within the current open fiscal period.
4. **Citation verification**: if the output cites a tax code, regulation, or prior transaction, the citation must be verifiable against a registered source.
5. **Cross-field consistency**: `amount = qty × rate` arithmetic checked; `tax_total = sum(tax_rows)` checked; currency consistency checked.

Grounding validator runs **after** the LLM, **before** any action. It is a deterministic code layer, not an LLM call. Failures return to the human review queue with the specific grounding failure flagged.

This single change is probably the highest-impact safety improvement to the plan.

---

## Summary of Technical Maturity Issues

| Issue | Severity | Fix |
|-------|----------|-----|
| Tier 3 agent overconfidence | High | Reframe as deterministic workflow with AI-augmented steps |
| NL-query accuracy claim | High | Constrain to curated report catalog; no arbitrary SQL |
| LLM confidence miscalibration | High | Pair with deterministic checks; budget ensemble confidence |
| Hallucination defense missing | High | Add grounding validator layer as core infrastructure |

All four should be addressed before Phase 0 begins. They are architectural, not implementation-detail fixes — retrofitting them later is costly.
