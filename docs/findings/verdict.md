# Verdict and Recommended Revisions

## Overall Assessment

The strategic skeleton is sound. The tactical details are optimistic and incomplete.

This is a plan that *presents* well — the right vocabulary (tiered autonomy, policy envelope, audit log), the right phasing (read-only → human-in-loop → conditional autonomy), and the right default posture (human in the chain for material actions). A savvy customer reading it carefully, or their auditor, will catch where it overreaches. A less-savvy customer will sign up for something that disappoints later.

## What Holds Up

These elements are correctly designed and should survive revision unchanged:

- **Tiered autonomy framework** (T0/T1/T2/T3) — provides the right governance granularity
- **AI Audit Log as a separate artifact from GL** — correct architectural choice; keeps financial records clean and audit log queryable
- **Policy-as-code with `AI Policy` DocType** — versioned, approvable, attachable to audit log entries
- **Fail-open design for hooks** — AI never blocks a user; if AI fails, workflow continues as before
- **Policy thresholds as hard gates regardless of confidence** — critical safety property
- **Kill switch as a first-class feature** — correct pattern, though the implementation details need more rigor
- **Phased approach** — read-only → human-in-loop → conditional autonomy → (eventually) orchestrated workflows
- **AI Service Accounts using Frappe User system** — integrates cleanly with existing permission model
- **Grounding in existing Frappe infrastructure** (RQ queue, hooks, DocTypes) rather than parallel stack

## What Needs Revision

### Must fix before customer presentation

1. **Scope**: cut the capability count to a realistic set prioritized by value × feasibility
2. **Delivery framing**: replace calendar-based phasing with milestone-based phasing tied to gating criteria
3. **Regulatory detail**: add EU AI Act Annex III analysis, mandatory DPIA, Colorado AI Act, state laws, data residency architecture, financial institution rules if applicable
4. **Prerequisite projects**: data quality cleansing, SoD audit of ERPNext baseline, governance committee establishment
5. **Technology honesty**: orchestrated workflows are not magic; confidence is not calibrated by default; prompt injection is not solved; hallucination is the dominant risk

### Must fix before engineering kickoff

6. **Grounding validator layer**: every entity in AI output verified against master data before use
7. **Orchestrator reframing**: "workflow orchestration with AI-augmented steps" not "agents"
8. **NL-query constraints**: LLM selects from approved report parameters, not arbitrary SQL
9. **True audit log immutability**: external timestamping + off-site witness, not just hash chain
10. **Kill switch race conditions**: in-flight, mid-workflow, about-to-commit paths specified and tested
11. **SoD audit**: document known `ignore_permissions=True` paths in ERPNext; set expectations

### Must fix before compliance review

12. **Audit log retention vs GDPR erasure**: pseudonymize after operational need; retain pseudonymized
13. **Cross-border data transfer**: Schrems II TIA; region-routing on `AI Model Configuration`
14. **Fundamental Rights Impact Assessment** for AI Act high-risk use cases
15. **Liability allocation framework**: who is responsible for AI-caused financial error
16. **External auditor engagement**: add to Phase 0 scope

## Rescoped Proposal

If the plan were resubmitted to the customer, it would present:

### Scope
- A tight set of high-value capabilities, not a broad catalog
- Prioritized from the top of the opportunities catalog by value × feasibility
- Explicit exclusions documented (what we're NOT doing and why)

### Delivery Framing
- Milestone-based phasing tied to gating criteria, not calendar commitments
- Phase 0 duration sized to include the full foundation + security review + DPIA, not just engineering
- Phase 2 scope kept small enough that each capability can complete shadow mode validation
- Phase 4 deferred until Phase 3 capabilities have operated stably

### Prerequisites (Phase -1)
These happen *before* AI work starts:
1. **Data quality assessment and cleansing**
2. **SoD audit of current ERPNext deployment**
3. **Governance committee establishment and policy framework approval**
4. **External legal/compliance review**
5. **Data Protection Impact Assessment** if GDPR-scope (parallel)

### Parallel Workstreams
- **Legal/compliance**: continuous across all phases
- **Change management**: starts at Phase 1 kickoff; training curricula developed per capability
- **Security/penetration testing**: at each phase transition
- **External auditor engagement**: starts at Phase 2, expands scope per phase

### Technical Revisions
- **Grounding validator** as core infrastructure
- **Hybrid retrieval** (structured queries + RAG augmentation), not RAG-first
- **Local model option** as first-class, not fallback
- **Build-vs-buy evaluation** per capability with explicit recommendation

### Governance Revisions
- **Decision points** tied to capability performance, not calendar
- **Auto-demote triggers** for any capability whose accuracy degrades
- **Independent AI model validation** by party not building the AI
- **Regulatory change monitoring** as an ongoing program

## Risks to the Revised Plan

Even with these revisions, there are residual risks:

- **Technology evolution**: LLM capabilities change rapidly; plans must adapt
- **Regulatory evolution**: AI Act implementing acts, Colorado AI Act rulemaking, other state laws — moving target
- **Customer organizational readiness**: revised plan still requires customer to stand up a governance committee, allocate dedicated change management, and invest in data quality — many customers resist this
- **Integration complexity**: every capability interacts with existing ERPNext logic; each reveals edge cases
- **Vendor risk**: LLM provider outage, pricing change, ToS change, or discontinuation

## The Go/No-Go Question

**Is this project worth doing?**

Yes, with revisions. The economic value is real (material finance team time reduction, faster close, improved working capital). The safety framework is defensible. The roadmap is the right shape.

But the realistic framing is a smaller scope, longer sequencing, and broader workstream coverage than the plan as written. Under-selling scope and over-selling capability is the original plan's signature problem.

**Recommendation**: revise per this critique, run a Phase 0 POC with two or three capabilities end-to-end, then commit to the full plan with customer signoff on realistic scope and delivery framing.

## What Happens If We Ship the Original Plan Unrevised

Concrete failure modes (ordered roughly by when they surface):

- User trust damaged by a plausible-but-wrong NL-query answer that led to a bad decision
- Compliance review flags EU AI Act gaps; rework required; schedule slips
- LLM run-rate exceeds expectation; pressure to disable capabilities to control cost
- Auditor identifies SOX ITGC scope expansion; audit effort expands
- Tier 2 capability makes material error due to confidence calibration drift; Governance Committee forces demotion; customer perception: "AI doesn't work"
- Team burnout from sustained aggressive pace; key engineers leave
- Project finishes well short of original scope — standard enterprise AI outcome

The revised plan avoids these by being honest up front.

## The One-Sentence Fix

Cut scope, replace calendar commitments with milestone gating, add a prerequisite phase for data quality and SoD audit, staff a legal workstream in parallel, and reframe agents as orchestrated workflows — then the plan is ready for the customer.
