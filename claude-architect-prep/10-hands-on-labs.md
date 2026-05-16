# 10 — Hands-On Labs

> Four progressive labs that exercise every domain. **Set aside 8-10 hours total** spread across the study month. Each lab maps to one of the official scenarios. Code is written in Python using the Claude Agent SDK.

---

## Setup (Once, Before Any Lab)

```bash
# Python 3.10+ required
pip install claude-agent-sdk anthropic

# Set your API key
export ANTHROPIC_API_KEY="sk-ant-..."

# Verify
python -c "from claude_agent_sdk import query; print('OK')"
```

Create a working directory:

```bash
mkdir -p ~/cca-labs/{lab1,lab2,lab3,lab4} && cd ~/cca-labs
```

---

## Lab 1 — Customer Support Resolution Agent (D1, D2, D5)

**Scenario:** Build a customer support agent that handles refunds with verified identity, structured tool errors, and escalation. Maps to **Scenario 1: Customer Support Resolution Agent** on the exam.

**You'll exercise:**
- Agentic loop with tool use (D1.1)
- Programmatic prerequisite enforcement via PreToolUse hook (D1.4)
- Structured error responses (D2.2)
- Case-facts persistence (D5.1)
- Escalation triggers (D5.2)

### Step 1 — Define the tools

```python
# lab1/tools.py
from claude_agent_sdk import tool

# Simulated database
CUSTOMERS = {
    "CUS-12345": {"name": "Alex Kumar", "email": "alex@example.com", "tier": "premium"},
    "CUS-67890": {"name": "Sam Lee", "email": "sam@example.com", "tier": "standard"},
}

ORDERS = {
    "ORD-001": {"customer_id": "CUS-12345", "total": 89.99, "status": "delivered"},
    "ORD-002": {"customer_id": "CUS-12345", "total": 750.00, "status": "delivered"},
    "ORD-003": {"customer_id": "CUS-67890", "total": 45.00, "status": "in_transit"},
}

@tool(
    "get_customer",
    "Verify and retrieve customer profile by customer ID. Returns name, email, tier, and verification status. Returns structured error if customer not found. Always call this BEFORE any operation that requires verified identity.",
    {"customer_id": str}
)
async def get_customer(args):
    customer_id = args["customer_id"]
    customer = CUSTOMERS.get(customer_id)
    if not customer:
        return {
            "content": [{"type": "text", "text": str({
                "isError": True,
                "errorCategory": "validation",
                "isRetryable": False,
                "errorCode": "CUSTOMER_NOT_FOUND",
                "humanReadable": f"No customer found with ID {customer_id}.",
            })}]
        }
    return {
        "content": [{"type": "text", "text": str({
            "isError": False,
            "customer_id": customer_id,
            "verified": True,
            **customer,
        })}]
    }

@tool(
    "lookup_order",
    "Retrieve order details by order ID (format: ORD-XXX). Returns order status, total, and customer ID. Use only AFTER customer identity has been verified via get_customer.",
    {"order_id": str}
)
async def lookup_order(args):
    order_id = args["order_id"]
    order = ORDERS.get(order_id)
    if not order:
        return {
            "content": [{"type": "text", "text": str({
                "isError": False,  # not finding an order is not an error
                "results": None,
                "metadata": {"query": order_id, "matched": 0},
            })}]
        }
    return {
        "content": [{"type": "text", "text": str({
            "isError": False,
            "order_id": order_id,
            **order,
        })}]
    }

@tool(
    "process_refund",
    "Initiate a refund for a verified customer. Refunds over $500 are blocked and require human escalation. Inputs: order_id, refund_amount.",
    {"order_id": str, "refund_amount": float}
)
async def process_refund(args):
    order_id = args["order_id"]
    amount = args["refund_amount"]
    # Note: amount > 500 is blocked by the PreToolUse hook, not here
    return {
        "content": [{"type": "text", "text": str({
            "isError": False,
            "refund_id": f"REF-{order_id}-001",
            "amount_refunded": amount,
            "status": "processed",
        })}]
    }

@tool(
    "escalate_to_human",
    "Escalate to a human agent with a structured handoff payload. Use when (a) customer explicitly requests a human, (b) policy is silent on the request, (c) the issue exceeds agent capabilities (e.g., refund > $500).",
    {
        "customer_id": str,
        "issue_summary": str,
        "investigation_completed": list,
        "recommended_action": str,
    }
)
async def escalate_to_human(args):
    return {
        "content": [{"type": "text", "text": str({
            "escalated": True,
            "handoff_id": f"HO-{args['customer_id']}-001",
            **args,
        })}]
    }
```

### Step 2 — Define the prerequisite hook

```python
# lab1/hooks.py
from claude_agent_sdk import HookMatcher

verified_customers = set()

async def enforce_verification(input_data, tool_use_id, context):
    """PreToolUse hook — block refunds and order lookups until customer is verified."""
    tool_name = input_data.get("tool_name")
    
    if tool_name == "get_customer":
        return {}  # allow
    
    if tool_name in ("lookup_order", "process_refund"):
        if not verified_customers:
            return {
                "hookSpecificOutput": {
                    "permissionDecision": "deny",
                    "permissionDecisionReason": (
                        "You must call get_customer first to verify the customer's identity "
                        "before looking up orders or processing refunds."
                    ),
                }
            }
    
    if tool_name == "process_refund":
        amount = input_data.get("tool_input", {}).get("refund_amount", 0)
        if amount > 500:
            return {
                "hookSpecificOutput": {
                    "permissionDecision": "deny",
                    "permissionDecisionReason": (
                        f"Refund of ${amount:.2f} exceeds the $500 limit. "
                        f"Use escalate_to_human with a structured handoff payload."
                    ),
                }
            }
    
    return {}

async def capture_verification(input_data, tool_use_id, context):
    """PostToolUse hook — record verified customer IDs."""
    if input_data.get("tool_name") == "get_customer":
        result_text = input_data.get("tool_response", {}).get("content", [{}])[0].get("text", "")
        if "CUS-" in result_text and "verified': True" in result_text:
            # Extract the customer_id from the result
            import re
            match = re.search(r"CUS-\d+", result_text)
            if match:
                verified_customers.add(match.group(0))
    return {}
```

### Step 3 — Run the agent

```python
# lab1/main.py
import asyncio
import json
from claude_agent_sdk import (
    ClaudeSDKClient, ClaudeAgentOptions, HookMatcher, create_sdk_mcp_server
)
from tools import get_customer, lookup_order, process_refund, escalate_to_human
from hooks import enforce_verification, capture_verification

# Case facts (Domain 5 pattern) — persist across turns
case_facts = {
    "session_started": "2026-05-03T10:00:00Z",
    "customer_verified": False,
    "issues_raised": [],
    "actions_taken": [],
}

async def main():
    server = create_sdk_mcp_server(
        name="support_tools",
        version="1.0.0",
        tools=[get_customer, lookup_order, process_refund, escalate_to_human],
    )
    
    options = ClaudeAgentOptions(
        system_prompt=f"""
You are a customer support agent.

CASE FACTS (current state — do not summarize away):
{json.dumps(case_facts, indent=2)}

ESCALATION TRIGGERS (escalate immediately if any of these occur):
- Customer explicitly requests a human ("just give me a manager", etc.)
- Policy is silent on the customer's request
- Refund exceeds $500 (system-blocked; use escalate_to_human)
- Multiple lookup matches require disambiguation that customer can't provide

RESOLUTION FIRST: If the issue is in scope and within policy, resolve it 
even if the customer is frustrated. Acknowledge frustration, then act.
""",
        mcp_servers={"support": server},
        allowed_tools=[
            "mcp__support__get_customer",
            "mcp__support__lookup_order",
            "mcp__support__process_refund",
            "mcp__support__escalate_to_human",
        ],
        hooks={
            "PreToolUse": [HookMatcher(matcher="*", hooks=[enforce_verification])],
            "PostToolUse": [HookMatcher(matcher="*", hooks=[capture_verification])],
        },
    )
    
    test_cases = [
        # Case 1: Should succeed
        "Customer CUS-12345 wants to refund order ORD-001 ($89.99) — item arrived damaged.",
        # Case 2: Should be blocked by hook (refund > $500)
        "Customer CUS-12345 wants to refund order ORD-002 ($750) — change of mind.",
        # Case 3: Should be blocked initially (no verification)
        "Look up order ORD-003 status.",
        # Case 4: Customer asks for human
        "Customer says: 'Just give me a manager. I don't want to deal with a bot.'",
    ]
    
    for i, prompt in enumerate(test_cases, 1):
        print(f"\n{'='*60}\nTEST {i}: {prompt}\n{'='*60}")
        async with ClaudeSDKClient(options=options) as client:
            await client.query(prompt)
            async for msg in client.receive_response():
                print(msg)

asyncio.run(main())
```

### Lab 1 success criteria

✅ Test 1 succeeds: customer verified → order looked up → refund processed
✅ Test 2: hook blocks the > $500 refund → agent escalates with structured handoff
✅ Test 3: hook blocks order lookup → agent calls `get_customer` first
✅ Test 4: agent escalates immediately without trying to resolve

### Lab 1 reflection questions

1. What happens if you remove the PreToolUse hook? Does the prompt instruction reliably enforce verification?
2. What happens to the `verified_customers` set across multiple `ClaudeSDKClient` sessions? How would you persist it?
3. Modify the hook to also block refunds for orders not belonging to the verified customer.

---

## Lab 2 — Multi-Agent Research System (D1, D2, D5)

**Scenario:** Build a research coordinator that spawns parallel search subagents, then a synthesizer that preserves source attribution. Maps to **Scenario 3: Multi-Agent Research System**.

**You'll exercise:**
- Coordinator-subagent architecture (D1.2)
- Parallel subagent invocation (D1.3)
- Tool scoping per agent (D2.3)
- Source provenance through synthesis (D5.6)

### Step 1 — Build the coordinator with two specialized subagents

```python
# lab2/main.py
import asyncio
from claude_agent_sdk import (
    ClaudeSDKClient, ClaudeAgentOptions, AgentDefinition
)

async def main():
    options = ClaudeAgentOptions(
        system_prompt="""
You are a research coordinator. Your job is to:

1. DECOMPOSE the research question into 3-4 distinct subtopics
   (Don't just route the literal question to a single search agent.
    Break it into angles that, together, give comprehensive coverage.)

2. INVOKE multiple search subagents IN PARALLEL by emitting multiple
   Agent tool calls in a single response.

3. EVALUATE coverage after subagents return. If coverage is thin
   on a subtopic, re-delegate with a more specific prompt.

4. Once coverage is sufficient, INVOKE the synthesizer subagent
   with all findings packed into its prompt.

QUALITY CRITERIA for subagent prompts:
- Specify research goals and quality criteria
- DO NOT write step-by-step procedural instructions
- Include source-type partitioning to avoid duplication

The synthesizer needs source URLs, publication dates, and
methodology context for every claim.
""",
        agents={
            "researcher": AgentDefinition(
                description=(
                    "Performs web research on a specified subtopic. "
                    "Use when the task requires finding external information. "
                    "Provide the subtopic, source preferences, and a brief on what "
                    "good coverage looks like in the prompt."
                ),
                prompt="""You are a research specialist.
                
For each finding, you MUST include:
- claim: the assertion in your own words
- source_url: where you found it
- source_excerpt: short verbatim quote from the source
- publication_date: when the source was published (best estimate if unclear)
- methodology_context: e.g., 'survey-based', 'experimental', 'opinion piece'
- confidence: 0.0-1.0

Return findings as a structured list, not free-form prose.""",
                tools=["WebSearch", "WebFetch"],
            ),
            "synthesizer": AgentDefinition(
                description=(
                    "Synthesizes research findings into a coherent report. "
                    "Use when search subagents have returned findings and you need a "
                    "final report. The synthesizer needs the structured findings "
                    "from researchers packed into the prompt — it does not see the "
                    "coordinator's history."
                ),
                prompt="""You are a synthesis specialist.

Produce a report with these sections:
1. Well-established findings (direction agreed across sources)
2. Contested findings (sources disagree — show both with attribution)
3. Coverage gaps (subtopics where evidence is thin)
4. Methodology notes

CRITICAL: Preserve source attribution for every claim. Never reduce
to "studies show X" — always say "Source A (date) reports X, while
Source B (date) reports Y."

When numerical values conflict between sources, annotate the
methodology difference (e.g., "Reuters survey n=234 vs.
McKinsey instrumented telemetry n=1200").""",
                tools=["Read"],
            ),
        },
        allowed_tools=["Agent"],  # coordinator can spawn
    )
    
    research_question = (
        "What is the impact of AI coding assistants on developer productivity, "
        "code quality, and long-term codebase maintainability in 2024-2025?"
    )
    
    async with ClaudeSDKClient(options=options) as client:
        await client.query(research_question)
        async for msg in client.receive_response():
            print(msg)

asyncio.run(main())
```

### Lab 2 success criteria

✅ Coordinator emits multiple `Agent` tool calls in one turn (parallel)
✅ Decomposition produces 3-4 distinct subtopics, not a single literal-query route
✅ Researchers return structured findings with source attribution
✅ Synthesizer's prompt contains the packed findings (no implicit context)
✅ Final report preserves source attribution and annotates conflicts

### Lab 2 reflection questions

1. Inspect the coordinator's first response. How many `Agent` tool calls did it emit? Did they run in parallel?
2. What happens if you remove the structured-finding requirement from the researcher's prompt? Does the synthesis lose attribution?
3. Add a `fact_checker` subagent that's invoked only for claims with numerical statistics. How do you make the coordinator decide when to use it?

---

## Lab 3 — Structured Data Extraction Pipeline (D4, D5)

**Scenario:** Build an invoice extraction pipeline with JSON schema, validation, retry-with-feedback, field confidence calibration, and stratified review routing. Maps to **Scenario 6: Structured Data Extraction**.

**You'll exercise:**
- Tool-use + JSON schema for structured output (D4.3)
- Validation + retry with feedback (D4.4)
- Self-validating schemas (paired stated/calculated fields) (D4.4)
- Field-level confidence routing (D5.5)

### Step 1 — Define the schema and extraction tool

```python
# lab3/extract.py
import anthropic
import json

INVOICE_SCHEMA = {
    "type": "object",
    "properties": {
        "invoice_number": {"type": "string"},
        "vendor_name": {"type": "string"},
        "issue_date": {"type": ["string", "null"], "description": "ISO 8601 (YYYY-MM-DD)"},
        "purchase_order_ref": {"type": ["string", "null"]},
        "payment_terms": {"type": ["string", "null"]},
        "currency": {"type": "string", "enum": ["USD", "EUR", "GBP", "INR", "other", "unclear"]},
        "currency_other_detail": {"type": ["string", "null"]},
        "line_items": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "description": {"type": "string"},
                    "quantity": {"type": "number"},
                    "unit_price": {"type": "number"},
                    "subtotal": {"type": "number"},
                },
                "required": ["description", "subtotal"],
            },
        },
        "stated_total": {"type": "number"},
        "calculated_total": {"type": "number", "description": "Sum of line item subtotals"},
        "totals_match": {"type": "boolean"},
        "field_confidence": {
            "type": "object",
            "properties": {
                "invoice_number": {"type": "number"},
                "vendor_name": {"type": "number"},
                "stated_total": {"type": "number"},
                "payment_terms": {"type": "number"},
            },
        },
    },
    "required": ["invoice_number", "vendor_name", "currency", "stated_total", 
                 "calculated_total", "totals_match", "line_items", "field_confidence"],
}

EXTRACTION_PROMPT = """Extract invoice data from the document below.

CRITICAL RULES:
1. If a field is genuinely absent in the source, return null. Do NOT fabricate.
2. Output dates in ISO 8601 (YYYY-MM-DD). If source uses MM/DD/YYYY assume US convention.
3. Fill stated_total from the document; compute calculated_total as sum of line_items subtotals;
   set totals_match to (stated_total == calculated_total).
4. For currency, use enum values; if other (e.g., AUD, JPY), set currency='other' and
   provide currency_other_detail.
5. For each field in field_confidence, report your confidence 0.0-1.0 based on
   how clearly that value appears in the source.

Document:
{document}
"""

def extract_invoice(document_text: str, max_retries: int = 3):
    client = anthropic.Anthropic()
    
    messages = [{"role": "user", "content": EXTRACTION_PROMPT.format(document=document_text)}]
    
    tools = [{
        "name": "extract_invoice",
        "description": "Extract structured invoice data from a document.",
        "input_schema": INVOICE_SCHEMA,
    }]
    
    for attempt in range(max_retries):
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=4096,
            tools=tools,
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages,
        )
        
        # Find the tool_use block
        for block in response.content:
            if block.type == "tool_use" and block.name == "extract_invoice":
                extracted = block.input
                
                # Semantic validation
                errors = validate(extracted)
                if not errors:
                    return extracted
                
                # Retry with specific feedback
                messages.append({"role": "assistant", "content": response.content})
                messages.append({
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": (
                            f"Extraction failed validation. Errors:\n"
                            f"{json.dumps(errors, indent=2)}\n\n"
                            f"Please review the document and correct."
                        ),
                        "is_error": True,
                    }],
                })
                break
    
    raise RuntimeError(f"Max retries exceeded. Last errors: {errors}")

def validate(extracted):
    errors = []
    
    if extracted["currency"] == "other" and not extracted.get("currency_other_detail"):
        errors.append("currency='other' requires currency_other_detail")
    
    computed = sum(item["subtotal"] for item in extracted["line_items"])
    if abs(computed - extracted["calculated_total"]) > 0.01:
        errors.append(
            f"calculated_total ({extracted['calculated_total']}) doesn't match "
            f"sum of line items ({computed})"
        )
    
    actual_match = abs(extracted["stated_total"] - extracted["calculated_total"]) < 0.01
    if extracted["totals_match"] != actual_match:
        errors.append(
            f"totals_match flag ({extracted['totals_match']}) doesn't reflect actual "
            f"comparison: stated={extracted['stated_total']}, "
            f"calculated={extracted['calculated_total']}"
        )
    
    return errors
```

### Step 2 — Routing to humans by confidence

```python
# lab3/route.py
def route_for_review(extracted, confidence_threshold=0.85):
    """Decide which extractions need human review based on field confidence."""
    needs_review = False
    review_reasons = []
    
    field_conf = extracted.get("field_confidence", {})
    for field, conf in field_conf.items():
        if conf < confidence_threshold:
            needs_review = True
            review_reasons.append(f"{field} confidence {conf:.2f} below threshold")
    
    if not extracted["totals_match"]:
        needs_review = True
        review_reasons.append("Stated and calculated totals don't match")
    
    if extracted["currency"] == "unclear":
        needs_review = True
        review_reasons.append("Currency unclear in source")
    
    return {
        "needs_review": needs_review,
        "reasons": review_reasons,
        "extracted": extracted,
    }
```

### Lab 3 success criteria

✅ Extraction returns valid JSON conforming to schema (no syntax errors)
✅ Missing fields return `null` instead of fabricated values
✅ `totals_match` is set correctly based on stated vs. calculated
✅ Validation retry works when totals don't sum
✅ Low-confidence extractions are routed to human review

### Lab 3 reflection questions

1. Run the same document extraction with `tool_choice: "auto"` instead of forced. What changes?
2. Add a `currency_unclear` test document. Does the model correctly use the "unclear" enum value?
3. Calibrate your threshold: label 50 extractions, plot confidence vs. accuracy.

---

## Lab 4 — Claude Code Project with Skills, Rules, and CI Integration (D3)

**Scenario:** Configure a real Claude Code project with the full `.claude/` ecosystem and a CI workflow. Maps to **Scenarios 2, 4, 5: Claude Code workflows**.

**You'll exercise:**
- CLAUDE.md hierarchy (D3.1)
- Slash commands (D3.2)
- Skills with `context: fork` (D3.2)
- Path-specific rules (D3.3)
- CI integration with `-p` flag (D3.6)

### Step 1 — Project structure

```bash
cd ~/cca-labs/lab4
git init
mkdir -p .claude/{commands,rules,skills/codebase-explorer}
mkdir -p src api
```

### Step 2 — Project CLAUDE.md (root)

```markdown
# CLAUDE.md

This is the project-wide context for Claude Code.

## Tech stack
- TypeScript 5.x with strict mode enabled
- Node.js 20+, ESM modules
- Vitest for testing
- pnpm workspace

## Conventions
- All async functions use try/catch with logger.error on failure
- No `any` types without an inline justification comment
- API routes follow REST conventions; versioned under /v2/
- All amounts in cents (integers), never dollars

@import .claude/rules/standards.md
```

### Step 3 — Path-specific rule

```markdown
<!-- .claude/rules/testing.md -->
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
---

When editing or generating test files:
- Use Vitest (NOT Jest). Imports: `import { describe, it, expect, vi } from 'vitest'`
- Tests follow Arrange-Act-Assert pattern with explicit comments
- Mock external dependencies with `vi.mock`
- Test file name matches implementation file with `.test.` infix
- Use `describe.concurrent` for parallelizable suites
```

### Step 4 — Project slash command

```markdown
<!-- .claude/commands/review.md -->
---
description: Run the team's standard PR review checklist
argument-hint: [files-or-PR-ref]
allowed-tools: ["Read", "Grep", "Glob"]
---

You are performing a code review for the following: $ARGUMENTS

Apply our team's review checklist:

1. **Type safety**: any `any` types must have inline justification, otherwise flag.
2. **Error handling**: every `await` should be in a try/catch with logger.error in catch.
3. **Test coverage**: new functions must have corresponding test cases. List functions without tests.
4. **Logging**: no `console.log` — must use `logger.info/warn/error` from `@/lib/logger`.
5. **Comments**: only flag a comment if it contradicts the actual behavior.
6. **API conventions**: routes must be under `/v2/`, return JSON with consistent error envelope.

Output format: structured JSON with `findings` array. Each finding has:
{location: {file, line}, severity: "CRITICAL|HIGH|MEDIUM|LOW", category, issue, suggested_fix}
```

### Step 5 — Skill with context: fork

```markdown
<!-- .claude/skills/codebase-explorer/SKILL.md -->
---
name: codebase-explorer
description: Map an unfamiliar codebase by tracing entry points, dependencies, and conventions. Returns a concise architecture summary.
context: fork
allowed-tools: ["Read", "Grep", "Glob"]
argument-hint: [scope-description]
---

You are a codebase exploration specialist.

When invoked, you will:
1. Use Glob to identify the project structure (package.json, tsconfig, src layout)
2. Use Grep to find entry points (main exports, route handlers, top-level functions)
3. Trace 3-5 most important data flows from entry to persistence
4. Note conventions discovered (error handling style, test patterns, async patterns)

Return ONLY a concise summary. Do NOT include verbose file content listings.

Output format:
## Architecture summary
- Entry points: ...
- Key modules: ...

## Key data flows
1. ...
2. ...

## Conventions discovered
- ...

Scope: $ARGUMENTS
```

### Step 6 — CI workflow

```yaml
# .github/workflows/claude-review.yml
name: Claude PR Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      
      - name: Get changed files
        id: changed
        run: |
          echo "files=$(git diff --name-only origin/main...HEAD | tr '\n' ' ')" >> $GITHUB_OUTPUT
      
      - name: Run Claude review (NON-INTERACTIVE)
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "/review ${{ steps.changed.outputs.files }}" \
            --output-format json \
            > review-output.json
      
      - name: Post findings as PR comments
        run: |
          # Parse review-output.json and post inline comments
          python .github/scripts/post-comments.py review-output.json
```

### Lab 4 success criteria

✅ `/memory` shows project CLAUDE.md loaded (and `~/.claude/` separately)
✅ Editing a `.test.ts` file shows the testing rule loaded
✅ `/review src/auth.ts` runs the team's checklist
✅ `/codebase-explorer "the auth module"` runs in forked context (verify by checking the main session isn't polluted with verbose Read outputs)
✅ CI workflow runs Claude with `-p` and doesn't hang

### Lab 4 reflection questions

1. Move the `.claude/rules/testing.md` rule into `src/CLAUDE.md` — does it still apply to test files outside `src/`?
2. Remove `context: fork` from the skill. Compare main-session token usage before/after.
3. Try running the CI workflow without `-p`. What happens?
4. Modify the slash command to take a severity threshold argument and only output findings at or above that severity.

---

## Capstone Reflection

After all four labs, answer these questions in writing (treat them like exam free-response practice):

1. In Lab 1, why is the PreToolUse hook *necessary* and not just *better* than the prompt instruction?
2. In Lab 2, what would have to change for the coordinator to invoke researchers sequentially instead of in parallel? Is there ever a case where sequential is preferred?
3. In Lab 3, what's the difference between an extraction failing **syntactic** validation (JSON schema) vs. **semantic** validation (totals don't match)? Which one does `tool_use` solve?
4. In Lab 4, why is `context: fork` more important for the codebase-explorer skill than for the `/review` slash command?

If you can answer these crisply in your own words, you've internalized the patterns the exam tests.
