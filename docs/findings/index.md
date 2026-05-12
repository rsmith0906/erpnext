# Critical Review of the AI Integration Plan

## Purpose

This folder contains a critical review of the AI integration plan documented in [`docs/ai-integration/`](../ai-integration/index.md). It is written in the voice of a skeptical reviewer, intended to surface weaknesses *before* the plan is presented to the customer.

**The strategic skeleton is sound. The tactical details are optimistic and incomplete.** These findings identify specifically where.

---

## Reading Order

Read in this order to understand the review's shape:

1. **[Verdict and Recommendations](verdict.md)** — What's right, what's wrong, what to change before the customer sees this plan
2. **[Technical Maturity Issues](technical-maturity.md)** — Where the plan overstates what current AI can reliably do
3. **[Regulatory Gaps](regulatory-gaps.md)** — What's missing in the compliance treatment (AI Act, GDPR, SOX mechanics, state laws)
4. **[Safety Architecture Weaknesses](safety-gaps.md)** — Where the ten-pillar safety framework has implementation gaps
5. **[Missing Topics](missing-topics.md)** — Prerequisites and operational concerns the plan doesn't address
6. **[Technology Choice Issues](technology-choices.md)** — Where the tech selections are biased or not rigorously evaluated

## Engagement-Specific Briefs

- **[AP/AR Engagement Scope](ap-ar-scope.md)** — What ERPNext already does for AP/AR vs. the genuine AI scope; capability set, technical questions, risks, non-negotiables, phased build order, and red flags for an AP/AR-focused engagement

---

## Summary of Findings

| Area | Severity | Finding |
|------|----------|---------|
| Orchestrated workflows (Tier 3) | **High** | Multi-step reliability overstated; error compounding not addressed |
| NL query accuracy | **High** | Accuracy claim not realistic for text-to-SQL on enterprise schemas |
| Confidence calibration | **High** | LLM confidence scores are poorly calibrated; plan leans on them too heavily |
| Hallucination | **High** | Grounding validator layer missing — AI could fabricate entity references |
| EU AI Act | **High** | Annex III classification not analyzed; conformity assessment obligations omitted |
| GDPR Article 22 | **High** | Human-in-the-loop mechanics inadequate under SCHUFA case law |
| SOX mechanics | **Medium** | ITGC scope expansion and auditor attestation effort not planned |
| US state AI laws | **Medium** | Colorado AI Act, NYC LL 144, CA ADMT not addressed |
| Scope realism | **Medium** | Capability count exceeds realistic delivery capacity; ongoing cost categories omitted |
| Schedule realism | **Medium** | Phase durations aggressive for the work specified |
| Audit log immutability | **Medium** | Internal hash chain is defense-in-depth, not true immutability |
| Kill switch races | **Medium** | In-flight LLM calls, workflows mid-execution, about-to-commit actions not addressed |
| SoD in Frappe | **Medium** | ERPNext baseline has `ignore_permissions=True` bypass paths; AI inherits these |
| Rollback scope | **Low** | Cascade, period close, external notifications limit automatic rollback |
| Prompt injection | **Medium** | Defenses described are insufficient against indirect injection and RAG poisoning |
| Data quality prereq | **High** | No data cleansing project in roadmap; AI amplifies existing data issues |
| Change management | **High** | Significant program of work absent from plan |
| Liability framework | **Medium** | Vendor contracts disclaim; customer bears risk — not surfaced |
| Insurance impact | **Low** | E&O, cyber, D&O premium increases not planned for |
| RAG overuse | **Medium** | Vector search is wrong tool for structured financial data |
| Build vs buy | **Medium** | Stampli, Numeric, MindBridge and peers not evaluated as alternatives |
| Provider bias | **Low** | Default Claude recommendation not justified per-capability |

---

## Who Should Read This

- **Before customer presentation** — product/sales owner must review and incorporate
- **Before engineering kickoff** — tech lead must understand limitations
- **Before legal review** — counsel needs the regulatory gaps list
- **Before scope commitment** — program management needs corrected delivery framing
- **Before recruiting** — team composition implied is light for actual scope

---

## One-Paragraph Conclusion

The plan's architectural direction is correct: tiered autonomy, policy-gated automation, audit-first design, human-in-the-loop defaults. But its capability count, its confidence in orchestrated-workflow maturity, its treatment of LLM accuracy as a solved problem, and its partial coverage of regulatory obligations would cause the plan to miss commitments, fail a compliance audit, or deliver a system that degrades user trust. The plan should be rescoped to a tighter set of high-value capabilities, with explicit prerequisite projects (data quality, SoD audit, governance establishment) and a parallel legal/compliance workstream. With those revisions, the plan becomes deliverable and defensible.
