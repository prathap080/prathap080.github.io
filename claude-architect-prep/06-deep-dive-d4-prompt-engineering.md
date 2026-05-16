# 06 — Domain 4: Prompt Engineering & Structured Output (20%)

> Maps to: CI/CD, Structured Data Extraction scenarios. This is where your LLM background pays off — much of this will be familiar.

---

## What This Domain Tests

Six task statements:
- 4.1 Explicit criteria over vague instructions
- 4.2 Few-shot prompting for consistency
- 4.3 Structured output via tool use + JSON schemas
- 4.4 Validation, retry, feedback loops
- 4.5 Batch processing strategies
- 4.6 Multi-instance / multi-pass review architectures

The unifying theme: **specify behavior precisely (criteria, schemas, examples), guarantee structure mechanically (tool_use), and run multiple passes when one isn't enough**.

---

## 4.1 — Explicit Criteria Over Vague Instructions

### The core principle

Vague instructions like "be conservative" or "only report high-confidence findings" do NOT meaningfully improve precision. They reduce *quantity* of findings but the precision-to-recall tradeoff stays similar.

**What works:** explicit categorical criteria.

### Bad → Good

❌ **Bad:** "Review the code and only report high-confidence issues."

✅ **Good:**
> "Report only issues in these categories:
> - **Bugs** that change program behavior incorrectly
> - **Security vulnerabilities** (injection, broken auth, secrets in code)
> - **Memory/resource leaks**
> 
> Do NOT report:
> - Style preferences
> - Minor naming inconsistencies
> - Patterns that are valid local conventions even if uncommon"

### The exam scenario pattern

> "Code review is producing too many false positives, undermining developer trust."

**Right:** Define explicit categorical criteria — what to report, what to skip — with examples for borderline cases.

**Wrong:** "Tell the model to be more conservative."
**Wrong:** "Have the model self-report confidence and filter at threshold."
**Wrong:** "Switch to a higher-tier model."

### Severity calibration with code examples

For consistent severity classification, give the model concrete code examples for each severity:

> **CRITICAL:** SQL injection, hardcoded secrets, unauthenticated PII access. Example: `db.query("SELECT * FROM users WHERE id=" + req.params.id)`
>
> **HIGH:** Missing input validation that could cause runtime crashes. Example: `JSON.parse(req.body)` with no try/catch.
>
> **MEDIUM:** Inefficient algorithms in hot paths. Example: Nested loop on a list that grows unbounded.
>
> **LOW:** Style/maintainability issues with no functional impact.

### Temporarily disabling false-positive categories

If one category is producing many false positives and undermining trust in *all* findings, **disable that category temporarily** while you improve its prompt — don't try to fix it in production where it's eroding credibility.

### 🔧 Practical Task 4.1
Take a code review prompt you'd write naturally. Then rewrite it with:
1. An explicit allow-list of categories to flag
2. An explicit deny-list of things NOT to flag
3. A severity rubric with code examples for each level

Run both versions on the same 5 PRs. Measure false positive rate.

---

## 4.2 — Few-Shot Prompting

### When few-shot is the right move

Few-shot examples are the **most effective technique** when:
- Detailed instructions alone produce inconsistent results
- You need a specific output format
- The task involves ambiguous-case judgment
- You want generalization to novel patterns

### How many examples

**2-4 targeted examples** beat 10 generic ones. Each example should:
- Cover an ambiguous case (not just easy ones)
- Show the **reasoning** for choosing one action over plausible alternatives
- Demonstrate the desired output format precisely

### Example: tool selection few-shot

```
EXAMPLE 1 (ambiguous case — order with implicit customer reference):
User: "Can you check on order ORD-789?"
Reasoning: User mentioned an order ID directly. Use lookup_order with the ID. Don't call get_customer first; the order ID alone is sufficient for an order status check.
Action: Call lookup_order(order_id="ORD-789")

EXAMPLE 2 (ambiguous case — refund implies need for verification first):
User: "Can I get a refund on order ORD-789?"
Reasoning: Refunds require verified customer identity per policy. Even though the user provided an order ID, I must call get_customer first to verify.
Action: Call get_customer first, then process the refund only after verification.
```

### Format demonstration

If you need consistent output format:

```
EXAMPLE OUTPUT FORMAT:
{
  "location": {"file": "auth.ts", "line": 42},
  "issue": "Password hashing uses MD5",
  "severity": "CRITICAL",
  "suggested_fix": "Use bcrypt or argon2 with appropriate cost factor"
}
```

Include 2-3 of these in the prompt and the model will reliably produce the same shape.

### Reducing hallucination in extraction

For document extraction with varied formats (inline citations vs. bibliographies, narrative vs. tabular data), few-shot examples **showing each variant** dramatically reduces empty/null extractions of fields that ARE present in the document.

### 🔧 Practical Task 4.2
Build an extraction task (e.g., extracting medication dosages from clinical notes). Try three approaches and compare:
1. Detailed prose instructions only
2. Add 2 generic examples
3. Replace with 3 targeted examples covering: standard format, abbreviated/informal format, missing dosage

Measure extraction accuracy and consistency.

---

## 4.3 — Structured Output via Tool Use + JSON Schemas

### Why tool_use + JSON schema is the gold standard

Asking the model to "respond in JSON format" produces output that often:
- Has syntax errors (trailing commas, missing quotes)
- Is wrapped in markdown code fences
- Has chatty preambles ("Here is the JSON:")

Using `tool_use` with a JSON schema as the tool's input parameters **eliminates JSON syntax errors entirely**. The model is forced to produce valid JSON conforming to the schema.

### What it does NOT eliminate

Structured output via tool_use eliminates **syntax** errors. It does NOT eliminate **semantic** errors:
- Line items that don't sum to the stated total
- Values placed in the wrong field
- Hallucinated values for required fields when source doesn't have them

This is critical for the exam — if the question is about syntax errors, the answer is tool_use + schema. If it's about semantic correctness, you need additional validation.

### tool_choice configurations

```python
# Auto: model decides whether to use the tool or respond with text
tool_choice = {"type": "auto"}

# Any: model MUST call some tool (any one) — prevents text response
tool_choice = {"type": "any"}

# Forced: model MUST call this specific tool first
tool_choice = {"type": "tool", "name": "extract_invoice"}
```

### Schema design — required vs. optional

❌ **Wrong:** Mark every field `required` — forces the model to fabricate values when source is missing them.

✅ **Right:** Mark fields **optional/nullable** when source documents may not contain them. The model returns `null` honestly instead of inventing values.

### The "other" + detail pattern

For extensible categories without forcing the model into a wrong bucket:

```json
{
  "type": "string",
  "enum": ["invoice", "receipt", "purchase_order", "credit_note", "other"]
},
"type_other_detail": {
  "type": "string",
  "description": "If type is 'other', describe the document type here"
}
```

Plus an `"unclear"` enum option for ambiguous cases.

### Format normalization in prompts

Even with strict schemas, source data is messy. Include normalization rules:

> "Output dates in ISO 8601 format (YYYY-MM-DD). If the source uses '12/05/2024' assume MM/DD/YYYY (US convention). If the source uses '2024-12-5' (no padding), pad to '2024-12-05'."

### 🔧 Practical Task 4.3
Build an invoice extraction tool:
1. Define a JSON schema with required fields (invoice_number, vendor) and optional fields (purchase_order_ref, payment_terms)
2. Use `tool_choice: {"type": "tool", "name": "extract_invoice"}` to force extraction
3. Test with a document missing optional fields — verify model returns null, not fabricated values
4. Add an `"other"` + detail enum for document type classification

---

## 4.4 — Validation, Retry, and Feedback Loops

### Retry with error feedback

When validation fails, don't just retry blindly. **Append the specific validation error to the prompt** so the model can self-correct:

```python
def extract_with_validation(document, schema, max_retries=3):
    messages = [{"role": "user", "content": document}]
    
    for attempt in range(max_retries):
        result = call_extraction_tool(messages)
        validation = validate_against_schema(result, schema)
        
        if validation.ok:
            return result
        
        # Feed the error back for self-correction
        messages.extend([
            {"role": "assistant", "content": [{"type": "tool_use", ...}]},
            {"role": "user", "content": (
                f"The extraction failed validation:\n"
                f"{validation.errors}\n\n"
                f"Please review the document and correct these specific issues."
            )},
        ])
    
    raise ExtractionFailed("Max retries exceeded", last_errors=validation.errors)
```

### When retries succeed vs. fail

| Error type | Retry helps? |
|------------|-------------|
| Format mismatch (e.g., date in wrong format) | ✅ Yes |
| Schema validation error (semantic, e.g., total doesn't sum) | ✅ Often |
| Required information genuinely absent from source | ❌ No — retry won't conjure data |

The exam tests this distinction. **If the information isn't in the source, more retries don't help — you need a different document or to mark the field optional.**

### Self-correction validation flows

For semantic validation, design schemas with **paired fields**:

```json
{
  "stated_total": 1500.00,    // What the document says
  "calculated_total": 1485.00, // Sum of line items
  "totals_match": false        // Computed flag
}
```

If they don't match, you've **detected** a semantic issue without needing a separate validator pass.

Similarly:
```json
{
  "primary_value": "...",
  "alternative_value": "...",
  "conflict_detected": true,
  "conflict_explanation": "Source page 3 states X, source page 7 states Y"
}
```

### Tracking patterns for analysis

Add a `detected_pattern` field to each finding so you can analyze what triggers false positives:

```json
{
  "issue": "Possible SQL injection",
  "severity": "HIGH",
  "detected_pattern": "string concatenation in db.query() call",
  "code_snippet": "..."
}
```

When developers dismiss findings, you can group by `detected_pattern` and identify which patterns produce systematic false positives.

### 🔧 Practical Task 4.4
1. Build the retry-with-feedback loop above
2. Test with a deliberately broken extraction (wrong field types) — verify it self-corrects
3. Test with a document missing required information — verify it doesn't fabricate
4. Add stated/calculated/match fields to your schema for self-validation

---

## 4.5 — Batch Processing Strategies

### Message Batches API characteristics (memorize these)

- **50% cost discount** vs. real-time API
- **Up to 24-hour processing window** — no guaranteed latency SLA
- **`custom_id`** field correlates each request with its response
- **No multi-turn tool calling** within a single request — you can't execute tools mid-batch and feed results back

### When batch is appropriate

✅ **Yes for:**
- Overnight reports
- Weekly audits  
- Nightly test generation
- Bulk classification of historical documents
- Backfill processing
- Anything where latency tolerance ≥ 24 hours

❌ **No for:**
- Pre-merge code review (developers wait)
- Interactive customer support
- Anything blocking a user
- Multi-turn agentic loops requiring intermediate tool execution

### The exam's canonical scenario

> "Reduce API costs. Real-time calls power (1) blocking pre-merge checks and (2) overnight technical debt reports. Manager proposes switching both to batch."

**Right answer:** Batch for the technical debt reports only; keep real-time for the blocking pre-merge checks.

**Wrong:** "Switch both with status polling" — relying on "often faster" completion isn't acceptable for blocking workflows.
**Wrong:** "Keep both real-time to avoid result ordering issues" — batch results correlate via `custom_id`, this isn't a real concern.
**Wrong:** "Switch both with timeout fallback to real-time" — adds complexity when matching each API to its appropriate use case is simpler.

### Batch SLA math

If your overall SLA is 30 hours and batch processing is up to 24 hours:
- You have 6 hours of buffer
- Submit batches at most every **6 hours** (so worst case = 24h batch + 6h buffer = 30h)
- Or submit every 4 hours for safety margin

### Handling batch failures

When some documents in a batch fail:
1. Identify failures by `custom_id`
2. Resubmit only the failed documents
3. Apply modifications (e.g., chunk documents that exceeded context limits)
4. Don't resubmit the whole batch

### Reducing iterative resubmission cost

Before batch-processing 10,000 documents, **refine the prompt on a 50-100 document sample** to maximize first-pass success rate. Otherwise you'll resubmit failures multiple times, eating into the cost savings.

### 🔧 Practical Task 4.5
1. Set up a small batch (10 documents) with the Message Batches API
2. Include some documents designed to fail (oversized, malformed) and verify only those resubmit
3. Calculate: at your batch frequency, what's your worst-case end-to-end latency?
4. Compare cost/latency of running the same workload synchronously

---

## 4.6 — Multi-Instance and Multi-Pass Review

### The self-review limitation

A model that just generated code retains its reasoning context. It's biased toward seeing the code as correct because it can recall *why* it made each decision. This bias means:
- Self-review (same session) catches fewer subtle issues
- Even extended thinking in the same session doesn't fully resolve this
- "Self-report confidence" prompts don't fix it either

### The fix: independent review instance

Spawn a **separate Claude instance** with NO prior reasoning context. It sees the code with fresh eyes and is much more likely to catch issues the original missed.

```python
# Generate
generation_response = await generate_code(spec)

# Review with INDEPENDENT instance (new session, no prior context)
review_response = await review_code(
    code=generation_response.code,
    review_criteria=criteria,
    # Fresh client, fresh session, fresh context
)
```

### Multi-pass review for large changes

For PRs with many files, single-pass review causes **attention dilution** (covered in Domain 1). The fix is multi-pass:

1. **Per-file local pass** — focused review of each file individually
2. **Cross-file integration pass** — examine data flow, type compatibility, naming consistency across files

### Why "use a larger context window" doesn't fix this

A larger context window admits more tokens. It does NOT improve attention quality across those tokens. The model can hold 14 files in context but still pay uneven attention to each. **Splitting by file forces equal attention.**

### Confidence-aligned review routing

For verification passes, have the model **self-report confidence alongside each finding**. Low-confidence findings get routed to humans; high-confidence ones get auto-applied. This is calibrated routing, not "filter by self-reported confidence" (which is the anti-pattern from Section 4.1).

The distinction: confidence-as-routing (good) vs. confidence-as-quality-gate (bad). Confidence-as-routing assumes some findings need humans; confidence-as-quality-gate assumes the model is well-calibrated, which it isn't.

### 🔧 Practical Task 4.6
1. Generate a non-trivial function (e.g., a JWT validator) in one session
2. Ask the same session to review its own code; capture findings
3. Open a fresh session and ask it to review the same code; compare findings
4. For a PR with 10+ files, run single-pass and multi-pass review; compare which catches more issues

---

## Domain 4 Summary — The Exam-Ready Cheat Sheet

| Concept | The right answer pattern |
|---------|--------------------------|
| Too many false positives | Explicit allow/deny lists of categories, NOT "be conservative" |
| Inconsistent output format | 2-4 few-shot examples showing exact format |
| Ambiguous-case judgment | Few-shot examples with reasoning shown |
| Need guaranteed JSON syntax | tool_use + JSON schema |
| Need guaranteed semantic correctness | tool_use + paired/computed fields + validation |
| Force a specific tool to run | `tool_choice: {"type": "tool", "name": "..."}` |
| Force any tool (no text response) | `tool_choice: {"type": "any"}` |
| Source document might lack a field | Mark field optional/nullable, let model return null |
| Extensible enum categorization | "other" + detail field, plus "unclear" for ambiguous |
| Validation failed (format/structure) | Retry with specific error in prompt — likely succeeds |
| Required info absent from source | Don't retry — can't conjure data |
| Detect semantic errors (totals don't sum) | Add stated_total + calculated_total + match flag to schema |
| Track false positive patterns | Add `detected_pattern` field for analysis |
| Non-blocking, latency-tolerant workload | Message Batches API (50% off, ≤24h) |
| Blocking pre-merge / interactive | Real-time API, NOT batch |
| Batch failures | Resubmit only failed `custom_id`s with modifications |
| Reviewing your own generated code | Independent instance, NOT same session |
| Reviewing 14-file PR with inconsistent depth | Per-file passes + cross-file integration pass |
| Self-reported confidence as quality gate | Anti-pattern (poorly calibrated) |
| Confidence as routing signal to humans | OK — routes attention, doesn't gate quality |

---

## Domain 4 Drill Questions (For Self-Test)

1. Code review has high false-positive rate; "be conservative" didn't help. Right fix?
2. Your extraction tool returns valid JSON syntax but line items don't sum to the total. What does this tell you?
3. Source documents sometimes lack a `purchase_order` field. Schema marks it required. What's wrong?
4. You need to process 50,000 documents in batch but pre-merge checks must stay fast. Strategy?
5. Single-pass review of a 14-file PR gives contradictory findings. Larger context window?
6. The same session that wrote a complex function reviews it and approves obvious bugs. Why?

(Answers: 1) Explicit categorical criteria with examples 2) Tool_use eliminates syntax errors but not semantic — add stated/calculated paired fields 3) Optional/nullable, otherwise model fabricates 4) Batch the bulk job; keep pre-merge real-time 5) No — attention dilution isn't context-size, split into per-file + integration passes 6) Self-review bias from retained reasoning context — use independent instance)
