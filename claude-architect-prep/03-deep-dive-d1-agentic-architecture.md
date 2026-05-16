# 03 — Domain 1: Agentic Architecture & Orchestration (27%)

> **Largest domain on the exam.** Master this and you've secured a quarter of your score. Maps to: Customer Support, Multi-Agent Research, Developer Productivity scenarios.

---

## What This Domain Tests

Seven task statements:
- 1.1 Agentic loop lifecycle
- 1.2 Coordinator-subagent orchestration
- 1.3 Subagent invocation, context passing, spawning
- 1.4 Multi-step workflows with enforcement and handoffs
- 1.5 Hooks for tool call interception and data normalization
- 1.6 Task decomposition strategies
- 1.7 Session state, resumption, forking

The unifying theme: **Claude decides what to do (model-driven)**, but **you enforce the invariants programmatically**.

---

## 1.1 — The Agentic Loop

### The control flow (memorize this exactly)

```
┌────────────────────────────────────┐
│ 1. Send messages to Claude         │
│    (system + tools + history)      │
└────────────────────────────────────┘
                ↓
┌────────────────────────────────────┐
│ 2. Inspect response.stop_reason    │
└────────────────────────────────────┘
        ↓                    ↓
   "tool_use"           "end_turn"
        ↓                    ↓
┌─────────────────┐   ┌──────────────┐
│ 3. Execute      │   │ Terminate.    │
│    requested    │   │ Return final  │
│    tool(s)      │   │ message.      │
└─────────────────┘   └──────────────┘
        ↓
┌────────────────────────────────────┐
│ 4. Append tool_use + tool_result   │
│    to conversation history         │
└────────────────────────────────────┘
        ↓
        └──→ back to step 1
```

### Why the model needs tool results in context

Tool results are appended to the conversation so the model can **reason about the next action** based on what the tool returned. If the order lookup returned "no results found," the model needs to see that to decide whether to ask for another identifier or escalate. If you swallow the result, the model loops blindly.

### Anti-patterns (every one of these is a wrong answer)

❌ **Parsing assistant text to detect completion.** The model might say "I'm done now!" even when it's not — or might be done without saying so. Use `stop_reason`.

❌ **Using iteration caps as the primary stopping mechanism.** Caps exist as a safety valve, not as flow control. They cut off legitimate work and don't prevent infinite loops in the right way.

❌ **Checking for assistant text content as a completion indicator.** Same problem. The model can return text *and* still want to use a tool next turn.

### Code (Python — Anthropic API directly, the loop you'd write yourself)

```python
import anthropic

client = anthropic.Anthropic()

def run_agent_loop(initial_user_message, tools, system_prompt):
    messages = [{"role": "user", "content": initial_user_message}]
    
    while True:
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages,
        )
        
        # Always append the assistant turn
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason == "end_turn":
            return response  # Done
        
        if response.stop_reason == "tool_use":
            # Execute every tool_use block in this response
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            messages.append({"role": "user", "content": tool_results})
            continue  # Loop back
        
        # Other stop_reasons: max_tokens, pause_turn, etc.
        # Handle defensively but don't ignore.
        raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")
```

### 🔧 Practical Task 1.1
Write the loop above. Then deliberately break it three ways and observe what happens:
1. Forget to append `tool_results` between iterations — what does the model do next turn?
2. Use `if "done" in response.content[0].text` instead of `stop_reason` — find a case where it fails.
3. Try to use both a `tool_use` block AND text in the same assistant response — verify the loop handles this.

---

## 1.2 — Coordinator-Subagent Architecture

### The hub-and-spoke pattern

```
                ┌──────────────┐
                │ Coordinator  │
                │  (the hub)   │
                └──────────────┘
                ↑   ↑   ↑   ↑
                │   │   │   │
        ┌───────┘   │   │   └────────┐
        ↓           ↓   ↓            ↓
   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
   │ Search │ │Analyze │ │Synth   │ │ Report │
   │ agent  │ │ agent  │ │ agent  │ │ agent  │
   └────────┘ └────────┘ └────────┘ └────────┘
```

**Rules of the pattern (these come up in exam questions):**

1. **All inter-subagent communication routes through the coordinator.** Subagent A doesn't talk to Subagent B directly — A returns to coordinator, coordinator decides whether to invoke B, packages context, sends.

2. **Subagents have isolated context.** They start with **fresh conversation history**. Anything they need to know must be in the prompt the coordinator sends.

3. **The coordinator decomposes the task.** Bad decomposition → bad coverage. If your coordinator decomposes "AI in creative industries" into only "digital art / graphic design / photography," it will completely miss music/writing/film. The fix is the coordinator's prompt, not the subagents.

4. **The coordinator dynamically picks subagents.** Don't always route through every subagent. Assess query complexity first.

5. **The coordinator runs iterative refinement loops.** After synthesis, evaluate for gaps → re-delegate to search/analysis with targeted queries → re-invoke synthesis.

### Why hub-and-spoke (vs. mesh / pipeline)

- **Observability:** all communication flows through one place
- **Consistent error handling:** the coordinator decides recovery
- **Controlled information flow:** the coordinator filters/transforms before passing on

A mesh (subagents talking to each other) makes failures hard to recover from and produces non-deterministic execution paths.

### Anti-patterns

❌ **"The synthesis agent failed to detect coverage gaps in findings."** When logs show the *coordinator's decomposition* was too narrow, blaming downstream agents is wrong. They executed correctly within their scope.

❌ **Always running the full pipeline regardless of query.** Wastes tokens and time on simple queries.

❌ **Subagents calling each other directly.** Loses observability, complicates error handling.

### 🔧 Practical Task 1.2
Build a coordinator with two subagents (`web_searcher` and `summarizer`) using the Claude Agent SDK:
1. Run a query like "What are recent breakthroughs in fusion energy?"
2. Verify the coordinator decomposes into search subtopics rather than just routing the literal query
3. Examine what context the summarizer subagent receives — confirm it does NOT have the user's original message unless explicitly passed
4. Add a third subagent (`fact_checker`) that's only invoked for synthesis with claims that include numerical statistics

---

## 1.3 — Subagent Invocation, Context Passing, Spawning

### The mechanics

In the Claude Agent SDK, subagents are spawned via the **`Agent` tool** (the exam guide calls it `Task` — this was renamed; both refer to the same thing).

```python
options = ClaudeAgentOptions(
    allowed_tools=["Agent"],  # ← MUST include for coordinator to spawn subagents
    agents={
        "researcher": AgentDefinition(
            description=(
                "Performs web research. Use when the task requires finding "
                "external information from web sources. Provide the research "
                "topic and any specific source preferences in the prompt."
            ),
            prompt="You are a research specialist...",
            tools=["WebSearch", "WebFetch"],
        ),
        "synthesizer": AgentDefinition(
            description="Synthesizes findings into a coherent report...",
            prompt="You are a synthesis specialist...",
            tools=["Read"],
        ),
    },
)
```

### Context isolation — the most-tested concept

When you spawn a subagent, **only the prompt string** crosses the boundary. The subagent does NOT see:
- The coordinator's conversation history
- Other subagents' findings
- Tool results from prior turns

If the synthesizer needs the searcher's findings, the **coordinator must pack them into the synthesizer's prompt**:

```python
# What the coordinator's tool call looks like (conceptually)
spawn_subagent(
    subagent_type="synthesizer",
    prompt=f"""
    Synthesize these findings into a 500-word brief.
    
    SOURCE 1 (web — Reuters, 2025-09):
    {searcher_finding_1}
    
    SOURCE 2 (web — NYT, 2025-08):
    {searcher_finding_2}
    
    Quality criteria: cite each claim with source attribution.
    Preserve numerical values exactly. Do not invent details.
    """
)
```

### Parallel subagent execution

To run subagents in parallel, the coordinator emits **multiple `Agent` tool calls in a single response**. This is one of the highest-leverage performance patterns the exam tests.

**Sequential (slow):**
```
Turn 1: spawn searcher → wait → Turn 2: spawn analyzer → wait → Turn 3: synthesize
```

**Parallel (fast):**
```
Turn 1: spawn searcher AND analyzer (both Task calls in one assistant response)
        → both run in parallel → Turn 2: synthesize with both results
```

### Forks (alternative branches)

`fork_session` creates an independent branch from a shared analysis baseline. Use when you want to explore divergent approaches without contaminating the main session. Example: "From this codebase analysis, fork into two branches — one explores REST migration, one explores GraphQL migration."

### Coordinator prompt design

✅ **Specify research goals and quality criteria.** Subagents need to know what good looks like.
❌ **Don't write step-by-step procedural instructions.** This kills adaptability.
✅ **Include source-type partitioning** so subagents don't duplicate work (e.g., "Searcher A: academic papers; Searcher B: industry blogs").

### 🔧 Practical Task 1.3
Spawn 3 search subagents in parallel for the topic "evolution of CSS frameworks 2020-2025":
- One assigned to "early influences (Bootstrap era)"
- One to "utility-first revolution (Tailwind)"
- One to "modern era (CSS variables, container queries)"
Verify they don't duplicate sources. Then have the coordinator detect that "early influences" came back thin and re-delegate with a more specific prompt.

---

## 1.4 — Multi-Step Workflows with Enforcement

### The core distinction

| Approach | When to use | Reliability |
|----------|-------------|-------------|
| **Programmatic enforcement** (hooks, prerequisite gates) | Required invariants (e.g., verify identity before refund) | Deterministic |
| **Prompt-based guidance** (system prompt instructions) | Soft preferences, non-critical ordering | Probabilistic (~95-99%, never 100%) |

**The exam pattern:** when correctness matters financially, legally, or for safety — **always pick programmatic**. Prompt instructions have a non-zero failure rate, and "non-zero" is unacceptable when refunds, identity, or compliance are involved.

### Programmatic prerequisites (the canonical exam example)

> "Block `process_refund` and `lookup_order` from being called until `get_customer` has returned a verified customer ID."

```python
verified_customers = set()

async def enforce_customer_verification(tool_input, context):
    """PreToolUse hook — runs before any tool call."""
    if context.tool_name == "get_customer":
        # Allow; capture the verified ID
        return {"hookSpecificOutput": {"permissionDecision": "allow"}}
    
    if context.tool_name in ("lookup_order", "process_refund"):
        if not verified_customers:
            return {
                "hookSpecificOutput": {
                    "permissionDecision": "deny",
                    "permissionDecisionReason": (
                        "Must call get_customer first to verify customer identity."
                    ),
                }
            }
    return {}

async def capture_verification(tool_input, tool_response, context):
    """PostToolUse hook — runs after."""
    if context.tool_name == "get_customer" and tool_response.get("verified"):
        verified_customers.add(tool_response["customer_id"])
    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [HookMatcher(matcher="*", hooks=[enforce_customer_verification])],
        "PostToolUse": [HookMatcher(matcher="*", hooks=[capture_verification])],
    }
)
```

### Decomposing multi-concern requests

When a customer says: "I want a refund for the damaged item AND I'd like to dispute the late fee from last month AND can you confirm my address change went through?"

✅ **Decompose into distinct items, investigate each in parallel using shared context, synthesize unified resolution.**
❌ Handle them sequentially in conversation order.
❌ Pick the "main" issue and silently drop the others.

### Structured handoff to humans

When escalating, the human agent does NOT see the conversation transcript. The handoff payload must include:

```python
handoff = {
    "customer_id": "CUS-12345",
    "verified_identity": True,
    "issue_summary": "Customer requesting price match for competitor offer",
    "root_cause_analysis": "Policy is silent on competitor matching (only addresses own-site adjustments)",
    "investigation_completed": [
        "Verified customer identity",
        "Confirmed order eligibility",
        "Located competitor price ($89.99 at retailer X)",
    ],
    "recommended_action": "Approve one-time competitor match per goodwill policy guidance",
    "blocking_constraint": "Requires policy exception authorization",
}
```

### 🔧 Practical Task 1.4
Implement the customer verification hook above. Then test:
1. Try to `process_refund` without first calling `get_customer` — verify the hook blocks
2. Call `get_customer` with an unverifiable ID — what happens on the next refund attempt?
3. Add a hook that blocks refunds > $500 and triggers a structured human handoff

---

## 1.5 — Hooks: Interception and Normalization

### Hook patterns the exam tests

**Pattern A — `PostToolUse` for data normalization**

You have three tools that return timestamps in different formats:
- `lookup_order` returns Unix timestamp (`1730409600`)
- `lookup_shipment` returns ISO 8601 (`"2025-09-12T10:00:00Z"`)
- `lookup_invoice` returns numeric status code (`200`)

Without normalization, the model wastes tokens reasoning about format conversion and may confuse fields. With a `PostToolUse` hook, you normalize all of them to ISO 8601 before the model ever sees them.

**Pattern B — `PreToolUse` for compliance enforcement**

Block any `process_refund` call where amount > $500. Don't just deny — **redirect to escalation workflow**.

```python
async def block_high_value_refunds(tool_input, context):
    if context.tool_name == "process_refund":
        if tool_input.get("amount", 0) > 500:
            return {
                "hookSpecificOutput": {
                    "permissionDecision": "deny",
                    "permissionDecisionReason": (
                        "Refund exceeds $500 limit. Use escalate_to_human "
                        "with the structured handoff payload."
                    ),
                }
            }
    return {}
```

### When hooks vs. when prompt instructions

| If the requirement is... | Use |
|--------------------------|-----|
| Legal / financial / safety / regulatory | **Hooks** (deterministic) |
| Style / preference / soft guidance | **Prompt** (probabilistic) |
| Multi-step ordering with consequences | **Hooks** (prerequisite gates) |
| Output format consistency | **Prompt** (with few-shot) |
| Data normalization across heterogeneous tools | **Hooks** (PostToolUse) |
| "Be polite to customers" | **Prompt** |

### 🔧 Practical Task 1.5
Build a `PostToolUse` hook that:
1. Normalizes timestamps from three different formats to ISO 8601
2. Trims tool responses to only fields actually relevant to the agent's current task (e.g., for an order lookup, keep only `order_id`, `status`, `total`, `items` — drop the other 35 fields)

---

## 1.6 — Task Decomposition Strategies

### Two flavors of decomposition

**Prompt chaining (sequential pipeline)**
- Use when: the workflow is **predictable** with multiple known aspects
- Example: code review → analyze each file individually → run cross-file integration pass

**Dynamic adaptive decomposition**
- Use when: the workflow is **open-ended** and depends on intermediate findings
- Example: "Add comprehensive tests to a legacy codebase" → first map structure → identify high-impact areas → create prioritized plan that adapts as dependencies are discovered

### The exam pattern

When the question describes:
- "predictable multi-aspect review of N files" → **prompt chaining (per-file + integration)**
- "open-ended investigation that adapts to what is discovered" → **dynamic decomposition**

### Why split per-file + integration

Reviewing 14 files in a single pass causes **attention dilution**: detailed feedback for some files, superficial for others, contradictory findings (flagging a pattern as a problem in one file while approving identical code elsewhere). Splitting:
- File-by-file passes ensure consistent depth
- A separate cross-file integration pass catches data-flow issues

This is why "switch to a larger context window" is the wrong answer to attention-dilution questions. Bigger context doesn't solve attention quality.

---

## 1.7 — Session State, Resumption, Forking

### Three operations

```bash
# Resume a named session
claude --resume my-investigation
```

```python
# Fork a session — create a parallel branch from a shared baseline
options = ClaudeAgentOptions(fork_session=True, resume="codebase-analysis-v1")
```

### When to resume vs. when to start fresh with summary

| Situation | Use |
|-----------|-----|
| Prior context is mostly valid, files unchanged | **Resume with `--resume`** |
| Files have been modified since prior session | **Resume + inform the agent of specific changes** |
| Prior tool results are stale (data changed, env shifted) | **Start fresh + inject structured summary** |
| You want to compare two divergent approaches from a baseline | **`fork_session`** |

### Why "start fresh with summary" beats stale resumption

If the agent resumes with stale tool results, it may make decisions based on outdated information without realizing it. A clean session with an injected summary forces the agent to reason from current ground truth.

### 🔧 Practical Task 1.7
Build a 2-phase investigation:
1. Phase 1: Use Claude Agent SDK to map a small open-source repo's architecture; save the session
2. Phase 2: Modify a file in the repo, then resume the session — explicitly tell Claude what changed
3. Phase 2-alt: Fork the session into two branches, one exploring "add feature X via approach A" and one via "approach B"

---

## Domain 1 Summary — The Exam-Ready Cheat Sheet

| Concept | The right answer pattern |
|---------|--------------------------|
| Loop control | Use `stop_reason`, never parse text, never iteration cap as primary control |
| Multi-agent orchestration | Hub-and-spoke; coordinator decides; subagents have fresh context |
| Spawning | `Agent` (exam: `Task`) in `allowedTools`; pack everything into prompt |
| Parallel execution | Multiple `Agent` calls in **one** assistant response |
| Required ordering (e.g., verify before refund) | Programmatic hook with prerequisite gate |
| Soft ordering or preferences | Prompt instruction with examples |
| Heterogeneous tool result formats | `PostToolUse` hook for normalization |
| Compliance blocking | `PreToolUse` hook with deny + redirect |
| Predictable multi-file review | Prompt chaining (per-file + integration pass) |
| Open-ended exploration | Dynamic adaptive decomposition |
| Coverage gap (logs show narrow decomposition) | Fix coordinator's decomposition prompt |
| Session continuity | `--resume` if context valid; fresh + summary if stale |
| Divergent exploration | `fork_session` |

---

## Domain 1 Drill Questions (For Self-Test)

1. The agent skips `get_customer` 12% of the time and proceeds directly to `process_refund`. What's the right fix?
2. Three subagents complete successfully but the final report misses entire subtopics. Whose problem is it?
3. The synthesis agent has access to all 18 tools across all subagents to "be flexible." What's wrong with this?
4. You're reviewing 14 files in one pass and getting contradictory findings. What's the fix?
5. The coordinator runs all 4 subagents on every query, even simple ones. What's the right pattern?

(Answers: 1) Programmatic prerequisite hook 2) Coordinator's decomposition 3) Tool overload degrades selection — scope each agent's tools 4) Split into per-file + integration passes 5) Coordinator should dynamically select subagents based on query complexity)
