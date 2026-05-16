# 08 — Anti-Patterns Reference

> The single highest-leverage exam tool. **Multiple-choice exams test recognition of wrong answers as much as the right one.** This file catalogs every wrong-answer pattern from the official guide with WHY it's wrong.

---

## How to Use This File

When you see an option in the exam that matches one of these patterns → eliminate immediately. You don't need to know the right answer to eliminate the wrong ones.

Memorize the **bolded reasons** — they're the diagnostic phrases that tell you it's a trap.

---

## Domain 1: Agentic Architecture Anti-Patterns

### A1.1 — "Parse the assistant's text to determine when to stop the loop"
**Why wrong:** The model can return text and still want to use a tool next turn, OR can be done without explicitly saying so. **`stop_reason` is the only reliable signal.**

### A1.2 — "Set an iteration cap as the primary stopping mechanism"
**Why wrong:** Iteration caps are a safety valve, not flow control. They cut off legitimate work and don't address the actual termination condition. **Use `stop_reason` for control; use caps only as a defensive backstop.**

### A1.3 — "Check for assistant text content as a completion indicator"
**Why wrong:** Same problem as A1.1 — text presence doesn't correlate with completion.

### A1.4 — "The synthesis agent failed to detect coverage gaps"
**Why wrong (when logs show coordinator decomposed too narrowly):** Downstream agents executed correctly within their assigned scope. **The bug is in the coordinator's decomposition, not the subagents' execution.**

### A1.5 — "Always route through the full pipeline regardless of query complexity"
**Why wrong:** Wastes tokens on simple queries. **Coordinator should dynamically select subagents based on query requirements.**

### A1.6 — "Add a system prompt instruction that the agent MUST verify customer identity before refunds"
**Why wrong (when financial correctness is required):** Prompt instructions have a non-zero failure rate. **Use programmatic enforcement (hooks) for required invariants.**

### A1.7 — "Subagents communicate directly with each other"
**Why wrong:** Loses observability, complicates error handling, breaks the hub-and-spoke pattern. **All inter-subagent communication should route through the coordinator.**

### A1.8 — "Spawn subagents sequentially across separate turns"
**Why wrong (when parallelism is possible):** Wastes time. **Emit multiple Task tool calls in a single coordinator response for parallelism.**

### A1.9 — "Subagents inherit the coordinator's conversation history automatically"
**Why wrong:** They don't. **Subagents start with fresh context; everything they need must be packed into the prompt.**

### A1.10 — "Resume a session even when prior tool results are stale"
**Why wrong:** Agent makes decisions based on outdated information without realizing it. **Start fresh with an injected summary when prior results are stale.**

---

## Domain 2: Tool Design Anti-Patterns

### A2.1 — "Add few-shot examples to fix tool selection between two similar tools"
**Why wrong (as first step):** Adds token overhead without addressing the root cause. **Improve the tool descriptions first** — they're the primary mechanism the LLM uses for selection.

### A2.2 — "Build a routing classifier that pre-selects tools"
**Why wrong:** Over-engineered. Bypasses Claude's natural language understanding. **Tool descriptions are the right lever.**

### A2.3 — "Consolidate similar tools into a single mega-tool with a `mode` parameter"
**Why wrong (as a fix for selection problems):** Architectural shift to fix a description-level problem. **Differentiate descriptions, or split into purpose-specific tools.**

### A2.4 — "Return generic 'Operation failed' on errors"
**Why wrong:** Forces agent to retry blindly or give up. **Return structured errors: `errorCategory`, `isRetryable`, `humanReadable`.**

### A2.5 — "Catch errors in subagents and return empty results as success"
**Why wrong:** Coordinator can't distinguish "no matches" from "lookup never ran." **Use `isError` flag + structured metadata to enable intelligent recovery.**

### A2.6 — "Give every agent access to all 18 tools for flexibility"
**Why wrong:** Tool overload degrades selection reliability. **Scope each agent's tools to its role (typically 4-5).**

### A2.7 — "Give the synthesis agent full web search access to avoid round-trips"
**Why wrong:** Over-provisions the synthesis agent, violating separation of concerns. **Use a scoped exception: a simple `verify_fact` tool for the 85% common case.**

### A2.8 — "Put MCP server config in `~/.claude.json` for team sharing"
**Why wrong:** `~/.claude.json` is user-only — not shared via git. **Put team-shared MCP servers in `.mcp.json` at the project root.**

### A2.9 — "Hardcode credentials in `.mcp.json`"
**Why wrong:** Commits secrets to git. **Use `${ENV_VAR}` expansion.**

### A2.10 — "Build a custom MCP server for standard JIRA integration"
**Why wrong:** Reinvents the wheel. **Use community MCP servers for standard integrations; reserve custom for team-specific needs.**

### A2.11 — "Agent uses Grep when our custom search tool would be better"
**Why wrong:** Tool description is weaker than Grep's. **Strengthen the MCP tool description so the agent prefers it.**

---

## Domain 3: Claude Code Anti-Patterns

### A3.1 — "Personal `~/.claude/CLAUDE.md` for team-shared standards"
**Why wrong:** Not committed to git, teammates won't see it. **Use project-root `CLAUDE.md`.**

### A3.2 — "Subdirectory `CLAUDE.md` for files spread across the codebase (e.g., test files everywhere)"
**Why wrong:** CLAUDE.md is directory-bound. Files spread across directories won't all match. **Use `.claude/rules/` with glob patterns in YAML frontmatter.**

### A3.3 — "Consolidate all conventions in root CLAUDE.md, relying on Claude to infer which apply"
**Why wrong:** Inference is unreliable. **Path-specific rules with explicit globs are deterministic.**

### A3.4 — "Use a skill that requires manual invocation for automatic application"
**Why wrong:** Skills are on-demand. For automatic application, use rules or CLAUDE.md.

### A3.5 — "Add `~/.claude/commands/review.md` for team-wide use"
**Why wrong:** User-scoped, not shared. **Use `.claude/commands/review.md` in the project.**

### A3.6 — "Add a commands array to `.claude/config.json`"
**Why wrong:** This mechanism doesn't exist. **Slash commands are markdown files in `.claude/commands/`.**

### A3.7 — "Use direct execution for monolith → microservices restructuring"
**Why wrong:** Architectural decisions, multi-file scope, multiple valid approaches → **plan mode is required.**

### A3.8 — "Start direct, switch to plan mode if complexity emerges"
**Why wrong (when complexity is already stated in requirements):** The complexity isn't going to "emerge" — it's already there. **Plan mode from the start.**

### A3.9 — "Use a larger context window to fix attention dilution in 14-file review"
**Why wrong:** Bigger context doesn't improve attention quality. **Split into per-file + cross-file integration passes.**

### A3.10 — "Have developers split large PRs into 3-4 file submissions"
**Why wrong:** Shifts burden to developers without fixing the system. **Multi-pass review architecture.**

### A3.11 — "Run 3 independent reviews and only flag issues appearing in 2+ runs"
**Why wrong:** Suppresses real bugs that may only be caught intermittently. **Multi-pass with focused per-file scope is better.**

### A3.12 — "Run `claude 'review this PR'` in CI"
**Why wrong:** Hangs waiting for interactive input. **Add `-p` (or `--print`) flag.**

### A3.13 — "Set `CLAUDE_HEADLESS=true` env var for CI"
**Why wrong:** This env var doesn't exist. **Use `-p` flag.**

### A3.14 — "Use `--batch` flag for non-interactive Claude Code"
**Why wrong:** This flag doesn't exist. **Use `-p`.**

### A3.15 — "Have the same Claude session that generated code review it"
**Why wrong:** Self-review bias from retained reasoning context. **Use an independent instance for review.**

### A3.16 — "Switch to a higher-tier model to fix the false positive rate"
**Why wrong (when prompt engineering hasn't been tried):** Model upgrade isn't the right lever for criteria precision issues. **Define explicit categorical criteria first.**

---

## Domain 4: Prompt Engineering Anti-Patterns

### A4.1 — "Tell the model to 'be conservative' or 'only report high-confidence findings'"
**Why wrong:** Vague instructions don't improve precision. **Use explicit categorical criteria with examples.**

### A4.2 — "Have model self-report confidence and filter at threshold"
**Why wrong:** LLM self-reported confidence is poorly calibrated. **Categorical criteria are reliable; confidence-as-quality-gate is not.**

### A4.3 — "Ask the model to respond in JSON format" (for guaranteed structured output)
**Why wrong:** Often produces syntax errors, markdown fences, chatty preambles. **Use `tool_use` + JSON schema.**

### A4.4 — "Mark all extraction fields as required for completeness"
**Why wrong:** Forces the model to fabricate values when source lacks them. **Mark fields nullable when they may be absent in source.**

### A4.5 — "Switch all workflows to batch API for cost savings"
**Why wrong (when blocking workflows are involved):** Batch has up to 24h SLA. **Use batch only for non-blocking; keep real-time for blocking.**

### A4.6 — "Use batch API with timeout fallback to real-time"
**Why wrong:** Adds complexity when matching each API to its appropriate workload is simpler.

### A4.7 — "Switch to batch API to avoid result ordering issues with synchronous calls"
**Why wrong:** Misconception. **Batch results correlate via `custom_id`; ordering isn't a real concern.**

### A4.8 — "Retry the extraction when required information is absent from the source"
**Why wrong:** Retries can't conjure data that doesn't exist. **Mark such fields nullable; retries succeed only for format/structure errors.**

### A4.9 — "Use extended thinking for the same instance to self-review"
**Why wrong:** Doesn't fully resolve self-review bias. **Use a fresh independent instance.**

---

## Domain 5: Context & Reliability Anti-Patterns

### A5.1 — "Rely on conversation summarization to preserve customer details"
**Why wrong:** Numbers, dates, IDs get lost in summarization. **Extract structured case facts persisted outside summarized history.**

### A5.2 — "Use sentiment analysis to detect cases needing escalation"
**Why wrong:** Frustration doesn't correlate with case complexity. **Use explicit escalation criteria.**

### A5.3 — "When customer says 'just give me a human,' attempt resolution first"
**Why wrong:** Ignores explicit request. **Honor immediately; escalate first.**

### A5.4 — "When `lookup_customer` returns multiple matches, pick the highest-spending"
**Why wrong:** Heuristic selection risks misidentification. **Ask the customer for additional identifiers.**

### A5.5 — "Subagent returns 'search unavailable' on timeout"
**Why wrong:** Generic status hides recovery context. **Return structured error with failure type, partial results, alternatives.**

### A5.6 — "Subagent terminates the entire workflow on a single failure"
**Why wrong:** Recoverable failures shouldn't kill the workflow. **Coordinator should attempt recovery from structured error context.**

### A5.7 — "Aggregate accuracy metric (97%) is sufficient for reducing human review"
**Why wrong:** Hides poor performance on specific document types or fields. **Validate per type and per field before reducing review.**

### A5.8 — "Random sampling for measuring extraction accuracy"
**Why wrong:** Oversamples common types, undersamples rare ones. **Stratified sampling for signal across all segments.**

### A5.9 — "Synthesis arbitrarily picks one value when sources conflict"
**Why wrong:** Loses provenance and methodology context. **Annotate conflicts with both source attributions.**

### A5.10 — "Source attribution can be reconstructed during summarization"
**Why wrong:** It's lost. **Require structured claim-source mappings preserved through synthesis.**

### A5.11 — "Speculative caching of context the synthesis agent might need"
**Why wrong:** Can't reliably predict what's needed. **Scoped tool exception for the common case.**

### A5.12 — "Batch all verification needs at the end of a synthesis pass"
**Why wrong:** Synthesis steps may depend on earlier verified facts; batching creates blocking dependencies. **Scoped exception for inline verification.**

---

## Cross-Cutting Anti-Patterns (Apply to Multiple Domains)

### X.1 — "Train a separate ML classifier"
**Why wrong (as first response to any LLM tooling problem):** Over-engineered. Requires labeled data and infrastructure. **Try prompt engineering or description fixes first.**

### X.2 — "Switch to a higher-tier / larger context window model"
**Why wrong (as fix for architectural issues):** Architecture problems aren't solved by bigger models. **Fix the architecture (decomposition, multi-pass, hooks).**

### X.3 — "Add comprehensive upfront instructions detailing exactly how it should work"
**Why wrong (when the work requires exploration):** Assumes you already know the answer; can't substitute for actual investigation. **Use plan mode or interview pattern.**

### X.4 — "Have Claude ask the user repeatedly for confirmation at each step"
**Why wrong:** Frustrates users and undermines autonomy. **Use programmatic enforcement for required gates; let Claude proceed otherwise.**

### X.5 — "Build a separate microservice / queue / scheduler infrastructure"
**Why wrong (when the issue is at the prompt/tool layer):** Infrastructure can't fix LLM behavior issues. **Address at the prompt or tool description layer first.**

---

## Anti-Pattern Recognition Drill

For each scenario, identify the wrong-answer pattern and the right approach:

**Q1.** "Add to the system prompt: 'You MUST verify customer identity before any refund.'"
→ Anti-pattern: A1.6 (prompt for required invariant) → Right: A1.4 hook with prerequisite gate

**Q2.** "Switch to Opus 4.7 to fix the false-positive rate."
→ Anti-pattern: X.2 (model upgrade for prompt issues) → Right: explicit categorical criteria

**Q3.** "Have the agent self-report confidence; route below 0.7 to humans."
→ Trap! This is OK for routing to humans (calibrated routing) but wrong as a quality gate. Read the question carefully.

**Q4.** "Place the testing standards in `.claude/CLAUDE.md` so all developers see them."
→ Trap! `.claude/CLAUDE.md` IS shared (not the same as `~/.claude/`). This is correct.

**Q5.** "Replace `analyze_content` and `analyze_document` with one `analyze` tool taking a `mode` parameter."
→ Anti-pattern: A2.3 (consolidation as fix for selection) → Right: better descriptions or split with focused names

---

## How to Use This in the Exam

When you read each multiple-choice question:

1. Read the scenario and the question.
2. **Before reading the options**, think: what's the root cause this scenario describes?
3. Read each option. Match against the anti-pattern catalog. Eliminate matches.
4. Of remaining options, pick the one that addresses the root cause most directly.
5. Verify your choice doesn't itself match an anti-pattern (some options sound right but trip on X.1-X.5).

This method, plus the 5 mental-model heuristics from `01-quick-start.md`, will resolve the vast majority of questions reliably.
