# Implementation Roadmap

A phased delivery sequence for AI-integrating ERPNext. Each phase has defined scope, success criteria, and gating conditions. No schedule or budget is specified here — those belong in a separate program-management artifact derived from customer-specific constraints (team composition, organizational readiness, regulatory scope, volume).

**Philosophy**: earn trust before taking autonomy. Start read-only, graduate to human-reviewed drafts, then to policy-bound autonomy, then to orchestrated workflows. Never skip phases.

---

## Phase 0: Foundation

**Goal**: Build the infrastructure that every AI capability will depend on. Ship nothing user-facing yet.

### Deliverables

1. **AI Integration Layer** — the core module with router, policy engine, audit writer
2. **New DocTypes** — `AI Settings`, `AI Capability`, `AI Policy`, `AI Audit Log`, `AI Prompt Template`, `AI Review Queue`, `AI Suggestion`, `AI Feedback`, `AI Model Configuration`
3. **Kill switch** — tested and operational before any capability ships
4. **LLM client library** — with retry, timeout, circuit breaker, cost tracking
5. **PII redaction layer** — with per-field rules
6. **Vector store** — deployed and integrated
7. **Embedding pipeline** — scheduled batch + on-change incremental
8. **Observability stack** — dashboards, alerts, cost monitoring
9. **AI service account model** — Frappe Users + Role Profiles for each capability
10. **Incident response runbook**

### Success Criteria

- Kill switch demo: flip flag, verify all AI workers halt promptly
- Audit log demo: run a test invocation, verify entry written with full provenance
- PII redaction demo: sample payload shows redactions applied before external calls
- Fail-open demo: LLM provider returns error, user save completes successfully
- Budget alert: hit threshold in test, verify alert fires
- Tenant isolation: verify vector search never returns cross-tenant data

### Gating Conditions

- Security review passed (especially: API key handling, prompt injection defense, tenant isolation)
- DPIA (Data Protection Impact Assessment) completed if in GDPR scope
- AI Governance Committee formed with accountable owners

---

## Phase 1: Read-Only AI

**Goal**: Ship Tier 0 capabilities. Generate value with zero financial-state write risk. Build user familiarity.

### Capabilities

1. **Natural-Language ERPNext Query** — constrained to curated approved reports
2. **Anomaly Detection over GL and Stock** — advisory-only flagging
3. **Fraud Detection** — dedicated queue separate from normal workflow
4. **Knowledge-Base Copilot** — RAG over ERPNext docs + company SOPs
5. **Customer Lifetime Value and Churn Prediction** — dashboards for sales and credit
6. **Inventory Anomalies** — slow-moving detection and count variance flagging
7. **Lead Scoring** — automatic with routing rules

### Success Criteria

- Phase 0 gating holds up under real load
- Natural-language query accuracy meets approved target on golden set
- Anomaly detection false-positive rate trending down after tuning period
- Majority of eligible users have invoked AI features
- Zero incidents of data leakage, unauthorized access, or AI-caused financial error
- LLM cost within approved budget

### Gating to Phase 2

- Clean audit log review by Internal Audit
- User feedback neutral-to-positive
- No open high-severity bugs
- Compliance sign-off on expanding to T1

---

## Phase 2: Human-in-the-Loop

**Goal**: Introduce Tier 1 capabilities. AI drafts and suggests; humans approve every action. Validate quality of AI output on financial workflows.

### Capabilities

1. **Invoice OCR and Draft Creation** — AP clerk reviews and submits
2. **Bank Reconciliation AI-Assisted Matching (T1)** — batch review pattern
3. **AP Three-Way Match Assistance (T1)** — variance explanation
4. **Journal Entry Account Classification (T1)** — AI proposes, user approves
5. **Expense Categorization (T1)** — suggestion-based
6. **Dunning Letter Personalization (T1)** — collections agent reviews and sends
7. **Customer Support Triage + Response Drafting (T1)** — agent reviews and sends
8. **Quote Generation from RFP (T1)** — sales rep reviews and submits
9. **Email Drafting for Sales (T1)** — rep reviews before send
10. **Contract Extraction (T1)** — legal reviews extracted terms

### Success Criteria

- AP clerk time per invoice materially reduced
- Bank recon throughput materially improved
- Majority of AI draft suggestions accepted without modification
- Review queue SLAs consistently met
- Zero AI-caused errors reaching submission (all caught in review)

### Gating to Phase 3

- Accuracy baseline established for each capability (used for Tier 2 policy approval)
- Extended period of zero-incident operation
- Review queue patterns well-understood
- Statistical confidence in capability accuracy

---

## Phase 3: Conditional Autonomy

**Goal**: Promote high-accuracy capabilities to Tier 2. AI acts autonomously within tight policy envelopes. Each escalation requires governance approval.

### Capabilities to Escalate (T1 → T2)

1. **Bank Reconciliation Auto-Match (T2)** — amount below configurable threshold; confidence above threshold; pattern has sufficient prior confirmed matches
2. **AP Three-Way Match Auto-Approve (T2)** — exact match; amount below configurable threshold; supplier in "trusted" tier
3. **Recurring Expense Auto-Categorization (T2)** — amount below configurable threshold; merchant/vendor seen sufficient prior times with consistent categorization

### New Capabilities at Tier 0/T1

4. **Cash Flow Forecasting (T0)**
5. **Customer Payment Date Prediction (T0)**
6. **Dynamic Reorder Level Calculation (T1)**
7. **Demand Forecasting (T0)**
8. **Supplier Scorecard Enhancement (T0)**
9. **Purchase Price Anomaly Detection (T0)**
10. **Lead Time Prediction (T0)**
11. **Production Bottleneck Detection (T0)**
12. **Quality Pattern Analysis (T0)**
13. **Project Timeline Estimation (T0)**
14. **Resource Allocation Suggestions (T1)**
15. **Win/Loss Analysis (T0)**
16. **Call Transcription and CRM Updates (T1)**
17. **Pipeline Forecasting (T0)**
18. **Semantic Product Search on Portal (T0)**

### Success Criteria

- Each T2 escalation requires AI Governance Committee approval with policy envelope and SLA
- Shadow mode run for an extended period before T2 activation
- Post-activation, AI accuracy within small margin of shadow-mode measurement
- Policy envelope never breached (bug if it is)
- Zero material financial errors from T2 capabilities

### Gating Conditions

Before any T1 → T2 promotion:
- Extended period of T1 operation with accuracy meeting target on golden dataset
- Shadow-mode data demonstrating consistent quality
- Explicit Governance Committee approval
- Rollback procedure tested
- Monitoring dashboards with auto-demote triggers in place

---

## Phase 4: Orchestrated Workflows

**Goal**: Multi-step workflow orchestration across capabilities. Highest value, highest complexity. Framed as deterministic state machines with AI-augmented steps — not autonomous agents.

### Capabilities

1. **Month-End Close Orchestrator** — runs reconciliations, flags variances, drafts closing entries; Controller approves each material item
2. **Customer Onboarding Workflow** — Lead to Customer conversion with KYC, credit check, setup
3. **Procurement Workflow** — monitors reorder, runs RFQ, analyzes responses, drafts POs within policy
4. **Production Scheduling Assistant** — optimizes Work Order sequencing with overrides preserved
5. **Advanced Expense Categorization (T2)** — higher thresholds with multi-signal confidence
6. **Financial Narrative Generation (T1)** — monthly commentary

### New Capabilities

- Resume screening with bias audits
- Defect image classification
- Predictive maintenance
- Portal self-service chatbot
- Deeper regulatory compliance

### Success Criteria

- Close cycle time materially reduced
- Close orchestrator handles majority of checklist items without human intervention
- Customer onboarding time materially reduced
- Zero orchestrator-caused incidents requiring regulatory notification

---

## Phase 5+: Continuous Evolution

After the initial build-out, the work shifts to:

### Ongoing
- **Model refresh cycle**: regular evaluation of new LLM releases
- **Capability expansion**: build out remaining opportunities from the catalog
- **Cross-capability optimization**: share context, reduce duplicate calls
- **Custom model adaptation**: fine-tune or adapt local models on tenant data (opt-in)

### Strategic Extensions
- **Industry vertical packages**: pre-configured AI policies for manufacturing, distribution, services
- **Multi-tenant benchmarking**: anonymized performance benchmarks across deployments (strict opt-in)
- **Partner AI integrations**: connect to vertical-specific services
- **Customer-specific capability development**: customers build their own using the framework

---

## Risk Register

### Phase Risks and Mitigations

| Phase | Top Risk | Mitigation |
|-------|----------|------------|
| 0 | Foundation gaps emerge under real load | Phase 1 proves the foundation with low-risk capabilities |
| 1 | Users dismiss AI as unhelpful | UX polish; clear value demos per capability |
| 2 | Draft quality requires heavy rework | Golden dataset gating; iterate prompts before broad rollout |
| 3 | T2 escalation causes an incident | Shadow mode + governance approval + auto-demote triggers |
| 4 | Workflow complexity leads to unpredictable behavior | Orchestrators use deterministic state machines; no novel autonomy |

### Non-Phase Risks

- **LLM provider pricing changes**: multi-provider strategy; local model fallback
- **Regulatory tightening**: explainability and human oversight already built in; easy compliance path
- **Model quality regressions on upgrades**: pinned versions; re-validation gate
- **Key personnel departures**: documentation standards; policy-as-code means knowledge lives in the system
- **Data poisoning via compromised inputs**: input validation; circuit breakers; anomaly monitoring on AI behavior itself

---

## Success Metrics Dashboard

Track these continuously across all phases:

### Business Metrics
- Finance team hours per close cycle
- AP invoice processing time
- Bank reconciliation cycle time
- DSO (Days Sales Outstanding)
- Inventory turns
- Stockout rate
- Pipeline conversion rate
- Support ticket resolution time

### AI Quality Metrics
- Accuracy per capability (vs golden dataset)
- User acceptance rate of AI suggestions
- Auto-action success rate (T2)
- Confidence calibration (predicted vs actual)
- Review queue throughput

### Safety Metrics
- Policy envelope breach attempts (should be zero)
- Kill switch activations
- Audit log integrity (hash chain verification)
- PII redaction validation sample pass rate
- Incident count and severity

### Cost Metrics
- LLM spend per capability
- Cost per auto-action (T2)
- Cost per document processed (T1)
- Budget adherence
- Spend efficiency trend

---

## Decision Points

Three explicit go/no-go decision points along the roadmap:

### Decision Point 1: End of Phase 1
**Question**: Does read-only AI deliver value without surprise risks?
- If yes: proceed to Phase 2
- If no: pause and remediate; deferred T1 rollout

### Decision Point 2: End of Phase 2
**Question**: Is AI-drafted output consistently accurate enough to graduate to policy-bound autonomy?
- If yes: proceed to Phase 3 with per-capability escalation approvals
- If no: extend Phase 2; invest in accuracy improvements

### Decision Point 3: End of Phase 3
**Question**: Have T2 capabilities operated without material incidents?
- If yes: proceed to orchestrated workflows
- If no: solidify T2 capabilities before adding complexity

Each decision point includes:
- Metrics review
- Governance Committee formal vote
- Customer/stakeholder readout
- Updated risk register

---

## The Pitch

> ERPNext becomes an AI-native platform where:
>
> - The finance team closes the books materially faster
> - AP processes the bulk of invoices with minimal human touch
> - Bank reconciliation auto-completes on most transactions
> - Sales teams work AI-scored leads with drafted emails
> - Inventory managers see demand forecasts, not just historical reports
> - Every AI decision is auditable, reversible, and bounded
>
> No GL entry is posted without human accountability in the chain. No material decision is made without explainable reasoning. No data leaves the tenant boundary without consent.
>
> This is ERPNext with AI as a first-class participant — safe enough for regulators, powerful enough to matter.
