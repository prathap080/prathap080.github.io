# 09 — Decision Frameworks ("When to Use What")

> The exam is structured around making the **right call** between plausible options. This file gives you decision trees for the seven recurring choices.

---

## Framework 1 — Programmatic Enforcement vs. Prompt Instruction

```
Is the requirement legal/financial/safety/regulatory/compliance?
├── YES → Programmatic (PreToolUse hook, prerequisite gate)
└── NO  → Will incorrect behavior have material consequences?
         ├── YES → Programmatic
         └── NO  → Could you tolerate ~95-99% reliability?
                  ├── YES → Prompt instruction
                  └── NO  → Programmatic
```

### Examples

| Requirement | Choice | Why |
|-------------|--------|-----|
| Verify customer ID before refund | **Hook** | Financial; reversal is costly |
| Check policy version before quoting price | **Hook** | Legal/contractual |
| Always greet customer politely | **Prompt** | Soft preference; ~99% is fine |
| Use markdown headings consistently | **Prompt** | Style; failures are cosmetic |
| Block PII from leaving the system | **Hook** (PostToolUse normalizer) | Compliance |
| Prefer Vitest over Jest in test files | **Prompt + .claude/rules** | Convention; tooling, not safety |

---

## Framework 2 — Plan Mode vs. Direct Execution

```
Are you making changes that affect multiple files / architecture / cross-cutting concerns?
├── YES → Plan mode
├── ARE there multiple valid approaches with different tradeoffs?
│   └── YES → Plan mode (regardless of file count)
└── Is the path forward clear (specific bug fix, scoped change)?
    └── YES → Direct execution
```

### Decision matrix

| Task | Mode | Why |
|------|------|-----|
| Fix off-by-one bug in `paginate()` | **Direct** | Clear, scoped |
| Restructure monolith → microservices (45+ files) | **Plan** | Architectural, multiple approaches |
| Migrate from Jest to Vitest | **Plan** | Cross-cutting, sequencing matters |
| Add date validation in one route | **Direct** | Single file, clear scope |
| Add retry logic to one HTTP client | **Direct** | Localized |
| Library migration affecting 45+ files | **Plan** | Multi-file, decisions about adoption strategy |
| Implement function whose signature is fixed | **Direct** | Clear contract |
| Design new caching layer | **Plan** | Multiple valid approaches |

### Combined pattern

For very large changes: **Plan mode → produce implementation plan → switch to direct execution to apply the plan**. The two modes are phases, not alternatives.

---

## Framework 3 — Real-Time API vs. Message Batches API

```
Will a human or another system block waiting for the response?
├── YES → Real-time
└── NO  → Is the workload going to take advantage of multi-turn tool calling?
         ├── YES → Real-time (batch doesn't support multi-turn within a single request)
         └── NO  → Can you tolerate up to 24 hours of latency?
                  ├── YES → Batch (50% discount)
                  └── NO  → Real-time
```

### Decision matrix

| Workload | API | Why |
|----------|-----|-----|
| Pre-merge code review (developer waits) | **Real-time** | Blocking |
| Customer support chat | **Real-time** | Interactive |
| Nightly technical debt report | **Batch** | Non-blocking, latency-tolerant |
| Backfill 50K historical document classifications | **Batch** | Bulk, no SLA |
| Multi-turn agent doing tool execution | **Real-time** | Batch lacks multi-turn |
| Weekly compliance audit | **Batch** | Scheduled, non-blocking |
| User-initiated extraction with progress UI | **Real-time** | User actively waiting |

### Mixed strategy

A common right answer: **batch for the bulk async work, real-time for interactive/blocking**. Don't switch everything to one or the other.

### Common trap

Wrong reasoning that batch is "often faster than real-time" so you can use it for blocking workflows with status polling. **Up to 24h means up to 24h.** Don't bet on faster.

---

## Framework 4 — Skill vs. CLAUDE.md vs. .claude/rules vs. Slash Command

```
Should this configuration be loaded automatically (no user invocation)?
├── YES → Path-specific or universal?
│         ├── Universal (every file) → CLAUDE.md (project root)
│         ├── Per directory → CLAUDE.md (in that directory)
│         └── Files spread across dirs by pattern → .claude/rules/<topic>.md with `paths` glob
└── NO (user invokes on demand) → Just instructions or a multi-step workflow?
         ├── Just instructions → Slash command (.claude/commands/<name>.md)
         └── Multi-step workflow with own context/tools → Skill (.claude/skills/<name>/SKILL.md)
```

### Examples

| Need | Mechanism |
|------|-----------|
| All test files use Vitest | `.claude/rules/testing.md` with `paths: ["**/*.test.*"]` |
| Project-wide TypeScript strict mode | Project-root `CLAUDE.md` |
| `/api` routes follow REST conventions | `<project>/api/CLAUDE.md` |
| `/review` standard checklist for PRs | `.claude/commands/review.md` |
| Multi-step refactoring workflow | Skill: `.claude/skills/refactor-async/SKILL.md` |
| Personal preference for verbose explanations | `~/.claude/CLAUDE.md` |
| Personal `/standup` command for status reports | `~/.claude/commands/standup.md` |

---

## Framework 5 — Skill `context: fork` Yes or No?

```
Will the skill produce verbose intermediate output (file reads, exploration, deep research)?
├── YES → context: fork (isolate; only summary returns to main)
└── NO  → Default (output goes into main conversation)
```

### Examples

| Skill | context: fork? |
|-------|---------------|
| Codebase architecture explorer | **Yes** — verbose discovery |
| Brief code formatter | **No** — output is the result |
| Refactor analyzer (reads many files) | **Yes** |
| Generate a function from a spec | **No** — output IS the deliverable |
| Deep research / literature review | **Yes** |

---

## Framework 6 — Escalate, Resolve, or Continue?

```
Did the customer explicitly ask for a human?
├── YES → Escalate immediately (with structured handoff)
└── NO → Is the request within agent capability AND policy?
         ├── YES → Resolve (acknowledge frustration if any, then act)
         └── NO  → Is policy silent or does it conflict?
                  ├── YES → Escalate (structured handoff with what was investigated)
                  └── NO  → Has the agent tried and failed to make progress?
                          ├── YES → Escalate
                          └── NO  → Continue investigating (don't escalate prematurely)
```

### What is NOT a valid escalation trigger

- Customer sentiment / frustration alone (if issue is in scope, resolve it)
- LLM self-reported confidence (poorly calibrated)
- Heuristic complexity assessment ("this looks complicated")
- Multiple matches on lookup (ask for additional identifier instead)

### Examples

| Situation | Action |
|-----------|--------|
| "Just give me a manager." | **Escalate** immediately |
| "I'm furious! Refund my $20!" (refund is straightforward) | **Resolve** + acknowledge frustration |
| Customer wants competitor price match; policy is silent | **Escalate** (policy gap) |
| `lookup_customer` returns 3 matches | **Continue** — ask for email or postal code |
| Refund > $500 (over policy limit) | **Escalate** (with hook + handoff) |
| Agent has tried 3 times to find the order, kept failing | **Escalate** |

---

## Framework 7 — Few-Shot vs. Explicit Criteria vs. Both

```
Does the task involve format consistency (output shape, JSON schema, naming)?
├── YES → Few-shot examples (show, don't tell)
├── Does the task involve judgment under ambiguity (which tool, which severity)?
│   └── YES → Few-shot examples WITH reasoning shown
└── Does the task have clear allow/deny categories (what to flag, what to skip)?
    └── YES → Explicit categorical criteria with examples
```

### Examples

| Task | Approach |
|------|----------|
| Consistent JSON output format | Few-shot (3 example outputs) |
| Code review precision | Categorical criteria (allow/deny lists) |
| Tool selection between similar tools | Few-shot with reasoning shown |
| Severity classification | Categorical criteria + code examples per level |
| Phone number normalization | Few-shot (input/output pairs) |
| Document type classification | Categorical with "other" + detail field |

---

## Framework 8 — Resume Session vs. Fresh Session with Summary vs. Fork

```
Is the prior context still valid (files unchanged, data fresh)?
├── YES → Are you exploring a divergent path from a baseline?
│         ├── YES → fork_session
│         └── NO  → --resume
└── NO  → Are some files modified since prior session?
         ├── YES → Resume + explicitly inform of changes
         └── NO  → Are tool results stale (data changed, env shifted)?
                  ├── YES → Start fresh + inject structured summary
                  └── NO  → --resume
```

### Examples

| Situation | Action |
|-----------|--------|
| Continuing yesterday's investigation, code unchanged | `--resume` |
| Prior session analyzed code; you fixed a bug; want to continue | `--resume` + tell agent what changed |
| Want to compare REST vs. GraphQL approaches from same baseline | `fork_session` |
| Prior session's API responses are stale (live data) | Fresh + summary |
| Prior session's CI run findings are stale (new commits) | Fresh + summary |

---

## Framework 9 — When to Use Tool_Choice Modes

```
Do you need to guarantee structured output (no text response allowed)?
├── YES → Specific tool required first, then auto?
│         ├── YES → tool_choice: {"type": "tool", "name": "..."}
│         └── NO  → tool_choice: {"type": "any"}
└── NO  → tool_choice: {"type": "auto"} (default)
```

### Examples

| Situation | Mode |
|-----------|------|
| Conversational agent, may or may not need tools | `auto` |
| Extraction pipeline that MUST output structured data | `any` |
| Pre-flight: must run `extract_metadata` before continuing | `{"type": "tool", "name": "extract_metadata"}` |
| Force one specific verification call | Specific named tool |

---

## Framework 10 — Multi-Pass Review or Single-Pass?

```
How many files / how much code?
├── 1-3 small files → Single-pass
├── 1-3 large files OR 4-6 small → Single-pass with focus
├── 4-10 mixed → Multi-pass (per-file + integration)
└── 10+ files → Multi-pass (per-file + integration), possibly in parallel
```

```
Are there cross-cutting concerns (data flow, naming consistency)?
├── YES → Add explicit cross-file integration pass
└── NO  → Per-file may be sufficient
```

### Why this matters

A 14-file PR reviewed in one pass shows attention dilution: detailed feedback for some files, superficial for others, contradictory findings. Splitting forces consistent attention quality per file.

---

## Master Summary — One-Page Decision Reference

| Question | Right answer |
|----------|--------------|
| Required invariant? | Hook |
| Soft preference? | Prompt |
| Architecture / multi-file / multiple approaches? | Plan mode |
| Single clear scope? | Direct execution |
| Blocking or interactive workload? | Real-time |
| Bulk async, ≥24h tolerance? | Batch |
| Universal project standard? | Project CLAUDE.md |
| Files spread by pattern? | .claude/rules/ |
| Subdirectory-specific? | Subdir CLAUDE.md |
| User-invoked workflow with own context? | Skill (with `context: fork` if verbose) |
| User-invoked instructions? | Slash command |
| Personal-only (any of above)? | `~/.claude/...` |
| Customer asks for human? | Escalate immediately |
| Policy gap? | Escalate |
| Multiple matches? | Continue + ask for ID |
| Format consistency? | Few-shot |
| Categorical filter? | Explicit criteria |
| Judgment under ambiguity? | Few-shot with reasoning |
| Guaranteed structure? | tool_use + JSON schema + tool_choice |
| Pre-flight required tool? | tool_choice with specific name |
| 14-file PR? | Multi-pass (per-file + integration) |
| Verbose discovery? | Subagent or `context: fork` skill |
| Self-review of generated code? | Independent instance, NOT same session |

This page alone resolves a large share of multiple-choice questions. Memorize the left column → reflex the right column.
