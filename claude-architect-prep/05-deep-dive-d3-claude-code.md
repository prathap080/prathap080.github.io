# 05 — Domain 3: Claude Code Configuration & Workflows (20%)

> **The most "new" domain for you** — Claude Code is Anthropic-specific tooling you haven't encountered in Strands or Azure. Maps to: Code Generation, Developer Productivity, CI/CD scenarios.

---

## What This Domain Tests

Six task statements:
- 3.1 CLAUDE.md hierarchy and modular organization
- 3.2 Custom slash commands and skills
- 3.3 Path-specific rules (`.claude/rules/`)
- 3.4 Plan mode vs. direct execution
- 3.5 Iterative refinement techniques
- 3.6 CI/CD pipeline integration

The unifying theme: **configure Claude Code so the right context is loaded at the right time** for the right user, automatically.

---

## 3.1 — CLAUDE.md Hierarchy

### The three levels (memorize the precedence)

```
~/.claude/CLAUDE.md                  ← USER level (NOT shared via git)
                                       Personal preferences, individual workflows

<project>/CLAUDE.md                  ← PROJECT level (shared via git)
or                                     Team-wide standards, conventions
<project>/.claude/CLAUDE.md            

<project>/<dir>/CLAUDE.md            ← DIRECTORY level (shared via git)
                                       Subdirectory-specific rules
```

### The exam's canonical scenario

> "A new team member follows the team's coding standards inconsistently. They have CLAUDE.md instructions on their machine but other developers don't see the same behavior."

**Diagnosis:** The instructions are in `~/.claude/CLAUDE.md` (user-level, not shared) instead of the project's CLAUDE.md (shared via git).

**Fix:** Move the instructions to the project root's CLAUDE.md.

### `@import` for modularity

A monolithic project CLAUDE.md becomes unmaintainable. Use `@import` to compose:

```markdown
# Project CLAUDE.md

## Standards (shared)
@import ./.claude/standards/typescript.md
@import ./.claude/standards/testing.md
@import ./.claude/standards/api-conventions.md

## Project-specific
- This monorepo uses pnpm workspaces
- All API endpoints versioned under /v2/
```

Each package can have its own CLAUDE.md that imports only the standards relevant to its maintainer's domain.

### `.claude/rules/` directory as alternative

For larger projects, instead of one big CLAUDE.md, organize topic-specific rules into `.claude/rules/`:

```
.claude/
├── rules/
│   ├── testing.md
│   ├── api-conventions.md
│   ├── deployment.md
│   └── security.md
└── CLAUDE.md  # Imports the above as needed
```

### `/memory` command — diagnostic

When developers see inconsistent behavior across sessions, run `/memory` to verify which memory files are actually loaded. Common discoveries:
- Personal CLAUDE.md is overriding project CLAUDE.md unexpectedly
- A subdirectory CLAUDE.md isn't loading because of its location
- @imports are silently failing

### 🔧 Practical Task 3.1
1. Set up CLAUDE.md hierarchy on a real project: project root, plus a subdirectory-specific one
2. Use `@import` to factor common standards into separate files
3. Run `/memory` to verify what's loaded
4. Move some instructions to `~/.claude/CLAUDE.md` and observe that they're personal-only

---

## 3.2 — Custom Slash Commands and Skills

### Slash commands — definition and scope

Custom slash commands are markdown files. Their scope is determined by their location:

| Location | Scope | Shared? |
|----------|-------|---------|
| `<project>/.claude/commands/` | Project-wide | **Yes** (via git) |
| `~/.claude/commands/` | User-only | No |

Each command file becomes a slash command. `/.claude/commands/review.md` → invokable as `/review`.

### Example: `/review` slash command

```markdown
---
description: Run team's standard code review checklist
argument-hint: [files-or-PR-number]
allowed-tools: ["Read", "Grep", "Glob"]
---

You are performing a code review using our team's checklist:

1. Type safety: any `any` types must be justified or removed
2. Error handling: all `await` calls in try/catch
3. Test coverage: new functions must have corresponding test cases
4. Logging: no `console.log` — use `logger.info/warn/error`
5. Comments: only flag when claimed behavior contradicts actual code behavior

Review the following: $ARGUMENTS
```

### The exam's canonical scenario

> "I want a `/review` command available to every developer when they clone the repo."

**Right answer:** `<project>/.claude/commands/review.md`
**Wrong:** `~/.claude/commands/` (personal only — wouldn't be shared)
**Wrong:** Adding it to CLAUDE.md (CLAUDE.md is for context, not command definitions)
**Wrong:** A `.claude/config.json` with a commands array (this mechanism doesn't exist)

### Skills — when and how

A **skill** is a reusable capability bundled with its own context, prompt, and tools. Lives in `.claude/skills/<name>/SKILL.md`.

```yaml
---
name: refactor-to-async
description: Convert callback-based code to async/await patterns
context: fork                           # ← runs in isolated subagent context
allowed-tools: ["Read", "Edit", "Write"]
argument-hint: [file-path]
---

You are an expert at refactoring callback patterns to async/await.

When invoked, you will:
1. Read the target file
2. Identify callback patterns (Node-style err-first, Promise.then chains)
3. Rewrite as async/await with proper error handling
...
```

### `context: fork` — the most-tested skill option

Without `context: fork`: skill output goes into the main conversation, polluting context with verbose discovery/exploration.

With `context: fork`: skill runs in an **isolated subagent context** — only the final summary returns to the main session. Use this for any skill that produces verbose intermediate output (codebase analysis, exploratory brainstorms, deep research).

### `allowed-tools` in skill frontmatter

Restrict tool access during the skill's execution. Common patterns:
- A "drafting" skill that should only Read and Write — never Bash
- A "search" skill restricted to Grep + Glob + WebSearch — no file modifications

### `argument-hint` frontmatter

Prompts the developer for required parameters when they invoke the skill without arguments. Improves discoverability and prevents silent failures.

### Skills vs. CLAUDE.md — when to use which

| Use CLAUDE.md when... | Use a skill when... |
|----------------------|---------------------|
| Always-loaded universal standards | Task-specific workflow invoked on demand |
| Coding conventions everyone needs | Capability used occasionally by some developers |
| Project glossary / domain knowledge | Multi-step procedure with its own prompt |

### Personal skill variants

To customize a project skill without affecting teammates: create a personal variant in `~/.claude/skills/` with a **different name** (e.g., team has `/.claude/skills/refactor`, you create `~/.claude/skills/refactor-mine`). Don't override the team's skill name — that creates confusion.

### 🔧 Practical Task 3.2
1. Create a `/check-pr` slash command in `.claude/commands/` with frontmatter and `$ARGUMENTS` interpolation
2. Create a skill `codebase-explorer` with `context: fork` and verify its output doesn't pollute your main session
3. Create a personal variant of the skill in `~/.claude/skills/` with a different name

---

## 3.3 — Path-Specific Rules (`.claude/rules/`)

### The problem these solve

You have test files spread throughout the codebase: `Button.test.tsx` next to `Button.tsx`, `useAuth.test.ts` next to `useAuth.ts`, and so on. You want **all tests** to follow the same conventions, regardless of where they live.

You can't put a CLAUDE.md "for tests" in any one directory — tests are everywhere.

### The right pattern: glob-pattern rules

Create `.claude/rules/testing.md`:

```yaml
---
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
  - "**/*.spec.tsx"
  - "**/*.spec.ts"
---

When editing or generating test files:
- Use Vitest (NOT Jest)
- Tests should follow the AAA pattern (Arrange-Act-Assert)
- Use `describe.concurrent` for parallel suites
- Mock external dependencies with `vi.mock`
- File should match the implementation file's name with `.test.` infix
```

This rule loads **only when editing files matching the glob patterns**, regardless of directory.

### The exam's canonical scenario

> "Test files are spread throughout the codebase alongside the code they test. You want all tests to follow the same conventions regardless of location."

**Right:** `.claude/rules/testing.md` with `paths: ["**/*.test.tsx"]`
**Wrong:** Place CLAUDE.md in each subdirectory (tests are spread across many dirs)
**Wrong:** Consolidate everything in root CLAUDE.md (relies on Claude inferring which rules apply)
**Wrong:** Use a skill (requires manual invocation, not automatic)

### When to use each mechanism

| If conventions apply to... | Use |
|---------------------------|-----|
| Universally every file in the project | Project-root CLAUDE.md |
| Every file in one directory | Subdirectory CLAUDE.md (in that directory) |
| Files matching a pattern, regardless of location | `.claude/rules/` with `paths` glob |
| A specific multi-step workflow on demand | Skill |

### 🔧 Practical Task 3.3
1. Create `.claude/rules/api-handlers.md` with `paths: ["src/api/**/*.ts"]` and conventions for async/await + error handling
2. Create `.claude/rules/react-components.md` with `paths: ["**/*.tsx"]` excluding tests
3. Edit a file in each pattern and verify the right rule loads (use `/memory` to confirm)

---

## 3.4 — Plan Mode vs. Direct Execution

### When to use which

**Plan mode** is for tasks that benefit from exploration and design **before** any changes:
- Large-scale changes (e.g., monolith → microservices)
- Multiple valid approaches with different tradeoffs
- Architectural decisions (library migration affecting 45+ files)
- Multi-file modifications with cross-cutting impact

**Direct execution** is for tasks where the path is clear:
- Single-file bug fix with a clear stack trace
- Adding a date validation conditional
- Renaming a variable across known locations
- Implementing a function whose signature is fixed

### The exam's canonical scenarios

✅ **"Restructure the monolith into microservices, dozens of files affected, decisions about service boundaries needed."** → Plan mode

❌ **"Start with direct execution; the implementation will reveal natural service boundaries."** → Wrong; this risks costly rework.

❌ **"Use direct execution with comprehensive upfront instructions."** → Wrong; assumes you already know the right structure.

❌ **"Begin direct, switch to plan mode if complexity emerges."** → Wrong; the complexity is *already stated* in the requirements.

### Combining plan + direct

A common pattern: **plan the migration in plan mode, then execute the planned approach with direct execution.** The two modes are complementary phases of one workflow, not either/or.

### The Explore subagent

When you need to do verbose discovery (read many files, trace dependencies, map an unfamiliar area) but don't want that output filling your main context — use the **Explore subagent**. It runs the discovery in isolation and returns a summary.

This pattern parallels the `context: fork` skill option but is more general-purpose.

### 🔧 Practical Task 3.4
On any non-trivial open-source repo:
1. Pick a "simple, well-scoped" task — use direct execution. Time it.
2. Pick an "architectural" task — use plan mode. Compare the quality of the resulting plan vs. what you'd have stumbled into with direct execution.
3. Use the Explore subagent on a part of the codebase you don't know — verify the summary it returns is genuinely useful without polluting your main session.

---

## 3.5 — Iterative Refinement Techniques

### Four techniques, each with a clear use case

#### Technique 1: Concrete input/output examples

When prose descriptions yield inconsistent results, **show 2-3 input/output pairs**. This is more effective than refining the prose.

```
Input: "  John  Doe  "
Output: "John Doe"

Input: "MARY-JANE"  
Output: "Mary-Jane"

Input: ""
Output: null
```

#### Technique 2: Test-driven iteration

Write the test suite first. Iterate by sharing test failures with Claude. This works because tests are unambiguous specifications and failures are precise feedback.

#### Technique 3: The interview pattern

Have Claude **ask clarifying questions before implementing**. Especially valuable in unfamiliar domains where you might miss design considerations.

> "Before implementing the cache, ask me 5 questions about how cache invalidation should work, what failure modes matter, and what the access patterns look like."

#### Technique 4: Sequential vs. parallel issue resolution

| Issues are... | Approach |
|---------------|----------|
| **Independent** of each other | Fix sequentially (one issue per round) |
| **Interacting** (one fix affects another) | Bundle into a single detailed message with all the issues |

The exam asks both directions:
- "Multiple issues that interact (a bug fix breaks unrelated tests)" → single message with all
- "Several independent issues" → sequential

### 🔧 Practical Task 3.5
1. On a transformation task (e.g., "normalize phone numbers"), first try with prose alone. Then add 3 input/output examples. Compare consistency.
2. Apply test-driven iteration: write tests first, paste failures, iterate
3. Use the interview pattern on a domain you don't know well (e.g., "design a rate limiter") — ask Claude to ask you 5 questions first
4. Practice both: send 4 independent fixes one-by-one, then send 4 interacting fixes as one message

---

## 3.6 — Claude Code in CI/CD Pipelines

### Non-interactive mode: the `-p` / `--print` flag

```bash
# WRONG — will hang waiting for input
claude "Analyze this PR for security issues"

# RIGHT — non-interactive, prints result, exits
claude -p "Analyze this PR for security issues"
```

The exam tests this directly. Wrong distractors:
- ❌ `CLAUDE_HEADLESS=true` env var (doesn't exist)
- ❌ `--batch` flag (doesn't exist)
- ❌ `< /dev/null` redirect (doesn't address the issue properly)

### Structured output for machine parsing

```bash
claude -p "Review PR #1234" \
  --output-format json \
  --json-schema review-schema.json
```

This produces machine-parseable findings that your CI script can post as inline PR comments.

### CLAUDE.md in CI

CLAUDE.md is the mechanism for providing **project context to CI-invoked Claude Code**:
- Testing standards
- Fixture conventions
- Review criteria (severity definitions, what to flag, what to skip)
- Available test helpers and patterns

This dramatically improves the quality of automated review and test generation.

### Session context isolation: don't review with the same session

A model that just **generated** code retains its reasoning context — it's biased toward seeing the code as correct. Reviewing it from the same session is less effective than:

✅ **Spawning an independent review instance** with no prior reasoning context — it sees the code with fresh eyes.

This is one of the most important architectural principles in Domain 4 too — the multi-instance review pattern.

### Re-running reviews after new commits

When new commits land and the review re-runs, you don't want **duplicate comments** for issues already discussed. The pattern:

1. Include prior review findings in the context
2. Instruct: "Report only NEW or still-UNADDRESSED issues."
3. The model compares the current diff against the prior findings.

### Test generation gotcha

When generating tests for a module, **provide the existing test files in context**. Otherwise the agent regenerates duplicate scenarios already covered, wasting effort and creating noise.

### 🔧 Practical Task 3.6
1. Set up a GitHub Actions workflow that runs `claude -p "review" --output-format json --json-schema X` on PRs and posts findings as inline comments
2. Add a CLAUDE.md with explicit review criteria; verify findings improve in quality
3. Configure the workflow to skip re-flagging issues already addressed in prior commits
4. Generate tests for a module while providing the existing tests — verify no duplicates

---

## Domain 3 Summary — The Exam-Ready Cheat Sheet

| Concept | The right answer pattern |
|---------|--------------------------|
| Team-shared standards | Project-root `CLAUDE.md` (committed) |
| Personal preferences | `~/.claude/CLAUDE.md` (NOT shared) |
| Subdirectory-specific rules | `<dir>/CLAUDE.md` |
| Files spread across dirs (e.g., tests) | `.claude/rules/<topic>.md` with `paths` globs |
| Modular organization | `@import` external files |
| Diagnostic for "which configs are loaded?" | `/memory` |
| Team-shared slash command | `<project>/.claude/commands/<name>.md` |
| Personal slash command | `~/.claude/commands/<name>.md` |
| Skill that produces verbose output | `context: fork` in frontmatter |
| Restrict skill's tool access | `allowed-tools` in frontmatter |
| Skill prompts user for params | `argument-hint` in frontmatter |
| Personal customization of project skill | New name in `~/.claude/skills/` |
| Architectural / multi-file / multiple-approach task | Plan mode |
| Single-file fix with clear scope | Direct execution |
| Verbose discovery polluting context | Explore subagent (or `context: fork` skill) |
| Inconsistent transformation results | Concrete input/output examples |
| Building toward a precise spec | Test-driven iteration |
| Unfamiliar domain | Interview pattern (ask Claude to ask you questions) |
| Multiple interacting issues | Single detailed message |
| Multiple independent issues | Sequential one-by-one |
| Running Claude Code in CI | `-p` / `--print` flag |
| Machine-parseable output in CI | `--output-format json --json-schema` |
| Re-review after new commits | Include prior findings, report only new/unaddressed |
| Avoid generating duplicate tests | Provide existing tests in context |
| Reviewing your own generated code | Use independent instance, not same session |

---

## Domain 3 Drill Questions (For Self-Test)

1. New team member doesn't follow team conventions despite having Claude Code installed. Where to look?
2. You want `/review` to be available to all developers after `git pull`. Where does it go?
3. Test files spread throughout codebase need consistent rules. Right mechanism?
4. Restructuring monolith → microservices, 30+ files. Plan mode or direct?
5. CI job hangs on `claude "review this"`. Fix?
6. Test generation keeps duplicating existing tests. Right adjustment?
7. The same agent that wrote the code reviews it and misses obvious bugs. Why and what to do?

(Answers: 1) Likely user-level CLAUDE.md instead of project-level 2) `<project>/.claude/commands/review.md` 3) `.claude/rules/testing.md` with `paths: ["**/*.test.*"]` 4) Plan mode — architectural decisions involved 5) Add `-p` flag for non-interactive mode 6) Provide existing test files in the context window 7) Same-session bias retains reasoning context — use an independent instance for review)
