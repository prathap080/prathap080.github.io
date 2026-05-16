# 12 — Four-Week Study Plan

> **The surprise addition.** A day-by-day calendar that integrates every file in this pack with your real-life schedule. Designed for ~1.5 hours/weekday + ~3 hours/weekend day. Total: ~14 hrs/week × 4 weeks = ~56 hours.

---

## Pacing Philosophy

This plan distributes effort by the same proportions as the exam:

| Week | Focus | Hours | Why |
|------|-------|-------|-----|
| Week 1 | Foundations + Domain 1 (largest) | 14 | Front-load the biggest domain when energy is fresh |
| Week 2 | Domain 2 + Domain 3 | 14 | Mid-weight domains; D3 is the most "new" for you |
| Week 3 | Domain 4 + Domain 5 | 14 | Wrap remaining domains; you already know prompt engineering well |
| Week 4 | Integration + practice + final review | 14 | Labs, mock exams, weakest-area triage |

**Built-in flexibility:** If you fall behind, the buffer days (Sundays) absorb slip. If you finish early, use buffer time to redo labs or build your own practice questions.

---

## Week 1 — Foundations + Domain 1 (Agentic Architecture, 27%)

### Monday — Orient
**Time:** 1.5 hrs
- [ ] Read `00-START-HERE.md` and `01-quick-start.md` (45 min)
- [ ] Open `cheatsheet.html` in a browser tab; bookmark it (5 min)
- [ ] Skim `02-strands-azure-translation.md` — flag concepts that surprise you (40 min)

**Deliverable:** A short note in your own words: "What I already know vs. what's new."

### Tuesday — Domain 1 Part A
**Time:** 1.5 hrs
- [ ] Read `03-deep-dive-d1-agentic-architecture.md` sections 1.1 to 1.3 (60 min)
- [ ] Practical Task 1.1: write the agentic loop and break it three ways (30 min)

### Wednesday — Domain 1 Part B
**Time:** 1.5 hrs
- [ ] Read `03-deep-dive-d1-agentic-architecture.md` sections 1.4 to 1.7 (50 min)
- [ ] Practical Task 1.4: implement the customer verification hook (40 min)

### Thursday — Domain 1 Synthesis
**Time:** 1.5 hrs
- [ ] Re-read your medical symptom checker mapping table in `02-strands-azure-translation.md` (15 min)
- [ ] Domain 1 drill questions at the bottom of file 03 (20 min)
- [ ] Practical Task 1.5: implement the PostToolUse normalization hook (45 min)
- [ ] Reflect: which Domain 1 patterns does your CRO architecture use? (10 min)

### Friday — Anti-Pattern Recognition for Domain 1
**Time:** 1.5 hrs
- [ ] Read the Domain 1 section of `08-anti-patterns-reference.md` (45 min)
- [ ] Domain 1 questions in `11-practice-questions.md` Section 1 (Q1.1 - Q1.11) (40 min)
- [ ] Note the anti-patterns you fell for (5 min)

### Saturday — Lab 1
**Time:** 3 hrs
- [ ] Set up the working environment per `10-hands-on-labs.md` setup section (30 min)
- [ ] Build Lab 1 — Customer Support Agent in full (2 hrs)
- [ ] Answer Lab 1 reflection questions (30 min)

### Sunday — Buffer / Catch-up / Review
**Time:** 2 hrs
- [ ] If on track: redo any missed Domain 1 practice questions
- [ ] If behind: catch up on what you missed
- [ ] Update your "things I keep forgetting" note

**Week 1 milestone:** You can articulate the hub-and-spoke pattern, the agentic loop, the hook-vs-prompt distinction, and recognize the major Domain 1 anti-patterns at a glance.

---

## Week 2 — Domain 2 (Tools & MCP, 18%) + Domain 3 (Claude Code, 20%)

### Monday — Domain 2
**Time:** 1.5 hrs
- [ ] Read `04-deep-dive-d2-tools-mcp.md` sections 2.1 to 2.3 (50 min)
- [ ] Practical Task 2.1: rewrite three tool descriptions following the 6-element template (40 min)

### Tuesday — Domain 2 (continued)
**Time:** 1.5 hrs
- [ ] Read `04-deep-dive-d2-tools-mcp.md` sections 2.4 to 2.5 (45 min)
- [ ] Practical Task 2.4: configure `.mcp.json` with env var expansion (45 min)

### Wednesday — Domain 2 Wrap
**Time:** 1.5 hrs
- [ ] Read Domain 2 anti-patterns in `08-anti-patterns-reference.md` (30 min)
- [ ] Domain 2 questions in `11-practice-questions.md` Section 2 (Q2.1 - Q2.7) (30 min)
- [ ] Domain 2 drill questions at the bottom of file 04 (20 min)
- [ ] Compare: what's different about MCP vs. Bedrock tool integration in your stack? (10 min)

### Thursday — Domain 3 Part A
**Time:** 1.5 hrs
- [ ] Read `05-deep-dive-d3-claude-code.md` sections 3.1 to 3.3 (60 min)
- [ ] Practical Task 3.1: set up CLAUDE.md hierarchy on a real project (30 min)

### Friday — Domain 3 Part B
**Time:** 1.5 hrs
- [ ] Read `05-deep-dive-d3-claude-code.md` sections 3.4 to 3.6 (50 min)
- [ ] Practical Task 3.6: try the `-p` flag in a script (40 min)

### Saturday — Domain 3 Lab
**Time:** 3 hrs
- [ ] Build Lab 4 — Claude Code Project (2 hrs)
- [ ] Domain 3 anti-patterns in `08-anti-patterns-reference.md` (30 min)
- [ ] Domain 3 practice questions Section 3 (Q3.1 - Q3.8) (30 min)

### Sunday — Buffer + Decision Frameworks
**Time:** 3 hrs
- [ ] Read `09-decision-frameworks.md` end-to-end (60 min)
- [ ] Memorize the master summary table at the bottom (30 min)
- [ ] Catch-up on any tasks
- [ ] Self-test: cover the right column of the master summary; recite the right answer for each question (30 min)

**Week 2 milestone:** You can apply Frameworks 1-9 from file 09 in 5 seconds when you see a question. You've configured a real `.claude/` directory and seen everything load.

---

## Week 3 — Domain 4 (Prompt Engineering, 20%) + Domain 5 (Context & Reliability, 15%)

### Monday — Domain 4 Part A
**Time:** 1.5 hrs
- [ ] Read `06-deep-dive-d4-prompt-engineering.md` sections 4.1 to 4.3 (60 min)
- [ ] Practical Task 4.1: rewrite a code review prompt with categorical criteria (30 min)

### Tuesday — Domain 4 Part B
**Time:** 1.5 hrs
- [ ] Read `06-deep-dive-d4-prompt-engineering.md` sections 4.4 to 4.6 (50 min)
- [ ] Practical Task 4.5: set up a small batch with the Message Batches API (40 min)

### Wednesday — Domain 4 Lab
**Time:** 1.5 hrs
- [ ] Build Lab 3 — Structured Data Extraction Pipeline (90 min)

### Thursday — Domain 5 Part A
**Time:** 1.5 hrs
- [ ] Read `07-deep-dive-d5-context-reliability.md` sections 5.1 to 5.3 (60 min)
- [ ] Practical Task 5.1: build a "case facts" extractor (30 min)

### Friday — Domain 5 Part B
**Time:** 1.5 hrs
- [ ] Read `07-deep-dive-d5-context-reliability.md` sections 5.4 to 5.6 (50 min)
- [ ] Practical Task 5.6: structured-finding research workflow (40 min)

### Saturday — Lab 2
**Time:** 3 hrs
- [ ] Build Lab 2 — Multi-Agent Research System (2 hrs)
- [ ] Domain 4 + 5 anti-patterns in `08-anti-patterns-reference.md` (30 min)
- [ ] Domain 4 + 5 practice questions Sections 4 and 5 (Q4.1 - Q5.6) (30 min)

### Sunday — Mid-Course Mock + Buffer
**Time:** 3 hrs
- [ ] Take the FULL `11-practice-questions.md` test in one sitting, timed (90 min)
- [ ] Score it (15 min)
- [ ] For every miss, identify the file/section and the anti-pattern (45 min)
- [ ] Mark your weakest 2 sections; allocate Week 4 time accordingly (30 min)

**Week 3 milestone:** You've completed all five deep dives and three of the four labs. You have a quantitative measure of your weakest domains.

---

## Week 4 — Integration, Final Labs, Mock Exam, Targeted Review

### Monday — Weakest Domain Targeted Review
**Time:** 1.5 hrs
- [ ] Identify the lowest-scoring domain from Sunday's mock (5 min)
- [ ] Re-read that domain's deep dive (60 min)
- [ ] Re-read that domain's anti-patterns section (25 min)

### Tuesday — Second-Weakest Domain Targeted Review
**Time:** 1.5 hrs
- [ ] Same approach for second-weakest (90 min)

### Wednesday — Cross-Domain Integration
**Time:** 1.5 hrs
- [ ] Re-read `09-decision-frameworks.md` master summary (15 min)
- [ ] Final 5 cross-domain practice questions (QF.1 - QF.5) (30 min)
- [ ] Pretend each scenario from the exam guide (Customer Support, Code Generation, Multi-Agent Research, Developer Productivity, CI/CD, Structured Extraction) and **for each** state in your own words: which domains apply, which patterns matter, which anti-patterns are tempting (45 min)

### Thursday — Lab Capstone
**Time:** 1.5 hrs
- [ ] Lab 1 reflection questions (if not done) (30 min)
- [ ] Lab 4 reflection questions (if not done) (30 min)
- [ ] Capstone reflection at the end of `10-hands-on-labs.md` — write out answers (30 min)

### Friday — Cheat Sheet Review + Memory Drill
**Time:** 1.5 hrs
- [ ] Open `cheatsheet.html` and quiz yourself on every section without looking (60 min)
- [ ] Re-read the "Top 10 must-know" in `01-quick-start.md` (10 min)
- [ ] Re-read the master summary in `09-decision-frameworks.md` (10 min)
- [ ] Final review of the anti-pattern catalog (`08-anti-patterns-reference.md`) (10 min)

### Saturday — Full Mock Exam + Triage
**Time:** 3 hrs
- [ ] Take the FULL `11-practice-questions.md` test again, timed (90 min)
- [ ] Score and compare to mid-course mock (15 min)
- [ ] For any STILL-missed question, write out the right answer in your own words and the trap pattern in the wrong options (75 min)

**Goal:** ≥85% on this final mock = ready for exam.

### Sunday — Light Day Before Exam
**Time:** 1 hr (no more)
- [ ] Re-read `01-quick-start.md` "10 things you must know cold" (10 min)
- [ ] Re-read `09-decision-frameworks.md` master summary (10 min)
- [ ] Re-read your "things I keep forgetting" notes (15 min)
- [ ] Sleep early. Trust your prep. ⚡

---

## Daily Habit (All 4 Weeks)

In addition to the scheduled tasks, do these every day (10 min total):

1. **5-minute spaced repetition.** Pick 5 anti-patterns at random from `08-anti-patterns-reference.md` and recite the diagnostic phrase + correct response.
2. **5-minute mental model.** Pick one mental model from `01-quick-start.md` and explain it out loud as if to a colleague.

---

## Adjustments by Performance

### If you're acing practice questions early (90%+ by end of Week 2):
- Skip the Sunday buffer days; use that time to **build your own practice questions** based on your work
- Re-do labs with intentional variations (e.g., remove the hook in Lab 1 and observe what happens)
- Try to break each pattern in adversarial ways

### If you're struggling (<70% on Week 3 mock):
- **Extend the timeline by a week.** You said this is fine. Don't rush.
- Slow down on the deep dives — read more carefully, take written notes
- Pair lab work with more drilling — re-do practice questions twice
- For the weakest domain, don't just re-read — **teach it back** to a friend or a rubber duck

### If your work schedule disrupts Week 2 or 3:
- Skip the deep dive's Practical Tasks (they're nice but not essential) and prioritize:
  1. Reading the deep dive
  2. Anti-patterns for that domain
  3. Practice questions for that domain
- The 4 labs are non-negotiable — at minimum, build Lab 1 and Lab 4

---

## Pre-Exam Checklist (Day Before)

- [ ] You can recite each of the 5 mental models without looking
- [ ] You can apply each of the 9 decision frameworks in <5 seconds
- [ ] You can identify each of the ~50 anti-patterns by their diagnostic phrase
- [ ] You scored ≥85% on the final mock
- [ ] You've built and tested Labs 1, 2, 3, AND 4 (or at minimum 1 and 4)
- [ ] You know the rename quirk: **Task tool → Agent tool** (use the exam's terminology)
- [ ] You know the domain weights: **D1 27 / D2 18 / D3 20 / D4 20 / D5 15**

---

## Emergency Recovery — The Last 48 Hours

If your plan falls apart and you have only 48 hours left:

**Day -2:**
1. `01-quick-start.md` — read fully (1 hr)
2. `08-anti-patterns-reference.md` — read fully (2 hrs)
3. `09-decision-frameworks.md` — memorize master summary (1 hr)
4. `11-practice-questions.md` — work through, focus on Sections 1, 3, 4 (highest weights) (3 hrs)

**Day -1:**
1. Re-read missed practice questions (2 hrs)
2. Re-read the deep dives for your weakest domain (2 hrs)
3. Re-take practice questions for that domain (1 hr)
4. Light review of the cheat sheet (1 hr)

**Day 0:**
- 30-minute reading of `01-quick-start.md` "10 things you must know cold"
- Take the exam.

This crash plan won't get you to 90% but should get you to ~750-800 if your background is solid (which it is).

---

Good luck. The plan works if you work the plan.
