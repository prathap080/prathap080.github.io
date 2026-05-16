# 04 — Domain 2: Tool Design & MCP Integration (18%)

> Maps to: Customer Support, Multi-Agent Research, Developer Productivity scenarios. Smaller domain by weight, but the patterns here often appear inside Domain 1 questions too.

---

## What This Domain Tests

Five task statements:
- 2.1 Effective tool interfaces (descriptions, boundaries)
- 2.2 Structured error responses for MCP tools
- 2.3 Tool distribution across agents + `tool_choice` configuration
- 2.4 MCP server integration (.mcp.json, scoping, env vars)
- 2.5 Built-in tools (Read, Write, Edit, Bash, Grep, Glob)

The unifying theme: **the LLM picks tools by reading their descriptions** — invest in description quality before reaching for routing layers.

---

## 2.1 — Tool Descriptions: The Primary Selection Mechanism

### The fundamental rule

> **Tool descriptions are the primary mechanism LLMs use to select tools.**

If `analyze_content` and `analyze_document` both say "Analyzes content" — Claude will misroute roughly half the time. The fix is **not** more sophistication elsewhere; it's better descriptions.

### What goes in a great tool description

1. **Purpose** — what specific job this tool does
2. **Input format** — what the tool accepts, with examples
3. **Output format** — what it returns
4. **When to use** — vs. similar alternatives
5. **Edge cases** — null returns, empty results, error conditions
6. **Boundaries** — what it does NOT do

### Bad → Better → Best example

**Bad:**
```json
{
  "name": "lookup_order",
  "description": "Retrieves order details"
}
```

**Better:**
```json
{
  "name": "lookup_order",
  "description": "Retrieves details for a specific order by order ID."
}
```

**Best:**
```json
{
  "name": "lookup_order",
  "description": "Retrieves order details when given an order ID (format: ORD-XXXXX). Returns order status, line items, totals, ship address, and current shipping status. Returns null if order not found. Use this for order inquiries; for customer profile information use get_customer instead. Does not handle returns or refunds — use process_refund for those."
}
```

### The exam pattern

When the scenario says:
- "The agent calls `get_customer` for order queries instead of `lookup_order`. Both descriptions are minimal." → **Improve descriptions** (correct first step)
- "Add few-shot examples of correct routing." → wrong (adds tokens, doesn't fix root cause)
- "Build a routing classifier." → wrong (over-engineered, bypasses LLM's natural understanding)
- "Consolidate into one mega-tool." → wrong (architectural shift to fix description-level problem)

### Splitting overloaded tools

If a single `analyze_document` tool is being misused for multiple distinct purposes, **split it into purpose-specific tools** with defined input/output contracts:

```
Before:   analyze_document(doc, mode="summary"|"extract"|"verify")
After:    extract_data_points(doc, fields=[...])
          summarize_content(doc, max_words=N)  
          verify_claim_against_source(claim, doc)
```

Each new tool gets its own focused description. Each description tells Claude when to pick it over the others.

### System prompt interaction

Watch for this gotcha: a system prompt instruction like "always check for invoices first" can override well-written tool descriptions because LLMs are keyword-sensitive. If your tool descriptions are good but selection is still wrong, **review the system prompt for keyword bias**.

### 🔧 Practical Task 2.1
Take any 3 tools from your medical symptom checker (e.g., `lookup_snomed`, `search_drug_db`, `get_patient_history`) and write descriptions following the 6-element template above. Then test by giving Claude an ambiguous query like "find the code for this condition" and see which tool it picks. Iterate descriptions until it picks reliably.

---

## 2.2 — Structured Error Responses

### MCP's `isError` flag

MCP tools signal failures via the `isError` boolean flag in their response. But signaling failure isn't enough — the agent needs **structured metadata** to decide what to do next.

### Four error categories you need to distinguish

| Category | Example | Right response |
|----------|---------|----------------|
| **Transient** | Timeout, 503, network blip | Retry (with backoff) |
| **Validation** | Invalid input format | Fix input, retry once |
| **Business** | Refund > policy limit, customer not eligible | Communicate to user / escalate |
| **Permission** | Auth missing, scope denied | Escalate / prompt for credentials |

### The right error response shape

```json
{
  "isError": true,
  "errorCategory": "business",
  "isRetryable": false,
  "errorCode": "REFUND_LIMIT_EXCEEDED",
  "humanReadable": "This refund exceeds the $500 policy limit and requires manager approval.",
  "context": {
    "requested_amount": 750.00,
    "policy_limit": 500.00,
    "alternative": "escalate_to_human with policy exception payload"
  }
}
```

### The wrong error response shape

```json
{
  "isError": true,
  "message": "Operation failed"
}
```

This forces the agent to either retry blindly (wasting calls) or give up (failing the user). Generic errors hide the information needed for intelligent recovery.

### Distinguishing access failure from valid empty result

A query that runs successfully but matches nothing is **not an error** — it's a successful empty result. Returning it as an error causes the agent to retry pointlessly.

```json
// Valid empty result (success)
{ "results": [], "isError": false, "metadata": { "query_executed": "..." } }

// Access failure (error)
{ "isError": true, "errorCategory": "transient", "isRetryable": true, ... }
```

### Local recovery vs. propagation

Subagents should attempt **local recovery** for transient errors before propagating. They should propagate only:
- Errors they cannot resolve locally
- WITH partial results (what they did manage to get)
- WITH what they attempted (so the coordinator can try alternatives)

### 🔧 Practical Task 2.2
Wrap a flaky external API in an MCP tool. Implement four distinct error responses (transient timeout, validation error, business rule violation, permission denied), each with the structured shape above. Verify the agent retries the timeout, gives up on validation after one fix, communicates the business rule to the user, and escalates the permission error.

---

## 2.3 — Tool Distribution + `tool_choice`

### Tool overload degrades selection

The exam tests this directly: an agent given **18 tools** instead of **4-5** will misroute more often. Even if every tool is well-described, the model has to evaluate more options each turn.

**Right pattern:**
- Each subagent gets only the tools it needs for its role
- Limited cross-role tools for high-frequency needs (see scoped exception below)
- Complex cases route through the coordinator

**Scoped exception example (from the exam):**
> The synthesis agent needs to verify simple facts 85% of the time. It currently goes through the coordinator → web search subagent → back, adding 40% latency. The right fix is to give the synthesis agent its own scoped `verify_fact` tool for simple lookups, while complex verifications still route through the coordinator.

This is the **principle of least privilege** with a pragmatic carve-out.

### `tool_choice` configurations

```python
# Default: model decides whether to use a tool or respond with text
tool_choice = {"type": "auto"}

# Force model to call SOME tool (any one) — not text response
tool_choice = {"type": "any"}

# Force model to call THIS specific tool
tool_choice = {"type": "tool", "name": "extract_metadata"}
```

| Scenario | Use |
|----------|-----|
| Conversational agent that may or may not need tools | `auto` |
| Extraction pipeline: must produce structured output | `any` |
| Pre-flight: must run `extract_metadata` before enrichment | `{"type": "tool", "name": "extract_metadata"}` |

After the forced tool runs, the next turn returns to `auto` and follow-up steps proceed normally.

### Restricting cross-specialization

A web search agent should not have access to `process_refund`. A synthesis agent should not have access to `WebSearch` (unless scoped, per above). Confine each agent to its specialty:

```python
agents = {
    "researcher": AgentDefinition(
        ...,
        tools=["WebSearch", "WebFetch", "Read"],  # search/read only
    ),
    "synthesizer": AgentDefinition(
        ...,
        tools=["Read", "verify_fact"],  # scoped exception for high-frequency verification
    ),
    "writer": AgentDefinition(
        ...,
        tools=["Write", "Read"],  # can produce output, read context
    ),
}
```

### 🔧 Practical Task 2.3
Build two versions of a research agent:
1. v1 — give it 12 tools (search, fetch, summarize, extract, classify, translate, lookup_db, search_papers, ...)
2. v2 — give it just 4 tools (search, fetch, summarize, extract)

Run the same 5 queries through both. Measure: tool selection accuracy, average tokens per turn, latency. Verify v2 wins on the first two metrics.

---

## 2.4 — MCP Server Integration

### Scoping: project vs. user

| File | Scope | Shared via git? | Use for |
|------|-------|-----------------|---------|
| `.mcp.json` (in project root) | Project-wide | **Yes** | Team-shared MCP servers |
| `~/.claude.json` | User-only | No | Personal/experimental servers |

**The exam pattern:** "A new team member doesn't have access to the JIRA MCP server" → it's in `~/.claude.json` instead of `.mcp.json`.

### `.mcp.json` example with env var expansion

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "jira": {
      "command": "node",
      "args": ["./mcp-servers/jira/index.js"],
      "env": {
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}",
        "JIRA_DOMAIN": "${JIRA_DOMAIN}"
      }
    }
  }
}
```

**Why env var expansion?** So you can commit `.mcp.json` to git without committing secrets. Each developer's local environment provides the actual token.

### Multi-server is the default

Tools from **all** configured MCP servers are discovered at connection time and available simultaneously. You don't need to "switch" between servers — they all coexist in `allowedTools`.

### MCP tools vs. MCP resources

| | Tools | Resources |
|---|-------|-----------|
| Purpose | Take an action / fetch dynamic data | Expose static or semi-static **content catalogs** |
| Examples | `create_issue`, `lookup_order` | Issue summaries list, doc hierarchy, DB schemas |
| Effect | Causes a tool call | Reduces exploratory tool calls (agent sees what's available upfront) |

If your agent is making lots of exploratory `list_*` calls just to discover what's available, the right answer is often: **expose those catalogs as MCP resources** instead.

### Custom vs. community MCP servers

Choose existing community MCP servers for standard integrations (GitHub, JIRA, Slack, Linear, Notion). Build custom servers only for **team-specific workflows** that no community server addresses.

### Tool description battleground

Built-in tools (Grep, Glob) come with strong descriptions. If your custom MCP tool description is weaker, the agent will prefer Grep even when your MCP tool is more capable. **Enhance your MCP tool descriptions to explain capabilities and outputs in detail** so the agent picks them when appropriate.

### 🔧 Practical Task 2.4
1. Configure a `.mcp.json` with the GitHub MCP server using `${GITHUB_TOKEN}` env expansion. Verify it works without committing the token.
2. Add a personal experimental MCP server in `~/.claude.json` (e.g., a Wikipedia tool you're prototyping). Verify both servers' tools are simultaneously available.
3. Build a tiny MCP server that exposes a list of available datasets as a **resource** (not a tool). Compare agent behavior with vs. without the resource — does the agent stop making exploratory `list_datasets` calls?

---

## 2.5 — Built-in Tools

### The five core tools

| Tool | Purpose | Use when... |
|------|---------|-------------|
| **Grep** | Search file contents for patterns | Finding function callers, error messages, import statements |
| **Glob** | Match file paths by pattern | Finding files by name/extension (e.g., `**/*.test.tsx`) |
| **Read** | Load full file contents | Need entire file (or known range) |
| **Write** | Create/overwrite a file | Wholesale file creation |
| **Edit** | Targeted modification using unique text matching | Changing a small part of a known file |
| **Bash** | Execute shell commands | Tests, builds, file operations not covered above |

### Edit's failure mode and the fallback

`Edit` requires **unique text matching** — the `old_string` you're replacing must appear exactly once. When it doesn't (e.g., the same import statement repeats in many files, or a comment appears multiple times in the same file), `Edit` fails.

**Fallback:** `Read` to load the full file, modify in memory, `Write` to overwrite. Works every time, but uses more tokens.

### Building codebase understanding incrementally

❌ Reading every file upfront (token-heavy, attention-diluting)
✅ **Grep first** to find entry points and patterns → **Read** to follow imports → trace flows incrementally

Example: "Find all callers of `parse_invoice`":
1. `Grep "parse_invoice"` → list of files referencing it
2. `Read` the most relevant ones to understand context
3. Possibly `Grep "from invoice_parser import"` if you find aliases

### Tracing wrapper modules

Some functions are re-exported through wrapper modules:

```python
# in utils/__init__.py
from .invoice import parse_invoice as pi
from .invoice import parse_invoice
```

A naive `Grep "parse_invoice"` finds direct callers. To find users of the alias `pi`, you need:
1. Identify all exported names: `Grep "parse_invoice"` in `__init__.py` files
2. For each alias, search for that alias across the codebase

### 🔧 Practical Task 2.5
On your medical symptom checker repo (or any repo):
1. Find all places that call your `lookup_snomed_code` function across the codebase using Grep
2. Find all test files using Glob with pattern `**/test_*.py`
3. Try to use `Edit` to change a common import pattern that appears in many files — observe the failure
4. Use `Read` + `Write` as the fallback

---

## Domain 2 Summary — The Exam-Ready Cheat Sheet

| Concept | The right answer pattern |
|---------|--------------------------|
| Tools getting misrouted | Improve tool descriptions FIRST (purpose, inputs, outputs, when-to-use, edge cases, boundaries) |
| Two tools doing similar things | Rename + redescribe to differentiate. Or split. |
| Generic "Operation failed" errors | Replace with structured errors (errorCategory, isRetryable, humanReadable) |
| Empty query result | Return success with empty array — NOT an error |
| 18 tools on one agent | Scope to 4-5 per agent's role |
| 85% common-case fact-check overhead | Scoped exception: give the agent the simple verify tool directly |
| Need to guarantee structured output | `tool_choice: "any"` or forced tool name |
| Pre-flight extraction before enrichment | `tool_choice: {"type": "tool", "name": "extract_metadata"}` |
| Team-shared MCP server | `.mcp.json` (project root, committed to git) |
| Personal experimental MCP server | `~/.claude.json` (user-only) |
| Secrets in MCP config | `${ENV_VAR}` expansion in `.mcp.json` |
| Standard integration (Jira, GitHub) | Use community MCP server, don't build custom |
| Lots of exploratory `list_*` calls | Expose catalogs as MCP **resources** |
| MCP tool being passed over for built-in Grep | Strengthen your MCP tool's description |
| Edit fails on non-unique text | Fall back to Read + Write |
| Mapping unfamiliar codebase | Grep entry points → Read selectively → trace incrementally |

---

## Domain 2 Drill Questions (For Self-Test)

1. Two tools (`analyze_content`, `analyze_document`) get confused by the agent. First fix?
2. Your tool returns "Operation failed" for everything. What's wrong?
3. Subagents are losing 40% of their time on cross-coordinator round-trips for simple fact checks. Right fix?
4. New team member doesn't get the JIRA MCP server. Where to look?
5. Agent prefers Grep over your custom search MCP tool that's more capable. Fix?
6. Agent makes 12 exploratory `list_*` calls before doing real work. Better pattern?

(Answers: 1) Improve descriptions to differentiate purpose 2) Generic errors prevent intelligent recovery — return structured errorCategory + isRetryable 3) Scoped tool exception — give synthesis agent its own simple verify tool 4) Should be in `.mcp.json` (committed), not `~/.claude.json` 5) Strengthen MCP tool description to explain capabilities clearly 6) Expose those catalogs as MCP resources)
