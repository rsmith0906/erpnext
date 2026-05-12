# Missing Topics

Beyond what the plan gets wrong, there are entire categories of concern the plan doesn't address at all. These are the topics a procurement committee, CISO, or deployment lead will ask about — and their absence from the plan undermines its credibility.

---

## Topic 1: Data Quality Prerequisite

### Why this matters

AI learns from and operates on existing data. Real-world ERPNext deployments have:

- **Duplicate customer/supplier records**: "Acme Corp", "ACME Corporation", "Acme, Inc." as separate records referring to the same entity
- **Inconsistent naming conventions**: item codes with different formats across years
- **Wrong historical classifications**: expenses posted to wrong accounts, corrected later via journal entries
- **Missing dimensions**: older transactions without cost center, project, or accounting dimension
- **Abandoned test data**: dev/QA records mixed with production
- **Orphaned records**: documents referencing deleted masters
- **Currency inconsistencies**: historical rates not applied uniformly
- **Schema evolution residue**: fields added later have NULL for historical records

### Why the plan missed this

The plan assumes clean data as input. It jumps into building AI capabilities without addressing the data they'll operate on.

### The consequence

AI applied to messy data amplifies the problems:

- Bank reconciliation matches to wrong customer because duplicate records make party similarity unreliable
- Expense classification proposes wrong account because historical data inconsistently classified
- Forecasting is inaccurate because historical data has gaps
- Anomaly detection flags historically-correct transactions as anomalies due to drift
- Fraud detection false positives from data quality issues, not fraud

### Recommended addition

A **Phase -1: Data Quality Assessment and Cleansing** before Phase 0:

- Initial data quality audit
- Deduplication (customer, supplier, item)
- Historical reclassification where needed
- Master data governance setup

This is almost always the longest-running surprise in enterprise AI projects. Calling it out up-front is critical.

---

## Topic 2: Change Management

### Why this matters

Technically-excellent AI platforms have failed in deployment because users didn't adopt them. For this plan to deliver value, users need to:

- Trust AI suggestions enough to act on them
- Know how to interpret AI outputs
- Understand the consequences of rejecting vs accepting
- Feel confident providing feedback that improves the system
- Not perceive AI as threatening their jobs

### Why the plan missed this

The plan is engineering-centric. User-facing concerns appear only as "UX suggestion panel" — insufficient.

### What change management actually involves

**Training curriculum** (one per user-facing capability):
- What the AI does and doesn't do
- How to interpret confidence scores
- How to provide feedback
- What to do when AI is wrong
- Who to escalate to
- Hands-on practice with real workflows

**Executive sponsorship**:
- Named executive champion per functional area
- Regular communication on goals, progress, results
- Visible use of AI features by leadership

**Champion network**:
- Identify power users willing to advocate
- Early access and input into design
- Peer-to-peer support

**Resistance management**:
- Some users will actively undermine AI to protect their jobs — this is predictable
- Staffing implications honestly communicated
- Redeployment plan for displaced roles
- Clear distinction between "AI replaces tasks" vs "AI replaces people"

**Feedback loops**:
- Easy mechanism to report wrong AI outputs
- Visible action on feedback (not black hole)
- Metrics on feedback volume and resolution

### Program impact

Change management typically consumes a material share of an enterprise AI program when done well. In this plan: nothing. It needs to be a first-class workstream with its own lead, curriculum, and metrics.

---

## Topic 3: Liability Allocation

### Why this matters

When AI makes a material financial error — wrong amount, wrong account, missed fraud — **who is liable?**

### The current state

- **LLM providers** (Anthropic, OpenAI): contracts aggressively disclaim liability. Typical terms: no warranty, liability capped at a nominal floor or a small multiple of fees paid, indemnification runs *from customer to provider*, not the other way.
- **ERPNext** (open source): no warranty whatsoever under GPL license.
- **Integrator/implementer**: contract typically has professional liability caps at a small multiple of fees paid.
- **Customer deploying it**: bears the financial loss when AI makes an error. Books are wrong. Regulators file against the customer, not the AI vendor.

### What's missing from the plan

- Model contract terms with LLM providers — negotiate beyond defaults for sensitive deployments
- Professional services agreement with integrator — appropriate liability allocation
- Customer-side risk acceptance framework — residual risk customer absorbs
- Insurance-backed guarantees where available

### Recommended addition

1. **Legal workstream Phase 0**: negotiate provider contracts; assess integrator terms
2. **Risk acceptance matrix**: per-capability, what magnitude of error is acceptable; who bears it
3. **Error budget concept**: "AI will be wrong some of the time; here is how we absorb that"
4. **Escalation authority**: for large errors, who decides whether to pursue provider liability vs absorb

---

## Topic 4: Insurance Implications

### What the plan missed

Deploying AI in financial workflows materially changes insurance posture:

**Errors and Omissions (E&O) / Professional Liability**:
- Coverage specifically for AI-caused errors varies by carrier
- Some carriers have AI exclusions or sub-limits
- Premium may rise when AI is added to scope
- Required controls (governance, audit trail, human oversight) align with the plan — but must be documented to carrier

**Cyber Insurance**:
- AI integration expands attack surface (LLM API keys, prompt injection vectors, model poisoning)
- Some carriers require specific AI security controls
- Ransomware coverage may exclude AI-system-targeted attacks
- Premium may rise

**Directors and Officers (D&O)**:
- Governance failures (AI causes material misstatement, regulator fine) are D&O events
- Carriers may require disclosure of AI usage
- Premium may rise

**Technology Errors and Omissions** (for vendors): if the customer is themselves a software vendor passing AI outputs to their customers, separate coverage consideration.

### Program impact

Insurance premium increases are material and are not in the plan. Broker engagement to negotiate these terms is itself a separate effort that needs to be planned for.

---

## Topic 5: Vendor Lock-in and LLM Provider Risk

### What the plan mentioned

"Multi-provider strategy" as a bullet.

### What's missing

**Provider outages**: Anthropic, OpenAI have had extended outages. During an outage:
- Sync AI capabilities fail (plan says "degrade to manual" — operationally what does that mean for users accustomed to AI?)
- Async processing backs up (invoice OCR queue grows; AP delayed)
- SLA exposure to AI customer's own customers

**Model version discontinuation**: Anthropic and OpenAI deprecate model versions with advance notice. Every deprecation requires:
- Revalidation on new version
- Golden dataset rerun
- Potential prompt/capability adjustments
- Meaningful engineering effort per major model upgrade

**Terms of service changes**: Providers have changed terms to allow training on customer data (opt-out by default in some cases). Customer must monitor and adjust.

**Geopolitical risk**: US LLM providers may be restricted in certain jurisdictions. China, Russia, and potentially others have or will have AI sovereignty requirements. Data exports to US providers may become non-compliant.

**Pricing changes**: Providers control pricing unilaterally. Material price changes are possible.

**Acquisition or discontinuation**: LLM provider acquired or discontinued. Unlikely for the major providers in the near term, but non-zero risk.

### Recommended addition

1. **Business continuity playbook**: what operations must continue during provider outage; what degrades; communication to users
2. **Multi-provider operational readiness**: not just "we could switch" but "we routinely exercise switching" — regular failover drills
3. **Local model fallback**: for critical capabilities, a local model as hot fallback
4. **Contract negotiation**: pricing stability clauses, data handling commitments, termination notice, data portability
5. **Scenario planning**: extended outage; price doubles; provider exits US market; etc.

---

## Topic 6: Competitive Intelligence Leakage

### What the plan missed

Sending detailed financial data to a third-party LLM provider exposes competitive intelligence. Even with "no training on our data" contracts:

- **Provider staff access** for abuse investigations — contracts typically allow this
- **Subpoena risk** — if provider is subpoenaed, customer data may be disclosed; some providers' Transparency Reports show this happens
- **Breach at provider** — if provider is breached, customer data exposed
- **Adversarial model extraction** — competitors querying the provider with crafted inputs to infer patterns about customer's data

### What's at stake

Sent to a third-party LLM:
- Customer lists, revenue per customer
- Margin details on specific products
- Supplier relationships and pricing
- New product development (BOMs reveal design)
- Strategic initiatives (M&A, new markets) leaked through natural language queries
- Pricing strategies

### Recommended addition

1. **Sensitivity classification** per capability: what data goes to external LLM, what stays in-tenant
2. **Local model preference for competitive-sensitive capabilities**: BOM analysis, pricing optimization, strategic planning assistance
3. **Data minimization**: send only what the model needs, not full document context
4. **Audit of what leaves**: sampling review of actual data sent to providers
5. **Provider selection criteria**: legal jurisdiction, subpoena transparency, breach history, independent audits (SOC 2 Type II)

---

## Topic 7: Environmental Disclosure

### What the plan missed

Large LLM inference has a measurable carbon footprint. This is increasingly a **disclosure item**:

- **SEC climate disclosure rules** (for public companies in US): Scope 3 emissions include cloud services; AI inference is a significant component for AI-heavy operations
- **EU Corporate Sustainability Reporting Directive (CSRD)**: mandatory for EU companies meeting thresholds; detailed AI/cloud emissions reporting
- **California SB-253**: mandatory Scope 1/2/3 reporting for companies over the applicable threshold
- **Voluntary disclosures**: CDP, TCFD, GRI — investors and customers increasingly ask

### What this means for the plan

For an AI-heavy operation:
- LLM inference emissions need tracking
- Provider emissions data required (Anthropic and OpenAI publish some; varies by region)
- Local model energy use if self-hosted
- Embedding model energy use

### Recommended addition

1. **Emissions tracking**: per capability, per provider, per region
2. **Carbon reporting**: alongside spend in governance dashboards
3. **Green-choice routing**: prefer lower-carbon inference regions where capability allows
4. **Customer commitments**: if customer has public net-zero commitments, AI emissions must fit within envelope

---

## Topic 8: Business Continuity

### What the plan claims

"Degraded-mode operation — every feature degrades cleanly to the existing manual path."

### What's operationally missing

"Degrades cleanly" is easy to write and hard to deliver. Real BC requirements:

- **Queue management during outage**: invoices keep arriving; what's the manual fallback at elevated volume?
- **Communication**: users need to know AI is off; when it's back; what changed
- **Capacity planning**: manual team must be sized for "AI off" operation, not "AI on"
- **Audit continuity**: even during AI outage, manual decisions need to be captured in audit log
- **SLA management**: customer-facing SLAs (support response, invoice processing) may be missed during outage
- **Partial-outage scenarios**: some capabilities work, others don't — what's the operating model?

### Recommended addition

1. **Per-capability BC playbook**: what does "AI off" mean for this specific capability?
2. **Capacity planning**: manual backfill team sized for realistic outage duration
3. **Communication templates**: pre-drafted user and customer communications for AI outage
4. **Partial-outage operating model**: prioritized capability list for degraded operation
5. **Exercise schedule**: regular full-AI-off drill

---

## Topic 9: Intellectual Property Implications

### What the plan missed

AI-generated content raises IP questions:

- **Ownership of AI outputs**: under current US Copyright Office guidance, purely AI-generated content has no copyright protection. Documents generated by AI (contracts, reports, narratives) may not be protectable.
- **Training data exposure**: if tenant opts in to training (some do, for price discount), tenant's proprietary data becomes embedded in provider models
- **Infringement risk**: AI may generate content that infringes third-party copyright (style mimicry, memorized training data)
- **Patent implications**: AI-generated technical designs (BOMs, product concepts) have unclear patent status
- **Trade secret**: sending trade secrets to third-party LLM may constitute disclosure, impairing trade secret protection

### Recommended addition

1. **IP impact analysis** per capability — what content does it generate, who owns it, what's the infringement risk
2. **Training data policy**: default OFF for training opt-in; explicit governance review for any opt-in
3. **Legal review**: outside IP counsel review of output usage, especially customer-facing content
4. **Disclosure to customers**: where AI-generated content is delivered to customers, consider disclosure

---

## Topic 10: Organizational Readiness

### What the plan assumes

That the customer has:
- An AI Governance Committee (or can establish one)
- A Compliance Officer
- A Controller who can evaluate AI policies
- Internal Audit capability
- IT security capability
- Legal counsel familiar with AI regulation
- Data engineering capability
- ML-adjacent engineering talent

### Reality for mid-market customers

Most customers of this scale have only a subset of these. The rest must be:
- Hired (meaningful lead time; significant cost per hire)
- Outsourced (ongoing engagement)
- Developed internally (time cost, risk of inadequate depth)

### Recommended addition

1. **Organizational readiness assessment** as Phase -1 deliverable
2. **Readiness gap plan**: which capabilities customer lacks; how to fill
3. **Managed governance offering**: if customer can't stand up governance committee, vendor offers managed governance service
4. **Template policies and procedures**: reduce the organizational burden

---

## Summary of Missing Topics

| Topic | Severity | Where to add |
|-------|----------|-------------|
| Data quality prerequisite | High | New Phase -1 |
| Change management | High | Parallel workstream; significant share of program |
| Liability allocation | Medium | Phase 0 legal workstream |
| Insurance implications | Low | Plan for premium impact; broker engagement |
| Vendor lock-in | Medium | BC playbook; multi-provider operational readiness |
| Competitive intelligence | Medium | Sensitivity classification; local model preference |
| Environmental disclosure | Low | Emissions tracking; green routing |
| Business continuity | Medium | Per-capability BC playbook; exercise schedule |
| IP implications | Medium | IP impact analysis per capability |
| Organizational readiness | High | Phase -1 assessment; managed services option |

These topics are not "nice to have" — they are what a sophisticated customer will ask about, and the plan's silence on them is a presentation weakness.
