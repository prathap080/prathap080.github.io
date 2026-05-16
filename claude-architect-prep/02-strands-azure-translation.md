# 02 — Strands / Azure → Claude Agent SDK Translation

> Goal: Map every concept from your Bedrock + Strands + Azure Agents background to the Claude Agent SDK so you can leverage what you already know.

---

## Mental Model Translation

### Your Strands GraphBuilder pattern → Claude Agent SDK

In Strands, you defined a graph of agents with explicit edges and loop conditions (your Reasoning ↔ Reviewer iterative loop). In Claude Agent SDK, **you don't define the graph statically.** You define agent capabilities and tool access, and the **coordinator agent dynamically decides** which subagents to invoke based on the query.

```python
# Strands (your current style — graph-based)
graph = GraphBuilder()
graph.add_node("reasoning", reasoning_agent)
graph.add_node("reviewer", reviewer_agent)
graph.add_edge("reasoning", "reviewer", condition=needs_review)
graph.add_edge("reviewer", "reasoning", condition=needs_iteration)

# Claude Agent SDK (declare capabilities, let the coordinator decide)
options = ClaudeAgentOptions(
    agents={
        "reasoning_lead": AgentDefinition(
            description="Performs clinical analysis and differential diagnosis",
            prompt="You are a clinical reasoning specialist...",
            tools=["Read", "lookup_snomed", "search_drug_db"],
        ),
        "reviewer": AgentDefinition(
            description="Audits clinical reasoning for gaps and inconsistencies",
            prompt="You are a senior physician reviewer...",
            tools=["Read", "lookup_snomed"],
        ),
    },
    allowed_tools=["Agent", "Read"],  # coordinator can spawn subagents
)
```

**Key shift:** Stop thinking in graphs. Start thinking in **capabilities + descriptions**. The coordinator chooses dynamically.

---

### Your Bedrock orchestrator → Claude Agent SDK coordinator

You wrote a deterministic orchestrator with `_should_invoke_reviewer()` and explicit status states (`IN_PROGRESS`, `REASONING_LOOP`, etc.). The exam-recommended pattern is:

1. **Programmatic enforcement** for invariants that must hold (use **hooks**, not LLM compliance)
2. **Coordinator agent** for dynamic routing decisions (let the LLM decide based on rich descriptions)
3. **Structured handoffs** when escalating to humans (always include customer ID, root cause, recommended action)

Your `_should_invoke_reviewer()` is a great pattern but in Claude Agent SDK terms, it would be implemented as **a `PreToolUse` hook** that intercepts certain tool calls and routes them.

---

### Your Comprehend Medical PHI detection → Claude Agent SDK hooks

In your stack, you use Comprehend Medical for PHI scrubbing on the way out. In Claude Agent SDK, this is a `PostToolUse` hook:

```python
async def phi_scrub_hook(tool_input, tool_response, context):
    """Intercepts tool results, scrubs PHI before model sees them."""
    if context.tool_name in ["lookup_patient_record", "fetch_history"]:
        scrubbed = await comprehend_medical.detect_phi(tool_response)
        return {"hookSpecificOutput": {"toolResult": scrubbed}}
    return {}

options = ClaudeAgentOptions(
    hooks={
        "PostToolUse": [HookMatcher(matcher="*", hooks=[phi_scrub_hook])]
    }
)
```

**Hook events available** (memorize these — exam tests them):
- `PreToolUse` — intercept before execution (block or modify)
- `PostToolUse` — intercept after execution (transform results)
- `PostToolUseFailure` — handle tool failures
- `UserPromptSubmit`, `Stop`, `SubagentStop`, `SubagentStart`
- `PreCompact`, `Notification`, `PermissionRequest`

---

### Your CRO (Clinical Reasoning Object) → Claude's "case facts" pattern

You persist CROs in Redis with archival to DynamoDB. The exam's analog is a **"case facts" block** that's:

1. Extracted from each interaction's structured data (amounts, dates, IDs, statuses)
2. Persisted **outside** the conversation summary
3. Re-injected into every prompt as a stable context layer
4. Survives compaction/summarization

```python
# This is your CRO pattern, expressed in Claude Agent SDK terms
case_facts = {
    "customer_id": "CUS-12345",
    "order_id": "ORD-78901",
    "refund_amount": 89.99,
    "verification_status": "verified",
    "issue_categories": ["damaged_item", "billing_dispute"],
}

system_prompt = f"""
You are a customer support resolution agent.

CASE FACTS (always current — do not summarize away):
{json.dumps(case_facts, indent=2)}

Conversation history follows below...
"""
```

---

## Terminology Side-by-Side

| Concept | Strands / Bedrock | Claude Agent SDK |
|---------|-------------------|------------------|
| Agent definition | `Agent(name=, system_prompt=, tools=)` | `AgentDefinition(description=, prompt=, tools=)` |
| Loop control | StatusCallback / GraphBuilder edges | `stop_reason` inspection ("tool_use" vs "end_turn") |
| Tool spawning | Direct invocation in graph | `Agent` (formerly `Task`) tool in `allowedTools` |
| Tool schemas | Pydantic / JSON schema in tool def | MCP tool schemas, `tool_use` blocks |
| Pre-call hooks | `BeforeInvocation` callbacks | `PreToolUse` hooks |
| Post-call hooks | `AfterInvocation` callbacks | `PostToolUse` hooks |
| Session state | DynamoDB checkpoints | `--resume <session-name>`, `fork_session` |
| Context isolation | Separate Bedrock invocations | Subagent fresh context (auto-isolated) |
| Forced tool call | `tool_choice` on Bedrock InvokeModel | `tool_choice: {"type": "tool", "name": "..."}` |
| Streaming | Bedrock `invoke_with_response_stream` | SSE + `query()` async generator |

---

## API Surface Map (Python)

```python
# Imports
from claude_agent_sdk import (
    query,                  # Async generator — main entry point
    ClaudeSDKClient,        # Stateful client (supports custom tools + hooks)
    ClaudeAgentOptions,     # Config object (was ClaudeCodeOptions)
    AgentDefinition,        # Subagent definition
    HookMatcher,            # Hook registration
    tool,                   # @tool decorator for custom tools
    create_sdk_mcp_server,  # In-process MCP server
)

# Simple query
async for message in query(prompt="Analyze this codebase"):
    print(message)

# With full options
options = ClaudeAgentOptions(
    system_prompt="You are a clinical reasoning agent...",
    allowed_tools=["Read", "Agent", "lookup_snomed"],   # subagents need "Agent"
    disallowed_tools=["Bash"],                          # explicit deny
    permission_mode="acceptEdits",                      # auto-accept edits
    cwd="/path/to/project",
    max_turns=20,
    agents={"reviewer": AgentDefinition(...)},
    mcp_servers={"snomed": {"command": "node", "args": ["snomed-server.js"]}},
    hooks={"PostToolUse": [HookMatcher(matcher="*", hooks=[scrub_phi])]},
)

# Stateful with custom tools
async with ClaudeSDKClient(options=options) as client:
    await client.query("Diagnose patient with chest pain")
    async for msg in client.receive_response():
        ...
```

---

## Mapping Your Medical Symptom Checker to Exam Concepts

Your project happens to be a near-perfect study aid because it touches every domain. Here's how:

| Your component | Maps to exam concept | Domain |
|----------------|---------------------|--------|
| Reasoning Lead + Reviewer agents | Coordinator-subagent architecture | D1 |
| Iterative loop until convergence | Iterative refinement loop in coordinator | D1 |
| `_should_invoke_reviewer()` routing | Programmatic prerequisites (hooks > prompts) | D1 |
| ARN-based + HTTP POST orchestrator | Same as Bedrock-vs-Anthropic API distinction | (background) |
| CRO persisted in Redis | Case facts / structured state extraction | D5 |
| Comprehend Medical PHI detection | `PostToolUse` hook for data normalization | D1 |
| Split-model strategy (Sonnet thinking + standard) | Model selection per agent role | (background) |
| Explicit status states with circuit breakers | Structured error propagation | D5 |
| EMERGENCY_STOP state | Escalation triggers (policy gaps, can't progress) | D5 |
| SNOMED CT lookup | MCP tool with detailed description | D2 |
| Differential diagnosis output | Structured output via `tool_use` + JSON schema | D4 |
| Confidence scoring | Field-level confidence calibration | D5 |

When you study Domain 5 (Context & Reliability), explicitly map back to your CRO architecture — that will cement it.

---

## What to Unlearn

A few things from your current stack that won't serve you on this exam:

1. **Stop thinking "graph-based orchestration first."** The Claude SDK favors capability-based dynamic routing for the coordinator's decisions. Reserve programmatic graphs for *invariants* (hooks).
2. **Don't reach for ML classifiers as the first solution.** The exam consistently marks "deploy a separate classifier" as the wrong answer when prompt/description fixes are available.
3. **Don't trust LLM self-reported confidence as a routing signal.** This is an exam-grade anti-pattern — your existing system probably uses confidence well *internally*, but as a routing trigger it's marked incorrect.
4. **MCP tools accept simple JSON schemas.** Don't over-engineer them with Pydantic class hierarchies (in code that's fine, but don't expect schema features beyond JSON schema basics).

---

## Next

Read `03-deep-dive-d1-agentic-architecture.md` — the largest domain at 27%.
