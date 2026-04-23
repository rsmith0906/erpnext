# Implementation Roadmap

An 18-month phased delivery plan for AI-integrating ERPNext. Each phase has defined scope, success criteria, gating conditions, and investment profile.

**Philosophy**: earn trust before taking autonomy. Start read-only, graduate to human-reviewed drafts, then to policy-bound autonomy, then to agents. Never skip phases.

---

## Phase 0: Foundation (Weeks 1–6)

**Goal**: Build the infrastructure that every AI capability will depend on. Ship nothing user-facing yet.

### Deliverables

1. **AI Integration Layer** — the core module with router, policy engine, audit writer
2. **New DocTypes** — `AI Settings`, `AI Capability`, `AI Policy`, `AI Audit Log`, `AI Prompt Template`, `AI Review Queue`, `AI Suggestion`, `AI Feedback`, `AI Model Configuration`
3. **Kill switch** — tested and operational before any capability ships
4. **LLM client library** — with retry, timeout, circuit breaker, cost tracking
5. **PII redaction layer** — with per-field rules
6. **Vector store** — deployed and integrated
7. **Embedding pipeline** — nightly batch + on-change incremental
8. **Observability stack** — dashboards, alerts, cost monitoring
9. **AI service account model** — Frappe Users + Role Profiles for each capability
10. **Incident response runbook**

### Success Criteria

- ☐ Kill switch demo: flip flag, verify all AI workers halt within 60s
- ☐ Audit log demo: run a test invocation, verify entry written with full provenance
- ☐ PII redaction demo: sample payload shows redactions applied before LLM call
- ☐ Fail-open demo: LLM provider returns 500, user save completes successfully
- ☐ Budget alert: hit 75% of monthly budget in test, verify alert fires
- ☐ Tenant isolation: verify vector search never returns cross-tenant data
- ☐ Cost <$500 in Phase 0 infrastructure testing

### Gating Conditions

- Security review passed (especially: API key handling, prompt injection defense, tenant isolation)
- DPIA (Data Protection Impact Assessment) completed if in GDPR scope
- AI Governance Committee formed with accountable owners

### Investment Profile

- **Team**: 2 backend engineers, 1 DevOps, 0.5 security engineer, 0.5 compliance
- **Infrastructure**: vector store (~$200/mo at start), LLM test costs (~$100/mo), observability (existing)
- **Duration**: 6 weeks

---

## Phase 1: Read-Only AI (Months 2–3)

**Goal**: Ship Tier 0 capabilities. Generate value with zero financial-state write risk. Build user familiarity.

### Capabilities

1. **Natural-Language ERPNext Query** (opp. #1)
   - Query bar in Desk: "What was our gross margin by product line in Q2?"
   - LLM translates to report parameters
   - Runs under asking user's permissions
   - Returns formatted table + chart

2. **Anomaly Detection over GL and Stock** (opp. #3, #9)
   - Background job scans new GL Entries and Stock Ledger Entries
   - Flags unusual amounts, accounts, patterns
   - Items go to an "Anomaly Review Queue"
   - No blocking of normal operations

3. **Fraud Detection** (opp. #9)
   - Dedicated queue for suspected fraud patterns
   - Duplicate invoice detection, unusual vendor activity, weekend postings
   - Routes to Internal Audit workspace

4. **Knowledge-Base Copilot** (cross-cutting)
   - Chat UI answering "how do I reverse a Journal Entry?"
   - RAG over ERPNext docs + company SOPs
   - Cites sources

5. **Customer Lifetime Value and Churn Prediction** (opp. #37, #38)
   - Dashboard showing top at-risk customers
   - Predicted LTV by segment
   - Drives sales and credit decisions

6. **Inventory Anomalies** (opp. #18, #19)
   - Slow-moving item detection
   - Count variance flagging during Stock Reconciliation

7. **Lead Scoring** (opp. #60)
   - Every Lead scored 0–100
   - Routing rules per score band

### Success Criteria

- ☐ Phase 0 gating holds up under real load
- ☐ Natural-language query answers correctly on 95% of a 100-question golden set
- ☐ Anomaly detection false-positive rate <20% after 30 days of tuning
- ☐ 50%+ of eligible users have invoked AI features at least once
- ☐ Zero incidents of data leakage, unauthorized access, or AI-caused financial error
- ☐ Monthly LLM cost within budget

### Gating to Phase 2

- Clean audit log review by Internal Audit
- User feedback neutral-to-positive (NPS ≥ 0)
- No open high-severity bugs
- Compliance sign-off on expanding to T1

### Investment

- **Team**: 3 backend engineers, 1 frontend, 0.5 data scientist
- **Duration**: 2 months
- **LLM costs**: $500–$2K/month for mid-market volume

---

## Phase 2: Human-in-the-Loop (Months 4–6)

**Goal**: Introduce Tier 1 capabilities. AI drafts and suggests; humans approve every action. Validate quality of AI output on financial workflows.

### Capabilities

1. **Invoice OCR and Draft Creation** (cross-cutting)
   - Email/upload inbound invoice → AI extracts → Draft Purchase Invoice
   - AP clerk reviews and submits
   - Confidence-weighted field highlighting in UI

2. **Bank Reconciliation AI-Assisted Matching (T1)** (opp. #1)
   - AI proposes matches with reasoning
   - User reviews batch, approves in bulk or individually
   - Target: 60–80% of suggested matches accepted without modification

3. **AP Three-Way Match Assistance (T1)** (opp. #2)
   - Per invoice: AI matches PO and Receipt lines
   - Shows variances with explanation
   - AP clerk accepts or escalates

4. **Journal Entry Account Classification (T1)** (opp. #3)
   - User describes transaction → AI proposes accounts
   - User approves or adjusts
   - Feeds back to retraining corpus

5. **Expense Categorization (T1)** (opp. #7)
   - Expense Claim uploads → AI suggests accounts
   - Employee/approver reviews

6. **Dunning Letter Personalization (T1)** (opp. #6)
   - Generated letters per customer
   - Collections agent reviews and sends

7. **Customer Support Triage + Response Drafting (T1)** (opp. #72, #73)
   - Incoming Issues classified and prioritized
   - Initial responses drafted
   - Agent reviews and sends

8. **Quote Generation from RFP (T1)** (opp. #35)
   - Paste RFP text → AI drafts Quotation
   - Sales rep reviews, adjusts, submits

9. **Email Drafting for Sales (T1)** (opp. #62)
   - Draft personalized outreach
   - Rep reviews before send

10. **Contract Extraction (T1)** (opp. #29)
    - Upload supplier contract → AI extracts terms
    - Legal reviews

### Success Criteria

- ☐ Average AP clerk time per invoice drops from 4min to 90s
- ☐ Bank recon throughput doubles
- ☐ 70%+ of AI draft suggestions accepted without modification
- ☐ User satisfaction NPS ≥ +20 on AI-enabled features
- ☐ Zero AI-caused errors reaching submission (all caught in review)
- ☐ Review queue SLAs met 95%+

### Gating to Phase 3

- Accuracy baseline established for each capability (used for Tier 2 policy approval)
- 30 days of zero-incident operation
- Review queue patterns well-understood
- Statistical confidence in capability accuracy

### Investment

- **Team**: 4 backend, 2 frontend, 1 data scientist, 1 UX designer
- **Duration**: 3 months
- **LLM costs**: $2K–$8K/month

---

## Phase 3: Conditional Autonomy (Months 7–12)

**Goal**: Promote high-accuracy capabilities to Tier 2. AI acts autonomously within tight policy envelopes. Each escalation requires governance approval.

### Capabilities to Escalate (T1 → T2)

1. **Bank Reconciliation Auto-Match (T2)** (opp. #1)
   - Envelope: amount < $10K, confidence > 95%, 3+ prior confirmed matches for pattern
   - Expected: 50–70% of all bank transactions auto-match

2. **AP Three-Way Match Auto-Approve (T2)** (opp. #2)
   - Envelope: exact match (qty + price), amount < $25K, supplier in "trusted" tier
   - Expected: 40–60% of PO invoices auto-approved for payment

3. **Recurring Expense Auto-Categorization (T2)** (opp. #7)
   - Envelope: amount < $500, merchant/vendor seen ≥ 5 times, prior categorization consistent
   - Expected: 70%+ of routine expense lines auto-categorized

### New Capabilities at Tier 0/T1

4. **Cash Flow Forecasting (T0)** (opp. #4)
5. **Customer Payment Date Prediction (T0)** (opp. #5)
6. **Dynamic Reorder Level Calculation (T1)** (opp. #17)
7. **Demand Forecasting (T0)** (opp. #16)
8. **Supplier Scorecard Enhancement (T0)** (opp. #26)
9. **Purchase Price Anomaly Detection (T0)** (opp. #28)
10. **Lead Time Prediction (T0)** (opp. #30)
11. **Production Bottleneck Detection (T0)** (opp. #46)
12. **Quality Pattern Analysis (T0)** (opp. #47)
13. **Project Timeline Estimation (T0)** (opp. #53)
14. **Resource Allocation Suggestions (T1)** (opp. #55)
15. **Win/Loss Analysis (T0)** (opp. #65)
16. **Call Transcription and CRM Updates (T1)** (opp. #64)
17. **Pipeline Forecasting (T0)** (opp. #66)
18. **Semantic Product Search on Portal (T0)** (opp. #77)

### Success Criteria

- ☐ Each T2 escalation requires AI Governance Committee approval with policy envelope and SLA
- ☐ Shadow mode ran 60+ days before T2 activation
- ☐ Post-activation, AI accuracy within 2% of shadow-mode measurement
- ☐ Policy envelope never breached (bug if it is)
- ☐ Zero material financial errors from T2 capabilities
- ☐ AP close cycle compressed 30%+
- ☐ DSO (Days Sales Outstanding) reduced 5–10% via better collections

### Gating Conditions

Before any T1 → T2 promotion:
- 90 days of T1 operation with accuracy ≥ 97% on golden dataset
- 3 months of shadow-mode data demonstrating consistent quality
- Explicit Governance Committee approval
- Rollback procedure tested
- Monitoring dashboards with auto-demote triggers in place

### Investment

- **Team**: 5 backend, 2 frontend, 2 data scientists, 1 ML engineer, 1 UX
- **Duration**: 6 months
- **LLM costs**: $5K–$15K/month (growing with auto-action volume)

---

## Phase 4: Agent Systems (Months 13–18)

**Goal**: Multi-step agents that orchestrate across capabilities. High value, highest complexity.

### Capabilities

1. **Month-End Close Agent (T3)** (opp. #10)
   - Orchestrates the close checklist
   - Runs reconciliations, flags variances, drafts closing entries
   - Requests Controller approval on each material item
   - Generates preliminary close pack
   - Target: 50% reduction in close cycle time

2. **Customer Onboarding Agent (T3)** (opp. #43)
   - Lead to Customer conversion
   - KYC document collection
   - Credit check workflow
   - Initial setup automation
   - Approvals gated per step

3. **Procurement Agent (T2 + T3)**
   - Monitors reorder signals
   - Runs RFQ process
   - Analyzes responses
   - Drafts POs up to policy limits
   - Tracks receipt and invoice match

4. **Production Scheduling Agent (T2)** (opp. #44)
   - Optimizes Work Order sequencing
   - Balances workstation load
   - Reacts to machine downtime
   - Communicates changes to shop floor

5. **Advanced Expense Categorization (T2)** (opp. #7 at scale)
   - Higher thresholds with multi-signal confidence
   - Self-improving from feedback loop

6. **AI-Native Financial Narrative Generation (T1)** (opp. #11)
   - Monthly MD&A draft
   - Variance commentary
   - KPI analysis

### New Capabilities

- Resume screening with bias audits (opp. #69)
- Defect image classification (opp. #82)
- Predictive maintenance (opp. #49)
- Portal self-service chatbot (opp. #78)
- Deeper regulatory compliance (tax code updates, etc.)

### Success Criteria

- ☐ Month-end close: 5-day process → 2–3-day process
- ☐ Close agent handles 80%+ of checklist items without human intervention
- ☐ Customer onboarding time reduced 50%
- ☐ Zero agent-caused incidents requiring regulatory notification
- ☐ Customer deployment metrics: 20–35% finance team time reduction achieved

### Gating to Expansion Beyond 18 Months

- Comprehensive accuracy review across all Tier 2/3 capabilities
- External security audit
- Compliance re-certification
- Customer reference accounts willing to be public references
- Production stability: 99.9% AI availability

### Investment

- **Team**: 6 backend, 3 frontend, 3 data scientists, 2 ML engineers, 1 security, 1 compliance, 1 UX
- **Duration**: 6 months
- **LLM costs**: $10K–$30K/month at scale

---

## Phase 5+: Continuous Evolution (Month 19+)

After the initial 18-month build-out, the work shifts to:

### Ongoing
- **Model refresh cycle**: quarterly evaluation of new LLM releases
- **Capability expansion**: build out remaining opportunities from the catalog
- **Cross-capability optimization**: share context, reduce duplicate calls
- **Custom model training**: fine-tune or adapt local models on tenant data (opt-in)

### Strategic Extensions
- **Industry vertical packages**: pre-configured AI policies for manufacturing, distribution, services, etc.
- **Multi-tenant benchmarking**: anonymized performance benchmarks across deployments (strict opt-in)
- **Partner AI integrations**: connect to vertical-specific AI services (e.g., shipping rate optimizers, tax compliance services)
- **Customer-specific agent development**: customers build their own AI capabilities using the framework

---

## Resource Planning Summary

| Phase | Duration | Team Size | Monthly Run-Rate Cost | Cumulative Cost |
|-------|----------|-----------|----------------------|-----------------|
| 0 | 6 weeks | 3.5 FTE | $35K (team) + $500 (infra) | $55K |
| 1 | 2 months | 4.5 FTE | $50K + $2K | $160K |
| 2 | 3 months | 8 FTE | $90K + $8K | $450K |
| 3 | 6 months | 11 FTE | $120K + $15K | $1.26M |
| 4 | 6 months | 16 FTE | $180K + $30K | $2.52M |

Total 18-month investment: ~$2.5M for a ground-up build targeting a productized offering.

For a single customer deployment of a pre-built platform, multiply the LLM run-rate (~$5K–$30K/month) by their volume and subtract the platform development cost — typically the customer pays a license + implementation, not the full R&D.

---

## Risk Register

### Phase Risks and Mitigations

| Phase | Top Risk | Mitigation |
|-------|----------|------------|
| 0 | Foundation gaps emerge under real load | Phase 1 proves the foundation with low-risk capabilities |
| 1 | Users dismiss AI as unhelpful | UX polish; clear value demos per capability |
| 2 | Draft quality requires heavy rework | Golden dataset gating; iterate prompts before broad rollout |
| 3 | T2 escalation causes an incident | Shadow mode + governance approval + auto-demote triggers |
| 4 | Agent complexity leads to unpredictable behavior | Agents orchestrate only T0/T1/T2 building blocks; no novel agent authority |

### Non-Phase Risks

- **LLM provider pricing changes**: multi-provider strategy; local model fallback
- **Regulatory tightening (AI Act)**: explainability and human oversight already built in; easy compliance path
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

### Decision Point 1: End of Phase 1 (Month 3)
**Question**: Does read-only AI deliver value without surprise risks?
- If yes: proceed to Phase 2
- If no: pause and remediate; deferred T1 rollout

### Decision Point 2: End of Phase 2 (Month 6)
**Question**: Is AI-drafted output consistently accurate enough to graduate to policy-bound autonomy?
- If yes: proceed to Phase 3 with per-capability escalation approvals
- If no: extend Phase 2; invest in accuracy improvements

### Decision Point 3: End of Phase 3 (Month 12)
**Question**: Have T2 capabilities operated without material incidents?
- If yes: proceed to agent systems
- If no: solidify T2 capabilities before adding complexity

Each decision point includes:
- Metrics review
- Governance Committee formal vote
- Customer/stakeholder readout
- Updated risk register

---

## The One-Year Elevator Pitch

> ERPNext becomes an AI-native platform where:
>
> - The finance team closes the books 40% faster
> - AP processes 70% of invoices with minimal human touch
> - Bank reconciliation auto-completes on most transactions
> - Sales teams work AI-scored leads with drafted emails
> - Inventory managers see demand forecasts, not just historical reports
> - Every AI decision is auditable, reversible, and bounded
>
> No GL entry is posted without human accountability in the chain. No material decision is made without explainable reasoning. No data leaves the tenant boundary without consent.
>
> This is ERPNext with AI as a first-class participant — safe enough for regulators, powerful enough to matter.
