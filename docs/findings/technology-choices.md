# Technology Choice Issues

The plan makes several technology recommendations that reflect author bias, incomplete evaluation, or uncritical preference. These should be revisited before committing.

---

## Issue 1: RAG Overuse for Structured Financial Data

### What the plan recommends

Vector store for master data, historical transactions, prior matches, with RAG as the primary retrieval mechanism for most capabilities.

### Why this is questionable

Retrieval-Augmented Generation shines for **unstructured** data — documents, articles, emails, narrative content. For **structured** financial data, it is often the wrong tool:

**Wrong-tool scenarios**:
- "Find all invoices for customer X above a threshold" — SQL does this perfectly and exactly; vector search is approximate and can miss rows
- "What's the current outstanding balance for supplier Y?" — exact aggregation query; RAG is useless
- "Which account was this expense posted to last year?" — direct lookup; RAG adds noise
- "List all GL entries in period Z" — range query; RAG is worse than useless
- "Does this supplier exist?" — existence check; RAG's "similar suppliers" returns 10 near-matches when we want yes/no

**Where RAG adds value for ERPNext**:
- Similar past documents for pattern matching ("find historical invoices that look like this one")
- SOP and knowledge base search
- Comment and description field semantic search
- Policy and regulation lookup
- Unstructured attachments

### Specific failure mode

If AI needs "current balance for customer ABC":
- **SQL approach**: `SELECT SUM(debit) - SUM(credit) FROM tabGL Entry WHERE party='ABC' AND docstatus=1`. Always correct.
- **RAG approach**: retrieve top-K similar documents, have LLM sum from them. May miss transactions, may double-count, may use stale embeddings.

### Recommended revision

**Hybrid retrieval as the default pattern**:

1. **Structured queries first**: if the question can be answered by SQL or `frappe.qb` against current data, do that
2. **RAG as augmentation**: retrieved documents provide context; they don't provide facts
3. **Capability-per-capability retrieval design**: each capability specifies what retrieval it needs; RAG is not assumed

Rewrite the architecture section's RAG chapter to clarify:
- Embed **unstructured** fields (descriptions, comments, SOPs)
- Do not embed structured transaction data as the primary retrieval mechanism
- Use SQL for "what is the state" questions
- Use RAG for "what's similar to this" questions

Estimated scope reduction: RAG infrastructure drops materially; retrieval accuracy improves.

---

## Issue 2: Self-Hosted Models Treated as Second-Class

### What the plan says

Local models (Llama, Qwen) are described as "for high-sensitivity deployments" — as a privacy fallback, not a primary option.

### Why this framing is wrong

**Self-hosted models are production-viable** for many ERPNext capabilities:

- **Llama 3.3 70B Instruct** — competitive with GPT-4 class on many benchmarks; Apache 2.0 licensed
- **Qwen 2.5 72B** — strong on reasoning; open license with commercial use allowed
- **Mistral Large / Mixtral** — production-ready
- **Domain-specific models** — FinLlama and similar trained on financial text

**Advantages self-hosting has**:
- **No egress**: data never leaves customer infrastructure — solves Schrems II, competitive intelligence, and PII concerns
- **Deterministic availability**: no provider outage risk; customer controls uptime
- **Cost predictability**: no surprise pricing changes; amortize hardware
- **Volume economics**: at sufficient volume, self-hosting is economically competitive with API
- **No contractual limits**: no rate limits imposed by provider
- **Custom fine-tuning**: train on tenant data with full control

**Operational considerations**:
- GPU infrastructure (owned or cloud-rented)
- Inference orchestration (vLLM, TensorRT-LLM): open source
- Ops burden: DevOps/MLOps capacity for production operations
- Model ops (version upgrades, evaluation): ongoing

For high-volume deployments, self-hosting becomes economically favorable over the program lifetime.

### When self-hosting is preferable (not fallback)

- Customer has strong data sovereignty requirements (EU, government, financial, healthcare)
- Customer has high inference volume
- Customer has in-house ML/DevOps capability
- Customer competitive sensitivity is high
- Customer needs deterministic latency (not subject to provider queue)

### Recommended revision

1. Present **self-hosted as a first-class deployment option**, not fallback
2. Add decision matrix: API vs self-hosted based on volume, sensitivity, ops capability
3. Architecture supports both from day one (`AI Model Configuration` already does — just change positioning)
4. Dual-path validation: each capability works with both API and local model
5. Migration path: customers can start with API and migrate to self-hosted at scale

---

## Issue 3: Claude Recommendation — Author Bias Not Disclosed

### What the plan recommends

Claude Sonnet 4.6 as default for most capabilities; Claude Opus 4.7 for high-accuracy reasoning; Claude Haiku 4.5 for volume classification.

### Why this is an issue

The plan's default-Claude posture was adopted without per-capability evaluation, which is:
- Not rigorous
- Not best-practice
- Potentially financially disadvantageous to the customer

### What rigorous selection looks like

Per capability, evaluate candidates on:

| Criterion | Weight |
|-----------|--------|
| Accuracy on golden dataset | High |
| Inference cost per unit | High |
| Latency (p50, p99) | Medium |
| Structured output reliability | High |
| Context window (for RAG) | Medium |
| Tool use quality | High |
| Data handling (training, retention) | High |
| Availability / SLA | Medium |
| Integration complexity | Low |

Candidates to consider for each capability:
- **Claude Sonnet, Opus, Haiku** (Anthropic)
- **GPT-4o, GPT-4 family, GPT-4-mini** (OpenAI)
- **Gemini Pro, Flash** (Google)
- **Llama, Qwen** (open weights)
- **Specialized**: Voyage for embeddings; Cohere for multilingual; Mistral for European deployment

### Specific capabilities where non-Claude may be better

- **OCR/vision extraction**: GPT-4o and Gemini have strong vision; evaluate
- **Multilingual (EU customers)**: Cohere Command R+ is strong; Mistral is EU-based
- **High-volume classification**: smaller/faster models competitive; open-weights models may suffice for simple classification
- **Code generation (for NL-query)**: GPT-4 class has historical edge, though the margin shrinks
- **Long-context (large Chart of Accounts)**: Gemini Pro has the largest context window

### Recommended revision

1. Replace "Claude as default" with "per-capability evaluation required"
2. Define **evaluation methodology**: benchmark on golden dataset, measure accuracy and cost, decide
3. **Model routing table**: each capability has a primary and fallback model, selected by evaluation
4. **Ongoing re-evaluation**: regular review as new models release
5. **Disclose source bias**: note that the original recommendations came from an Anthropic-authored source and need independent validation

---

## Issue 4: Build vs Buy Not Evaluated

### What the plan assumes

"Build everything in-house."

### Why this is wrong

For many capabilities in the plan, **specialized vendors offer production-ready solutions**:

| Plan capability | Specialized vendors |
|----------------|---------------------|
| Invoice OCR + AP automation (#2) | Stampli, Tipalti, Bill.com, AP automation suites |
| Expense management (#7) | Ramp, Brex, Airbase, Navan |
| Close automation (#10) | Numeric, FloQast, BlackLine |
| Anomaly detection / fraud (#3, #9) | MindBridge, Trintech, AppZen |
| AR / collections (#5, #6) | HighRadius, Billtrust, Esker |
| Contract extraction (#29) | LinkSquares, Ironclad, Spotdraft |
| Natural-language-to-SQL (#1) | Vanna.ai, Databricks Genie, Snowflake Cortex |
| Customer support AI (#72, #73) | Intercom Fin, Zendesk AI, Forethought |
| Lead scoring / sales AI (#60, #61) | Salesforce Einstein, HubSpot AI, Gong |

These are **mature, SOC 2 certified, battle-tested** products with ERPNext integrations often pre-built.

### Build vs buy decision factors

Build when:
- Deep integration with ERPNext-specific workflows
- Capability is unique to customer's business
- Customer has in-house capability and capacity
- Multi-year cost curve favors build at customer scale
- Vendor options don't meet security/data sovereignty requirements

Buy when:
- Commodity capability (OCR, expense reports)
- Short time-to-value needed
- Customer lacks AI/ML expertise
- Vendor has SOC 2, regulatory certifications customer needs
- Integration complexity with ERPNext is manageable via existing APIs
- Total cost of ownership favors subscription

### Recommended revision

1. Add **build-vs-buy decision matrix** to Phase 0 planning per capability
2. For each plan capability, explicitly evaluate: build, buy, or hybrid (buy + integrate)
3. Change Phase 1/2 scope based on outcomes — likely a substantial share of "build" capabilities become "integrate vendor"
4. Integration-focused engineering work may replace building capabilities from scratch
5. Customer-facing narrative: "best-of-breed AI ecosystem" not "monolithic build"

This may materially reduce build scope and improve time-to-value.

---

## Issue 5: Frappe Framework Assumptions Unverified

### What the plan assumes

- `doc_events` supports wildcard `"*"` for capturing all DocType events
- All AI hooks fire reliably across normal and bulk operations
- `frappe.get_doc(...).cancel()` cleanly reverses any submitted document
- Frappe's background job (RQ) capacity is sufficient for AI workloads
- `frappe.get_cached_doc` behaves predictably across workers

### What needs verification

**`doc_events` wildcard**: The plan uses `"*"` as a universal DocType hook. Frappe may or may not support this syntax; if not, every DocType must be listed explicitly. Needs code-level verification and may change integration pattern materially.

**Hook reliability**: Hooks do NOT fire for:
- Bulk imports (`frappe.flags.in_import`)
- Data migrations (`frappe.flags.in_migrate`)
- Direct SQL updates (`frappe.db.sql("UPDATE ...")`)
- Patches applied at upgrade time
- Document operations with `flags.ignore_validate`

AI anomaly detection via hooks will miss these paths. Whether that's acceptable depends on capability.

**Cancel cascades**: Cancelling a submitted document is not always clean:
- Depends on document type
- May fail with "linked documents exist" error
- May require cascading cancel of dependent documents in specific order
- Period close boundaries prevent cancel
- Some DocTypes have custom cancel logic that doesn't reverse all side effects

**RQ capacity**: Frappe's default RQ setup has a few workers. AI workloads (embedding generation, async LLM calls, batch scans) can easily saturate. Requires:
- Dedicated AI queue separate from standard Frappe jobs
- Auto-scaling worker pool
- Backpressure management

**`frappe.get_cached_doc` across workers**: Cache is per-process. Multi-worker deployments need distributed cache (Redis); stale-cache issues are possible.

### Recommended revision

1. **Technical spike Phase 0**: validate each Frappe assumption with real code against a real ERPNext instance
2. **Document actual behavior** per assumption in an architecture decision record
3. **Adjust plan** where assumptions don't hold:
   - Explicit per-DocType hook registration if wildcard unsupported
   - Additional scan-based detection for paths that bypass hooks
   - Cancel-cascade documentation per DocType
   - Dedicated RQ infrastructure for AI
4. **Integration test suite** that exercises the hook integration in representative scenarios

---

## Issue 6: Embedding Model Lock-in

### What the plan missed

The choice of embedding model for the vector store is a **long-term commitment**:

- Changing embedding model requires **full re-embed of entire corpus**
- Re-embedding cost scales with data volume (every item, customer, supplier, every historical document)
- Vector store filter indexes must rebuild
- Downstream AI capabilities built on embeddings must revalidate

### Implications

**Choice matters**:
- Open-source embedding models (BGE, Nomic) — free, but may be weaker
- OpenAI `text-embedding-3-large` — capable, pricing stable but not guaranteed
- Voyage AI `voyage-3` — specialized for retrieval, strong quality
- Cohere embed-v3 — good multilingual support
- Local embeddings (sentence-transformers) — full control but quality varies

**Migration cost** is not planned for:
- Re-embed of full record corpus (inference cost per migration)
- Engineering labor per migration
- Validation: retrieval quality regression testing

### Recommended revision

1. **Embedding model evaluation** in Phase 0 — benchmark on ERPNext retrieval tasks
2. **Lock choice for an extended period** before considering migration
3. **Plan for migration** in ongoing operational considerations
4. **Version embeddings alongside records** — enables side-by-side migration validation
5. **Tenant-specific flexibility** — different tenants may have different embedding providers

---

## Summary of Technology Choice Issues

| Issue | Severity | Fix |
|-------|----------|-----|
| RAG overuse for structured data | Medium | Hybrid retrieval (structured first, RAG for unstructured) |
| Self-hosted as second-class | Medium | First-class option with decision matrix |
| Claude-default bias | Low | Per-capability evaluation; disclose author affinity |
| Build vs buy not evaluated | Medium | Decision matrix per capability; likely a material share becomes "buy" |
| Frappe assumptions unverified | Medium | Technical spike Phase 0; ADRs; integration tests |
| Embedding model lock-in | Low | Evaluate before commit; budget migration |

The net effect of addressing these: **lower scope, faster time-to-value, better-suited tools per job, less vendor lock-in**. Customer wins on all dimensions.
