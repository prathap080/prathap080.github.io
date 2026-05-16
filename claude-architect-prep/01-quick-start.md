# 01 — Quick Start (One-Hour Orientation)

> Goal: After this file, you should be able to (a) recognize every term in the exam guide and (b) have a mental model for each domain that you can extend with the deep dives.

---

## The Five Mental Models (One Per Domain)

### 1. Agentic Architecture → "The thermostat loop"
A thermostat keeps checking the temperature and deciding whether to act. Claude works the same way: send a message, look at `stop_reason`, if it's `tool_use` execute the tool and feed the result back, if it's `end_turn` stop. **You never inspect natural-language signals to decide whether to loop.** Just like you wouldn't have a thermostat read a poem about being warm.

For multi-agent systems, picture a **hub-and-spoke wheel**: a coordinator at the center, subagents on the rim. All communication routes through the hub. Subagents never inherit the hub's context — anything they need must be packed into the prompt the hub sends them.

### 2. Tool Design → "The job description"
A tool description is a job listing. If it says "handles content," every tool will apply for the job. If it says "extracts numeric fields from invoice PDFs, returns null if not found, errors if file is not a PDF" — only the right tool applies. **Tool descriptions are the primary mechanism Claude uses to pick a tool.** When two tools sound similar, Claude misroutes — fix this by rewriting descriptions, not by adding a classifier.

For errors: think of tool errors like HTTP responses. `503 Service Unavailable` (transient, retry) is different from `400 Bad Request` (validation, fix the input) is different from `403 Forbidden` (permission, escalate). Generic "Error" tells the agent nothing.

### 3. Claude Code → "The team handbook"
`CLAUDE.md` is the company handbook that everyone reads on day one. It lives at three levels:
- `~/.claude/CLAUDE.md` → your personal preferences (NOT shared)
- Project root `CLAUDE.md` → team-wide (shared via git)
- Subdirectory `CLAUDE.md` → folder-specific rules

For rules that need to apply to **files spread across many folders** (like `**/*.test.tsx`), you can't use directory-bound CLAUDE.md — you need `.claude/rules/` with glob patterns in YAML frontmatter.

Plan mode is "look before you leap" — required for architectural changes. Direct execution is for clear, scoped fixes.

### 4. Prompt Engineering → "The recipe vs. the dish"
Few-shot examples beat detailed instructions when output format matters. **Show, don't tell.** For structured output, use `tool_use` with a JSON schema — this *guarantees* schema compliance (no JSON syntax errors) but does NOT guarantee semantic correctness (line items might still not sum to total).

For batch work: think of the Message Batches API as the **overnight delivery** option. 50% off, but takes up to 24 hours. Never use it for blocking workflows.

### 5. Context Management → "The lossy compression problem"
Every time you summarize, you lose something. Numbers, dates, customer-stated expectations — all get rounded into vague text. The fix is to **extract structured facts into a persistent layer** that travels alongside the summarized prose, not buried inside it.

The "lost in the middle" effect is real: models are reliable at the start and end of long inputs, less so in the middle. Put critical findings at the top.

---

## What's Familiar (From Your Strands/Azure Background)

You already know these — the names will be different but the patterns are identical:

| You know it as | Claude Agent SDK calls it |
|----------------|---------------------------|
| Strands GraphBuilder iterative loops | Coordinator-subagent loops with structured handoffs |
| Strands Hooks (BeforeInvocation) | `PreToolUse` hooks |
| Strands Hooks (AfterInvocation) | `PostToolUse` hooks |
| Bedrock AgentCore session state | `ClaudeAgentOptions` + session resumption |
| Multi-agent orchestrator | Coordinator with `Agent` (formerly `Task`) tool |
| Tool schemas in Strands | MCP tool schemas |
| Your CRO (Clinical Reasoning Object) in Redis | "Case facts" persistent block + scratchpad files |
| `_should_invoke_reviewer()` deterministic routing | Programmatic prerequisites enforced via hooks |

**Your medical symptom checker architecture is genuinely well-aligned with the patterns this exam tests.** The Reasoning Lead ↔ Reviewer loop, the orchestrator with explicit status states, your hook-style enforcement — these are the exam's preferred patterns.

---

## What's Genuinely New

These you have not seen on Azure or in Strands:

1. **Claude Code as a configurable team tool** (entire Domain 3 — 20%). The whole `.claude/` directory ecosystem, slash commands, skills with `context: fork`, plan mode, CLAUDE.md hierarchy.
2. **MCP** as an open protocol (vs. proprietary Bedrock tool integration). Specifically `.mcp.json` scoping (project vs. user), `${VAR}` env expansion, MCP **resources** (not just tools).
3. **The Anthropic-specific `tool_use` + `tool_choice` JSON schema enforcement pattern** for structured output (this is different from Strands' approach).
4. **Message Batches API** semantics (24hr SLA, 50% discount, no multi-turn tool calling).
5. **Claude Code CLI flags** (`-p`, `--output-format json`, `--json-schema`, `--resume`).

You'll spend most of your "new learning" time on Domain 3 (Claude Code) and the MCP-specific bits of Domain 2.

---

## The Ten Things You Must Know Cold

If your brain is fried at midnight before the exam, these ten facts will save you:

1. **Loop control:** Continue while `stop_reason == "tool_use"`. Stop when `"end_turn"`. Never parse text to decide.
2. **Subagent spawning:** Requires `Agent` (exam: `Task`) in `allowedTools`. Subagents start with **fresh context** — pack everything into the prompt.
3. **Coordinator pattern:** Hub-and-spoke. All inter-subagent communication routes through the coordinator. Spawn parallel by emitting multiple Task calls in one response.
4. **Tool selection problems → improve descriptions first.** Few-shot is second resort. Routing classifiers are over-engineering.
5. **Critical sequences (e.g., verify identity before refund) → use hooks (programmatic), not prompt instructions.**
6. **Structured errors:** `errorCategory`, `isRetryable`, human description. Generic errors are an anti-pattern.
7. **CLAUDE.md hierarchy:** user (`~/.claude/`) is personal, project (`.claude/` or root) is shared via git. Glob-pattern rules go in `.claude/rules/` with YAML frontmatter.
8. **Plan mode** for architectural/multi-file/multiple-valid-approaches tasks. **Direct execution** for clear scoped changes.
9. **Structured output:** `tool_use` + JSON schema. Use `tool_choice: "any"` to force a tool call; `{"type": "tool", "name": "X"}` to force a specific tool.
10. **Batch API:** 50% off, ≤24hr, **no multi-turn tool calling**. Only for non-blocking workloads.

---

## How to Approach an Exam Question (The 4-Step Filter)

Every exam question gives you 4 options. Apply this filter in order:

1. **Eliminate "do nothing" / "wait and see" / "switch model" options.** Larger context windows don't fix attention dilution. Higher-tier models don't fix bad architecture.
2. **Eliminate options that rely on probabilistic LLM behavior when correctness is required.** "Add to system prompt that the agent MUST..." is wrong when there's a programmatic alternative.
3. **Eliminate over-engineered options when a proportionate first response exists.** "Train a classifier" / "Add ML routing" / "Build microservice" are usually wrong when "improve the tool description" is on the table.
4. **Pick the option that addresses the root cause shown in the scenario logs.** If logs show wrong tool selection, fix tool descriptions. If logs show too narrow decomposition, fix the coordinator. Don't blame downstream agents that are working correctly within their assigned scope.

This filter alone resolves probably 60% of exam questions.

---

## The Anti-Pattern Catalog (Quick Recognition)

These exact things appear as **wrong answers** in exam questions. If you see them, eliminate:

- ❌ Parsing assistant text to decide loop termination
- ❌ Setting iteration caps as the primary stopping mechanism
- ❌ "Be conservative" / "only report high-confidence" as precision improvements
- ❌ LLM self-reported confidence scores as escalation signal
- ❌ Sentiment analysis as escalation signal
- ❌ Heuristic selection when multiple customer/order matches exist
- ❌ Catching subagent errors and returning empty success
- ❌ Generic "operation failed" error responses
- ❌ Self-review by the same instance that generated the code
- ❌ Larger context window as fix for attention dilution
- ❌ Routing classifier when tool descriptions are inadequate
- ❌ Consolidating tools into a single mega-tool to fix selection problems
- ❌ Switching to batch API for blocking workflows
- ❌ Subdirectory `CLAUDE.md` for files spread across the codebase (use glob rules instead)
- ❌ Personal `~/.claude/CLAUDE.md` for team standards (won't be shared)

Full catalog with explanations is in `08-anti-patterns-reference.md`.

---

## Next

Now read `02-strands-azure-translation.md` to map your existing knowledge onto Claude SDK terminology, then dive into the deep dives in order. Keep `cheatsheet.html` open as a reference while studying.
