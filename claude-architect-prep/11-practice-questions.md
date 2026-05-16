# 11 — Practice Questions (40 MCQs)

> Distributed across domains by exam weight. Each question has 4 options (A-D); answer + reasoning at the bottom of each section. Cover the answers and work through them as a real exam would. Aim for 80%+ on a fresh pass to feel confident going in.

**Recommended:** Take this in three sittings, simulating exam conditions. Don't look back at the deep dives during the test.

---

## Section 1 — Domain 1: Agentic Architecture (11 questions, ~27%)

### Q1.1
A customer support agent should always call `get_customer` to verify identity before calling `process_refund`. In production, the agent skips verification 12% of the time. Which approach makes this requirement reliable?

A. Add a stronger system prompt instruction emphasizing the requirement
B. Add few-shot examples showing the correct sequence
C. Implement a `PreToolUse` hook that denies `process_refund` until `get_customer` has been called
D. Lower the temperature parameter to make the agent more deterministic

---

### Q1.2
A multi-agent research system has a coordinator and three subagents (web_searcher, doc_analyzer, synthesizer). Logs show the synthesis output misses entire subtopics, but each subagent's individual outputs look correct and complete. What's the root cause?

A. The synthesizer's context window is too small
B. The coordinator's task decomposition is too narrow, missing whole subtopic categories
C. The subagents are not propagating errors correctly
D. The synthesizer needs a more powerful model

---

### Q1.3
You're spawning subagents and notice they don't have access to information the coordinator has. What's the correct mental model?

A. Subagents inherit the coordinator's conversation history automatically
B. Subagents inherit the coordinator's tool results automatically
C. Subagents start with fresh context; only the prompt string crosses the boundary
D. Subagents share state via a global memory store managed by the SDK

---

### Q1.4
You want three search subagents to run in parallel rather than sequentially. How do you accomplish this in the Claude Agent SDK?

A. Set a `parallel: true` flag in the coordinator's options
B. The coordinator emits multiple `Agent` (formerly `Task`) tool calls in a single assistant response
C. Use Python's `asyncio.gather` to wrap subagent invocations
D. Use the `--parallel` flag when launching the SDK

---

### Q1.5
Three tools (`lookup_order`, `lookup_shipment`, `lookup_invoice`) return timestamps in three different formats: Unix epoch, ISO 8601, and a numeric status code. The agent wastes tokens reasoning about format conversion and occasionally misinterprets fields. What's the right fix?

A. Add a system prompt instruction explaining each format
B. Implement a `PostToolUse` hook that normalizes all timestamps to ISO 8601 before the model sees the result
C. Train a separate classification model to detect format mismatches
D. Force the agent to call a `convert_timestamp` tool after every tool result

---

### Q1.6
A coordinator runs all 4 subagents on every query, including simple ones that only need 1. What's the right pattern?

A. Always run all subagents to ensure consistency
B. Implement a routing classifier upstream of the coordinator
C. The coordinator should dynamically select subagents based on query requirements
D. Configure the SDK to skip subagents whose tools were not invoked

---

### Q1.7
You're using `claude --resume` to continue a session from yesterday, but yesterday's session loaded data via tool calls that may now be outdated (e.g., pricing data). What's the right approach?

A. Always resume — the agent will detect stale data via `stop_reason`
B. Start fresh and inject a structured summary of yesterday's findings
C. Resume but clear the tool result cache
D. Use `fork_session` instead of `--resume`

---

### Q1.8
A subagent encounters a transient error (network timeout). What should it do?

A. Immediately propagate the error to the coordinator with a generic "operation failed" message
B. Catch the exception silently and return empty results as success
C. Attempt local recovery (retry with backoff); propagate only if unrecoverable, with structured error context including partial results
D. Terminate the entire workflow to ensure no inconsistent state

---

### Q1.9
For a codebase task: "Add comprehensive tests to a 5-year-old legacy module with unclear dependencies and changing requirements." Which decomposition strategy is most appropriate?

A. Prompt chaining: process each file in a fixed sequence
B. Dynamic adaptive decomposition: map structure first, identify high-impact areas, create a prioritized plan that adapts as dependencies are discovered
C. Single comprehensive prompt with all files included
D. Iteration cap: process files in random order until budget exhausted

---

### Q1.10
The coordinator agent's prompt currently reads: "Step 1: invoke web_searcher. Step 2: invoke doc_analyzer. Step 3: invoke synthesizer. Always do all three in order." This produces inflexible behavior. What's a better prompt design?

A. Replace with detailed step-by-step pseudocode
B. Specify research goals, quality criteria, and let the coordinator decide which subagents to invoke based on the query
C. Remove the prompt entirely; let the coordinator infer everything
D. Add timeout values for each step

---

### Q1.11
You want to compare two architectural approaches (REST vs. GraphQL migration) starting from a shared codebase analysis baseline. What's the right session operation?

A. `--resume` the analysis session twice with different prompts
B. Use `fork_session` to create two parallel branches from the shared baseline
C. Start two independent fresh sessions
D. Use `/compact` to reduce context, then explore

---

### Section 1 Answers + Reasoning

| Q | Answer | Reasoning |
|---|--------|-----------|
| 1.1 | **C** | Required invariant → programmatic enforcement (hook). A/B are probabilistic. D doesn't change behavior on this requirement. |
| 1.2 | **B** | Logs show subagents working correctly within scope; the bug is upstream in decomposition. |
| 1.3 | **C** | Fresh context is the defining property. Pack everything needed into the prompt. |
| 1.4 | **B** | Multiple `Agent` tool calls in one response = parallel. No SDK flag for this; it's a model emission pattern. |
| 1.5 | **B** | `PostToolUse` hook for normalization. A is probabilistic; C/D are over-engineered. |
| 1.6 | **C** | Coordinator dynamic selection is the standard pattern. A wastes resources; B is over-engineered. |
| 1.7 | **B** | Stale tool results → fresh + structured summary. Resume preserves outdated context. |
| 1.8 | **C** | Local recovery first; structured propagation only if unrecoverable. |
| 1.9 | **B** | "Open-ended" + "adapts to discoveries" → dynamic decomposition. |
| 1.10 | **B** | Specify goals + criteria; let the coordinator decide. Procedural pseudocode kills adaptability. |
| 1.11 | **B** | `fork_session` is precisely for divergent exploration from a shared baseline. |

---

## Section 2 — Domain 2: Tools & MCP (7 questions, ~18%)

### Q2.1
Two tools, `analyze_content` and `analyze_document`, both have the description "Analyzes content." The agent misroutes between them ~40% of the time. What's the FIRST thing to try?

A. Build a routing classifier upstream of the agent
B. Add few-shot examples of correct routing
C. Rewrite the tool descriptions to differentiate purpose, inputs, outputs, and when-to-use
D. Consolidate the two tools into one with a `mode` parameter

---

### Q2.2
An MCP tool returns `{"isError": true, "message": "Operation failed"}` for every error condition. The agent retries the same way for every failure. What's wrong with this design?

A. The agent doesn't trust generic errors and stops calling the tool
B. Generic errors prevent the agent from making intelligent recovery decisions; structured errors with `errorCategory` and `isRetryable` are needed
C. MCP doesn't support `isError`; remove that field
D. The error message is too long

---

### Q2.3
The synthesis agent in your research system needs to verify simple facts in 85% of its work. Currently it routes through coordinator → web search subagent → back, adding 40% latency. What's the right architectural fix?

A. Give the synthesis agent full web search access
B. Cache web search results upstream
C. Give the synthesis agent its own scoped `verify_fact` tool for simple lookups; complex verifications still route through the coordinator
D. Replace the synthesis agent with a more powerful model

---

### Q2.4
Your team uses a JIRA MCP server. New team members report they don't have access to it after cloning the repo. The lead developer says it works for them. Where to look?

A. The MCP server requires individual API tokens
B. The configuration is in the lead's `~/.claude.json` (user-only) instead of the project's `.mcp.json`
C. The new team members' Claude Code is out of date
D. The MCP server protocol version mismatch

---

### Q2.5
You want to commit `.mcp.json` to git but it contains an API token. What's the right pattern?

A. Encrypt the file with git-crypt
B. Use `${ENV_VAR}` expansion in `.mcp.json`; each developer provides the token in their environment
C. Commit a template and have developers copy it locally with their token
D. Use `.gitignore` and require manual setup

---

### Q2.6
For force-feeding structured output, you want the model to call ONE specific tool first (`extract_metadata`) before normal flow. How?

A. `tool_choice: {"type": "any"}`
B. `tool_choice: {"type": "auto"}` with a system prompt instruction
C. `tool_choice: {"type": "tool", "name": "extract_metadata"}`
D. There's no way to force a specific tool

---

### Q2.7
Your custom MCP search tool is more capable than the built-in Grep, but the agent keeps preferring Grep. Why and what's the fix?

A. Built-in tools have higher priority by default; you can't override
B. Your tool's description is weaker than Grep's; strengthen it to explain capabilities and when to prefer it
C. Grep is faster, so the agent picks it; this is correct behavior
D. Disable Grep with `disallowed_tools`

---

### Section 2 Answers + Reasoning

| Q | Answer | Reasoning |
|---|--------|-----------|
| 2.1 | **C** | Tool descriptions are the primary selection mechanism. Address root cause first. |
| 2.2 | **B** | Structured errors with category enable intelligent recovery routing. |
| 2.3 | **C** | Scoped exception for the high-frequency common case; preserve separation for complex. |
| 2.4 | **B** | `.mcp.json` (project, committed) vs. `~/.claude.json` (user-only). Classic scoping mistake. |
| 2.5 | **B** | Env var expansion is the right pattern; commits the structure without secrets. |
| 2.6 | **C** | Specific tool name forces it; `any` only forces some tool, not THIS tool. |
| 2.7 | **B** | Description quality drives selection. Disabling Grep (D) is heavy-handed. |

---

## Section 3 — Domain 3: Claude Code (8 questions, ~20%)

### Q3.1
A new team member doesn't follow team conventions consistently. They say they have CLAUDE.md instructions on their machine. Where might the problem be?

A. They have the instructions in `~/.claude/CLAUDE.md` (personal-only) instead of the project's CLAUDE.md
B. Their Claude Code version is out of date
C. They need to run `/sync` to pick up team conventions
D. CLAUDE.md only applies to certain file types

---

### Q3.2
You want a `/review` slash command available to every developer who clones the repo. Where does it go?

A. `~/.claude/commands/review.md`
B. `<project>/.claude/commands/review.md`
C. In the project's `CLAUDE.md` as instructions
D. In `.claude/config.json` under a `commands` array

---

### Q3.3
Test files are spread throughout the codebase (e.g., `Button.test.tsx` next to `Button.tsx`). You want consistent test conventions everywhere. What's the right mechanism?

A. Place a CLAUDE.md in each subdirectory
B. Consolidate everything in root CLAUDE.md and rely on Claude to infer applicability
C. Create `.claude/rules/testing.md` with `paths: ["**/*.test.*"]` in YAML frontmatter
D. Create a skill that developers must manually invoke before editing tests

---

### Q3.4
You're restructuring a monolith into microservices, affecting 30+ files. Multiple valid architectural approaches exist. Plan mode or direct execution?

A. Direct execution with comprehensive upfront instructions
B. Direct execution; switch to plan mode if complexity emerges
C. Plan mode to evaluate approaches before changes
D. Direct execution; the implementation reveals natural service boundaries

---

### Q3.5
Your CI runs `claude "review this PR"` but the job hangs indefinitely. What's the fix?

A. Set `CLAUDE_HEADLESS=true` env var
B. Add the `-p` (or `--print`) flag for non-interactive mode
C. Pipe `< /dev/null` to the command
D. Add `--batch` flag

---

### Q3.6
A skill explores a large codebase reading dozens of files. Its verbose output pollutes the main conversation. What's the right configuration?

A. Increase the main conversation's context window
B. Add `context: fork` to the skill's frontmatter so it runs in isolated context
C. Reduce the skill's tool access
D. Move the skill to `~/.claude/skills/`

---

### Q3.7
You have multiple independent issues to fix in the codebase. Should you send them to Claude one at a time or all in one message?

A. All in one message, since context-switching is expensive
B. One at a time, since the issues are independent
C. Bundle them into a single message; the model handles them in parallel
D. It doesn't matter

---

### Q3.8
Your code-generation agent reviews its own output and approves obviously buggy code. Why?

A. The model isn't powerful enough; upgrade
B. Self-review bias from retained reasoning context; use an independent instance for review
C. The review prompt isn't strict enough
D. Same-session review is faster, so accuracy is sacrificed

---

### Section 3 Answers + Reasoning

| Q | Answer | Reasoning |
|---|--------|-----------|
| 3.1 | **A** | Personal `~/.claude/CLAUDE.md` isn't shared via git. Move to project CLAUDE.md. |
| 3.2 | **B** | Project-scoped slash commands live in `.claude/commands/`. |
| 3.3 | **C** | Files spread across dirs → glob-pattern rules in `.claude/rules/`. |
| 3.4 | **C** | Architectural + multi-file + multiple-approach → plan mode is required. |
| 3.5 | **B** | `-p` is the correct flag. CLAUDE_HEADLESS doesn't exist; --batch doesn't exist. |
| 3.6 | **B** | `context: fork` isolates the skill's verbose work from the main session. |
| 3.7 | **B** | Independent issues → sequential. (Interacting issues → one message.) |
| 3.8 | **B** | Self-review bias; independent instance with no prior reasoning context. |

---

## Section 4 — Domain 4: Prompt Engineering (8 questions, ~20%)

### Q4.1
Your code review agent has a high false-positive rate. You've already told it to "be conservative" with no improvement. Next step?

A. Switch to a higher-tier model
B. Define explicit categorical criteria — what to flag, what to skip — with code examples
C. Lower the temperature
D. Have the model self-report confidence and filter at threshold 0.7

---

### Q4.2
You need to extract data from invoices in JSON format with guaranteed schema compliance. What's the most reliable approach?

A. Ask the model in the prompt to "respond in JSON format"
B. Use `tool_use` with a JSON schema; force with `tool_choice: {"type": "tool", "name": "extract_invoice"}`
C. Use markdown code fences in the prompt
D. Post-process model output with a JSON cleaner

---

### Q4.3
Your extraction tool returns valid JSON syntax conforming to the schema. But for 8% of invoices, the line items don't sum to the stated total. What does this tell you?

A. The schema is broken
B. tool_use eliminates JSON syntax errors but does NOT eliminate semantic errors; add stated_total + calculated_total + match flag
C. The model is hallucinating
D. The invoices are corrupted

---

### Q4.4
Some source documents lack a `purchase_order_ref` field. Your schema marks it `required`. What's the consequence and the fix?

A. The extraction crashes; mark the field optional
B. The model fabricates plausible PO numbers when the field is required; mark it nullable so the model returns null honestly
C. The model returns an empty string; mark it optional
D. The model retries indefinitely; add a max_retries parameter

---

### Q4.5
You need to process 50,000 historical documents for classification. They aren't blocking any user; the report can be ready overnight. Real-time API or batch?

A. Real-time, for predictable latency
B. Batch (50% discount, ≤24h SLA fits the requirement)
C. Real-time with parallel requests
D. Batch with timeout fallback to real-time

---

### Q4.6
Your manager proposes switching pre-merge code reviews (which block developers) to the Batch API for cost savings. Why is this wrong?

A. Batch API doesn't support code review tasks
B. Batch API has up to a 24-hour SLA; developers can't wait that long for a blocking workflow
C. Batch API doesn't support tool use
D. Batch API is more expensive, not less

---

### Q4.7
A 14-file PR review produces inconsistent depth (detailed in some files, superficial in others, contradictory findings). The team proposes "use a larger context window." Why is this wrong?

A. Larger context windows don't exist for this model
B. Bigger context doesn't improve attention quality across the input; the right fix is multi-pass review (per-file + cross-file integration)
C. Cost is too high
D. It does fix the problem; the team is correct

---

### Q4.8
Source data has ambiguous document types beyond your enum (`invoice`, `receipt`, `purchase_order`, `credit_note`). What's the right schema design?

A. Add every possible document type to the enum
B. Use a free-form string field
C. Add `"other"` and `"unclear"` to the enum, with an `"other_detail"` field for the model to describe the type
D. Reject ambiguous documents

---

### Section 4 Answers + Reasoning

| Q | Answer | Reasoning |
|---|--------|-----------|
| 4.1 | **B** | Vague "be conservative" doesn't work; explicit categorical criteria is the standard fix. |
| 4.2 | **B** | tool_use + schema + forced choice is the gold standard for guaranteed structure. |
| 4.3 | **B** | Distinguish syntactic (tool_use solves) from semantic (need self-validation). |
| 4.4 | **B** | Required forces fabrication; nullable lets the model honestly say "absent". |
| 4.5 | **B** | Non-blocking + latency-tolerant = batch. |
| 4.6 | **B** | Up to 24h is unacceptable for blocking workflows. |
| 4.7 | **B** | Attention dilution isn't a context-size problem. |
| 4.8 | **C** | "Other" + detail + "unclear" preserves extensibility without forcing wrong buckets. |

---

## Section 5 — Domain 5: Context & Reliability (6 questions, ~15%)

### Q5.1
After 25 messages, your customer support agent forgets the customer's verified ID and asks for it again. What's the right architectural fix?

A. Increase the conversation's max_tokens
B. Disable conversation summarization
C. Extract a structured "case facts" block (customer_id, verification_status, etc.) and inject it into every prompt outside the summarized history
D. Switch to a model with a longer context window

---

### Q5.2
A customer says "Just give me a manager." The agent investigates first, then escalates. Why is this wrong?

A. The agent wastes time
B. The customer's explicit request for a human should be honored immediately with a structured handoff
C. Investigation is forbidden when escalating
D. The agent should ask for the customer ID first

---

### Q5.3
A `lookup_customer("John Smith")` returns 3 matches. The agent picks the most recent one heuristically and proceeds. What's wrong?

A. Most-recent isn't always correct; ask the customer for additional identifiers (email, postal code, last order date)
B. The lookup tool should only return one match
C. The agent should escalate immediately
D. Heuristic selection is fine for low-stakes cases

---

### Q5.4
Your subagent times out and returns "search unavailable" to the coordinator. Why is this insufficient?

A. The coordinator can't distinguish the failure type, recoverability, or what was partially achieved; need structured error context with failure_type, partial_results, alternatives
B. "Search unavailable" is too vague for the user
C. The coordinator should crash on subagent failures
D. The coordinator should retry indefinitely

---

### Q5.5
A multi-source synthesis agent compiles a report combining 4 sources with conflicting statistics. The output reads "studies show a 25% improvement." What's the issue?

A. 25% is wrong
B. Source attribution is lost; synthesis should preserve "Reuters (2025) reports 30%, McKinsey (2025) reports 18% — methodology differs (survey vs. telemetry)"
C. The synthesizer needed a more powerful model
D. The sources should have been validated upstream

---

### Q5.6
You want to reduce human review by 80% based on an aggregate accuracy metric of 97%. What should you check first?

A. Whether the model can be upgraded
B. Per-document-type and per-field accuracy — the 97% may hide poor performance on rare types or specific fields like `payment_terms`
C. Whether the model's confidence scores are above threshold
D. Whether the human reviewers are expensive

---

### Section 5 Answers + Reasoning

| Q | Answer | Reasoning |
|---|--------|-----------|
| 5.1 | **C** | Persistent case facts outside summarization. Increasing tokens (A) and bigger context (D) just delay the problem. |
| 5.2 | **B** | Honor explicit human requests immediately. |
| 5.3 | **A** | Ask for additional identifier; heuristic selection risks misidentification. |
| 5.4 | **A** | Generic status hides recovery context; structured error context enables intelligent routing. |
| 5.5 | **B** | Provenance must be preserved through synthesis; conflicts annotated, not arbitrated. |
| 5.6 | **B** | Aggregate metrics hide segment-level failures; stratified validation is required. |

---

## Final 5 — Cross-Domain Mixed Questions

### QF.1
You're designing an automated PR reviewer that runs in CI. Pick the BEST combined approach:

A. Real-time API + `claude` (no flags) + same-session review of generated code
B. Batch API + `-p` flag + multi-pass per-file + cross-file review with independent instance
C. Real-time API + `-p` flag + multi-pass per-file + cross-file review with independent instance + structured JSON output for inline PR comments
D. Real-time API + `-p` flag + single-pass review + plain text output

---

### QF.2
A research workflow needs to: (1) run 4 search subagents in parallel, (2) detect coverage gaps, (3) re-delegate gap-filling to specific subagents, (4) synthesize a final report preserving source attribution. Which combination is right?

A. Coordinator with sequential subagents + post-hoc review pass
B. Coordinator emitting parallel `Agent` calls + iterative refinement loop after synthesis evaluation + structured findings preserved through synthesis
C. Single mega-agent with 12 tools handling everything
D. Pipeline of fixed agents with no coordinator

---

### QF.3
Your customer support agent must (1) verify identity before refunds, (2) escalate refunds > $500, (3) preserve customer-stated specifics across long conversations, (4) handle "give me a human" requests immediately. Pick the best combined design:

A. System prompt with all four rules + larger context window + sentiment analysis for escalation
B. PreToolUse hook for verification + PreToolUse hook blocking > $500 with redirect to escalate_to_human + persistent case facts injected outside summarization + explicit escalation criteria with examples
C. PreToolUse hook for verification + amount limit checked in the tool itself + summarization disabled + LLM confidence threshold for escalation
D. Train custom routing classifiers for each rule

---

### QF.4
Your invoice extraction pipeline must process millions of documents nightly with field-level confidence scoring and human review for low-confidence extractions. Pick the best combined design:

A. Real-time API + tool_use + JSON schema + field_confidence in schema + threshold-based human routing + stratified accuracy validation per document type
B. Batch API + tool_use + JSON schema + field_confidence in schema + threshold-based human routing + stratified accuracy validation per document type
C. Batch API + freeform JSON in prompt + aggregate accuracy validation
D. Real-time API + freeform JSON in prompt + manual sampling

---

### QF.5
A team wants to roll out Claude Code with: (1) shared standards for testing and API conventions, (2) a `/review` slash command, (3) a personal `/standup` command for individual developers, (4) automatic test conventions on `*.test.ts` files anywhere, (5) a codebase-explorer skill that doesn't pollute the main conversation. Pick the best combined design:

A. Project CLAUDE.md + project `.claude/commands/review.md` + user `~/.claude/commands/standup.md` + `.claude/rules/testing.md` with `paths` glob + skill with `context: fork`
B. User CLAUDE.md + user commands + project rules + skill without context: fork
C. Project CLAUDE.md + project commands + project standup command + subdirectory CLAUDE.md for tests + regular skill
D. `.claude/config.json` with all configs

---

### Final Section Answers

| Q | Answer | Reasoning |
|---|--------|-----------|
| F.1 | **C** | Real-time (blocking) + -p (CI) + multi-pass + independent reviewer + structured output for inline comments. |
| F.2 | **B** | Parallel + iterative + structured findings is the canonical research architecture. |
| F.3 | **B** | Programmatic for invariants + persistent case facts + explicit escalation criteria. |
| F.4 | **B** | Batch (overnight, latency-tolerant) + structured + confidence + stratified validation. |
| F.5 | **A** | Project = shared (committed); user = personal; rules with globs for path patterns; context: fork for verbose skills. |

---

## Self-Assessment

Tally your score:
- **36-40 / 40 (90%+)**: Exam-ready. Time to do labs and review nuances.
- **32-35 / 40 (80-87%)**: Strong. Review missed questions and the corresponding deep dives.
- **28-31 / 40 (70-77%)**: Borderline. Re-read deep dives for the domains where you missed; redo this test in a week.
- **<28 / 40**: Significant gaps. Plan another full pass through the deep dives before retrying.

Track which **anti-patterns** you fell for. The most common trap pattern is your weakest area.
