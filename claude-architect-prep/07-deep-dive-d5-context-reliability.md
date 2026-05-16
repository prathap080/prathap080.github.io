# 07 — Domain 5: Context Management & Reliability (15%)

> Smallest by weight but threads through every scenario. Maps to: Customer Support, Multi-Agent Research, Structured Data Extraction. Many Domain 1 questions secretly test these patterns too.

---

## What This Domain Tests

Six task statements:
- 5.1 Conversation context preservation
- 5.2 Escalation and ambiguity resolution
- 5.3 Error propagation in multi-agent systems
- 5.4 Context management in large codebase exploration
- 5.5 Human review workflows and confidence calibration
- 5.6 Information provenance in multi-source synthesis

The unifying theme: **structure outlives summarization**. Anything important must be in a structured form that survives compression, handoffs, and time.

---

## 5.1 — Preserving Critical Information Across Long Interactions

### The lossy-summarization problem

Long conversations get summarized to fit context budgets. Summarization is **lossy in predictable ways**:
- Numerical values become "approximately" or "around"
- Specific dates become "earlier this week"
- Customer-stated expectations become "the customer wants resolution"
- Order/policy IDs disappear

For a customer support agent on a 30-message conversation, this causes the agent to "forget" specifics it needs to act correctly.

### The fix: persistent case facts

Extract structured facts into a **separate context layer** that travels with every prompt, outside the summarized history:

```python
case_facts = {
    "customer_id": "CUS-12345",
    "verified_identity": True,
    "verification_method": "order_id_plus_email",
    "primary_issue": "damaged_item",
    "secondary_issues": ["billing_dispute", "address_change_confirmation"],
    "order_id": "ORD-78901",
    "order_total": 149.99,
    "refund_amount_authorized": 89.99,
    "case_opened_at": "2025-09-12T14:30:00Z",
    "policy_exceptions_invoked": ["competitor_match_one_time"],
}

system_prompt = f"""
You are a customer support agent.

CASE FACTS (current state — do not summarize away):
{json.dumps(case_facts, indent=2)}

Recent conversation summary:
{summarized_history}

Current customer message: {user_message}
"""
```

This is **your CRO pattern from the medical symptom checker, applied to customer support.**

### "Lost in the middle" effect

Models are reliable at processing information at the **beginning and end** of long inputs. Information in the middle may be omitted from reasoning.

**Mitigations:**
- Place key findings summaries at the **beginning** of aggregated inputs
- Organize detailed results with **explicit section headers**
- For very long content, repeat critical facts at both top and bottom

### Trimming verbose tool outputs

A typical `lookup_order` returns 40+ fields, but only 5 are relevant to a return request (order_id, status, items, total, return_eligibility). Storing all 40 in conversation context disproportionately consumes tokens.

**Pattern:** apply a `PostToolUse` hook that strips to relevant fields before storing in conversation history.

```python
async def trim_order_response(tool_input, tool_response, context):
    if context.tool_name == "lookup_order":
        # Keep only what's relevant for downstream reasoning
        trimmed = {
            "order_id": tool_response["order_id"],
            "status": tool_response["status"],
            "total": tool_response["total"],
            "items": [
                {"sku": i["sku"], "name": i["name"], "qty": i["qty"]}
                for i in tool_response["items"]
            ],
            "return_eligible": tool_response["return_eligible"],
        }
        return {"hookSpecificOutput": {"toolResult": trimmed}}
    return {}
```

### Subagent metadata contracts

When subagents return findings to the coordinator, **require structured metadata in their outputs**:
- Source URL / document name
- Page number / section reference
- Timestamp (publication or collection date)
- Confidence / methodology context

Otherwise the coordinator and downstream synthesis lose attribution.

### Modifying upstream agents for downstream context budgets

If the synthesis agent has a tight context budget, modify the search/analysis agents to return **structured data (key facts + citations + relevance scores) instead of verbose content + reasoning chains.**

### 🔧 Practical Task 5.1
1. Build a "case facts" extractor that pulls structured facts from each customer interaction and persists them
2. Modify your prompt template to inject case facts at the top of every system prompt
3. Run a 20-message simulated conversation with progressive summarization disabled vs. enabled — measure how often the agent loses specific facts
4. Add a `PostToolUse` hook that trims a verbose API response to relevant fields

---

## 5.2 — Escalation and Ambiguity Resolution

### Three valid escalation triggers

1. **Customer explicitly requests a human** — honor immediately, don't try to investigate first
2. **Policy exceptions or gaps** — when policy is silent on the customer's specific request
3. **Inability to make meaningful progress** — agent has tried and is stuck

### Three invalid escalation signals (anti-patterns)

❌ **Sentiment-based** — angry customer doesn't mean complex case
❌ **Self-reported confidence scores** — LLM confidence is poorly calibrated; the agent is *most* confidently wrong on the cases that need humans
❌ **Heuristic complexity assessment** — "this looks complicated" isn't a reliable signal

### Honor explicit human requests immediately

> Customer: "Just give me a human."

Wrong: "Let me try to help you first..." (frustrating, ignores the explicit request)
Right: Acknowledge, escalate immediately with a structured handoff payload.

### Acknowledge frustration when issue is in scope

> Customer: "This is the worst experience I've ever had! Refund my $20 now."

If the refund is straightforward and within agent capability:
- Acknowledge the frustration ("I understand this has been frustrating")
- Offer immediate resolution ("I can process the refund now")
- Escalate ONLY if the customer reiterates they want a human

### Policy gaps

> Customer requests competitor price match. Your policy only addresses own-site price adjustments — silent on competitors.

This is an escalation case. Don't:
- Make up a policy ("I'm sure we don't do that")
- Refuse without explanation
- Pretend the policy covers it

Do: escalate with the structured payload showing what was investigated and why human judgment is needed.

### Multiple match disambiguation

> `lookup_customer("John Smith")` returns 3 matches.

Wrong: pick the most recent / first / highest-spending heuristically.
Right: ask the customer for additional identifiers (email, postal code, last order date).

### 🔧 Practical Task 5.2
1. Add explicit escalation criteria to a customer support prompt with 3 few-shot examples (one for each valid trigger)
2. Test with: explicit human request, sentiment-based frustration with simple issue, policy gap, multi-match ambiguity
3. Verify the agent escalates the right ones and resolves the right ones
4. Build the structured handoff payload format

---

## 5.3 — Error Propagation in Multi-Agent Systems

### What good error context looks like

When a subagent fails, the coordinator needs enough context to make recovery decisions. The right shape:

```python
{
    "subagent": "web_searcher",
    "failure_type": "timeout",
    "attempted_query": "Renaissance art markets pre-1500",
    "partial_results": [
        {"title": "Florentine guilds...", "url": "...", "fetched": True},
        {"title": "Venetian merchants...", "url": "...", "fetched": False},
    ],
    "alternatives_attempted": [
        {"approach": "narrowed query", "result": "still timed out"},
    ],
    "alternatives_suggested": [
        "Try a more specific date range",
        "Try a different source type (academic vs. blog)",
    ],
}
```

### Anti-patterns

❌ **"Search unavailable"** — generic, hides recovery context

❌ **Catching exceptions and returning empty success** — coordinator can't distinguish "no matches found" from "lookup never executed"

❌ **Propagating exceptions up to terminate the entire workflow** — a single subagent failure shouldn't kill the whole research task

### Distinguishing access failure from valid empty result

```python
# Valid success — query ran, no matches
{"isError": False, "results": [], "metadata": {"query_ran": "..."}}

# Access failure — query never completed
{"isError": True, "errorCategory": "transient", "isRetryable": True, ...}
```

The coordinator's recovery logic differs sharply between these. Generic statuses collapse them and prevent intelligent routing.

### Local recovery before propagation

Subagents should:
1. Attempt local recovery for transient failures (retry, narrower query)
2. Propagate ONLY errors they cannot resolve locally
3. Include partial results when propagating

### Coverage annotations in synthesis output

When the synthesis agent compiles a report from partial subagent results, it should annotate:

```markdown
## Coverage Status

✅ Well-supported sections: Visual arts (3 sources), Music industry (4 sources)
⚠️  Partial coverage: Film production (1 source — verify before publishing)
❌ Coverage gap: Television writing (web searcher unavailable; recommend re-running)
```

This makes downstream consumers aware of what's reliable vs. what needs follow-up.

### 🔧 Practical Task 5.3
1. Build a subagent that simulates a flaky tool (random timeouts)
2. Wrap it with local recovery (1 retry on transient errors)
3. On unrecoverable errors, propagate the structured context
4. Have the coordinator make recovery decisions based on the context type
5. Generate a synthesis output with explicit coverage annotations

---

## 5.4 — Large Codebase Context Management

### Context degradation in extended sessions

In sessions that exceed a few hundred tool calls, the model starts:
- Giving inconsistent answers to the same question
- Referencing "typical patterns" rather than the specific classes/functions discovered earlier
- Confusing details between different parts of the codebase

### Three mitigations

#### 1. Scratchpad files

Have the agent maintain a `.claude/scratchpad/findings.md` file recording key findings:

```markdown
# Codebase findings — payment module

## Entry points
- `app/api/payments/route.ts` — POST handler
- `app/api/refunds/route.ts` — POST handler (calls payments service)

## Key classes
- `PaymentProcessor` (lib/payments/processor.ts) — handles Stripe integration
- `RefundService` (lib/payments/refunds.ts) — depends on PaymentProcessor

## Conventions
- All amounts in cents (integers), never dollars (floats)
- All Stripe calls wrapped in retry logic
```

The agent references this for subsequent questions instead of re-discovering.

#### 2. Subagent delegation for verbose discovery

> "Find all test files," "Trace the refund flow's dependencies," "Map the API routing structure"

Each of these spawns enormous tool output. Delegate to a subagent (or use `context: fork` skill) so the verbose discovery happens in isolation. The main agent receives only the summary.

#### 3. `/compact` for in-session reduction

When extended exploration has filled context with verbose tool results, run `/compact` to reduce. This summarizes earlier turns while preserving the most recent.

### Crash recovery via structured state

For long-running multi-agent workflows, design crash recovery using **state manifests**:

```python
# Each agent exports state to a known location
state_manifest = {
    "session_id": "research-2025-09-12-001",
    "phase": "synthesis",
    "completed_subagents": {
        "web_searcher": {"output_path": "./state/web_results.json", "status": "complete"},
        "doc_analyzer": {"output_path": "./state/doc_analysis.json", "status": "complete"},
        "synthesizer": {"output_path": None, "status": "in_progress"},
    },
    "checkpoint_at": "2025-09-12T14:35:00Z",
}

# On resume, the coordinator loads the manifest and injects state into prompts
```

### Phase summaries before phase transitions

Before spawning subagents for the next phase, **summarize key findings from the current phase** and inject them into the new subagents' initial context. Don't let the subagents discover what's already known.

### 🔧 Practical Task 5.4
On a medium-sized open-source repo:
1. Have Claude maintain a `findings.md` scratchpad while exploring
2. After 30 minutes of exploration, ask a question requiring memory of earlier findings — verify the scratchpad helps
3. Trigger `/compact` and observe what gets preserved vs. summarized
4. Implement a crash recovery flow with state manifests

---

## 5.5 — Human Review and Confidence Calibration

### The aggregate accuracy trap

> "Our extraction system has 97% accuracy."

This number can hide:
- 99% accuracy on common document types, 50% on rare ones
- 99% on the `vendor_name` field, 60% on `payment_terms`
- 98% on legible scans, 70% on poor-quality scans

**Always validate by document type and field segment** before reducing human review.

### Stratified random sampling

Instead of random sampling (which oversamples common types), use **stratified sampling**:
- 5% of "type A" documents
- 5% of "type B"
- 5% of "type C" (rare, but you want signal)

This catches novel error patterns and gives error rate confidence per stratum.

### Field-level confidence calibration

Have the model output **per-field confidence scores**:

```json
{
  "vendor_name": {"value": "Acme Corp", "confidence": 0.97},
  "invoice_total": {"value": 1499.50, "confidence": 0.92},
  "payment_terms": {"value": "NET 30", "confidence": 0.45}
}
```

**Calibrate the threshold** using a labeled validation set. If "0.45 confidence" corresponds to "60% accurate" empirically, you can route everything below that to humans.

### Routing limited reviewer capacity

Reviewers are scarce. Route to them:
- Low-confidence extractions
- Ambiguous source documents (e.g., faxed scans)
- Contradictory data (page 3 says X, page 7 says Y)
- Documents from new vendors (no calibration data yet)

### 🔧 Practical Task 5.5
1. Run extraction on 100 documents; capture per-field confidence
2. Have humans label a subset; compare model confidence to actual accuracy
3. Find the calibration curve (model confidence vs. accuracy)
4. Set thresholds based on the curve
5. Stratify the sample by document type and verify accuracy doesn't drop in any segment

---

## 5.6 — Provenance and Multi-Source Synthesis

### The provenance loss problem

When synthesis agents combine findings from multiple sources, source attribution is **easily lost during summarization**. The output reads as "studies show X" without saying which studies.

### The fix: structured claim-source mappings

Each subagent must output structured findings:

```json
{
  "claim": "AI assistants reduce code review time by 30%",
  "source_url": "https://...",
  "source_document": "Dev Productivity Study 2025",
  "source_excerpt": "On average, teams using AI-assisted review completed reviews 30% faster (n=234, p<0.01)",
  "publication_date": "2025-06-15",
  "methodology_context": "Survey-based, self-reported metrics",
  "confidence": 0.85
}
```

The synthesis agent then **preserves and merges these mappings** into the final report — never reduces them to "studies show".

### Conflicting statistics

When two credible sources give different numbers:

❌ **Wrong:** Pick one arbitrarily, present as fact.
✅ **Right:** Annotate the conflict with attribution:

> "Studies report varying time-savings: Reuters (2025) found a 30% reduction (survey methodology, n=234), while McKinsey (2025) reported 18% (instrumented telemetry, n=1,200). The methodology differences may explain the discrepancy."

### Temporal context

A source from 2020 about "current AI capabilities" is not contradicting a 2025 source — they're describing different time points. **Require subagents to include publication/collection dates** so temporal differences aren't misread as factual contradictions.

### Distinguishing well-established from contested

Synthesis output should structure findings:

```markdown
## Well-established findings
- AI assistants increase coding throughput (multiple studies, consistent direction, varying magnitudes)
- Code review quality with AI assistance is comparable to without

## Contested findings
- Magnitude of throughput increase (range: 10%-40% across studies)
- Effect on long-term maintainability (insufficient data, mixed signals)

## Methodological notes
- Most studies use self-reported metrics; few use instrumented telemetry
```

### Rendering by content type

Don't force a uniform format on heterogeneous content:
- Financial data → tables
- News/narrative → prose
- Technical findings → structured lists
- Statistical comparisons → numbered tables with footnoted methodology

### 🔧 Practical Task 5.6
1. Build a research workflow where 3 subagents return findings on the same topic
2. Define a structured output schema requiring source URL, excerpt, date, methodology
3. Have synthesis combine them, preserving claim-source mappings
4. Inject conflicting data deliberately; verify the synthesis annotates rather than picks
5. Inject sources from different time periods; verify they're not treated as contradictions

---

## Domain 5 Summary — The Exam-Ready Cheat Sheet

| Concept | The right answer pattern |
|---------|--------------------------|
| Specifics getting lost in summarization | Extract structured "case facts" persisted alongside summaries |
| Lost in the middle effect | Place key findings at start; explicit section headers |
| 40-field tool response, only 5 relevant | `PostToolUse` hook to trim before storing |
| Subagent loses source attribution | Structured metadata required in every subagent output |
| Customer asks for human | Honor immediately; structured handoff |
| Frustrated customer with simple issue | Acknowledge + offer resolution; escalate only if reiterated |
| Policy is silent on the request | Escalate (policy gap) |
| Multiple customer matches | Ask for additional identifier — don't pick heuristically |
| Sentiment as escalation trigger | Anti-pattern |
| LLM self-reported confidence as escalation gate | Anti-pattern (poorly calibrated) |
| Subagent timeout | Structured error context (failure type, partial results, alternatives) |
| Catching errors and returning empty success | Anti-pattern — masks failure |
| Single subagent failure terminating workflow | Anti-pattern — coordinator should attempt recovery |
| Distinguish access failure from empty results | Different fields for `isError` vs. valid empty |
| Long codebase exploration losing memory | Scratchpad files + subagent delegation + `/compact` |
| Multi-phase workflow crash recovery | Structured state manifests, coordinator loads on resume |
| 97% accuracy boast | Validate per document type, per field segment |
| Random sampling biases toward common types | Stratified sampling |
| Field-level confidence | Calibrate thresholds against labeled validation set |
| Source attribution lost in synthesis | Structured claim-source mappings preserved through pipeline |
| Conflicting statistics from two sources | Annotate the conflict with both attributions and methodology |
| Sources from different dates | Require dates in structured output; don't treat as contradiction |
| Heterogeneous content rendered uniformly | Render by type — tables for data, prose for narrative |

---

## Domain 5 Drill Questions (For Self-Test)

1. After 25 messages, your support agent forgets the customer's verified ID. Right fix?
2. Customer says "just escalate me." Agent investigates first. What's wrong?
3. Subagent timeout — coordinator gets "search unavailable." Why insufficient?
4. After 200 tool calls, agent gives inconsistent answers about a class it discovered. Fix?
5. Two credible sources report different statistics. Synthesis picks one. What's the correct behavior?

(Answers: 1) Persistent case-facts block injected outside summarized history 2) Honor explicit human requests immediately; investigate later if at all 3) Generic status hides recovery context — need structured failure type, partial results, alternatives 4) Scratchpad files + delegate verbose discovery to subagents + `/compact` 5) Annotate the conflict with both source attributions and methodology context)
