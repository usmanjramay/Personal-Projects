# AI Agent System Design

**Status:** Planning
**Last updated:** 2026-04-25

> A living reference for how to use AI agents more effectively — updated as we learn and experiment.

---

## To Be Researched

- What is the best research skill available these days?

---

## Open Tasks and Ideas

- [ ] Decide which skill works best for the User Story step (first-principles, office-hours, or another) — run each once and compare
- [ ] Test the dual-source search workflow (Claude vs. Gemini/Perplexity) on a real task and document what differences emerge
- [ ] Explore setting up a clean Chrome profile (no personal accounts) for final visual testing
- [ ] Define what "adversarial review" prompts look like in practice — write reusable templates for each type
- Idea: consider whether the adversarial review agents should run in parallel or sequentially

---

## Process

### Step 1 — User Story

With exploration complete, sharpen the problem through structured questioning. The goal is to get clarity on what the task is actually about and what a good outcome looks like.

**Skill options to use for this step (try each once and see what works best):**
- `/first-principles` — Socratic breakdown; good for ambiguous or complex problems
- `/office-hours` — YC-style challenge mode; good when the idea needs stress-testing

**Output:** A clear user story with pain points, wish list, and any known constraints.

---

### Step 2 — Research

The full research phase before any implementation begins. This step has three parts that run in sequence.

#### 2a — Exploration Search

Survey the problem space before committing to a direction. The goal is to understand what already exists, what approaches are available, and what constraints matter.

- Run an exploration pass — survey existing tools, approaches, and prior art
- Run the same search through Claude and through Gemini or Perplexity separately, then compare the results — note where they diverge

**Output:** A summary of what exists, what looks promising, and what was considered but ruled out.

---

#### 2b — Adversarial Reviews

Before finalizing a direction, run three separate adversarial agents to challenge the proposed solution:

| Agent | Question it answers |
|-------|---------------------|
| Requirements check | Are all the requirements from the user story actually being met by this solution? |
| Better solutions | Are there meaningfully better alternatives that were missed or dismissed too quickly? |
| Risk check | What could go wrong with this solution, and how serious are those risks? |

These can run in parallel. Bring the outputs back to the user before moving forward.

---

#### 2c — Finalize

Bring all findings — exploration results, user story, and adversarial review outputs — back to the user for a short conversation. Agree on a direction before moving to implementation.

**Output:** A confirmed direction with open questions resolved.

---

### Step 3 — Implementation Plan

Once the direction is agreed upon, build a concrete implementation plan with task breakdown.

**Structure:**
- Break the work into discrete steps that can each be executed and tested independently
- Each step should be small enough to be handed to a sub-agent with clear inputs and expected outputs
- Follow the pattern that superpowers uses: each task verifiable before the next one starts. After each sub-task in the implementation plan, verify the result is correct before proceeding. Do not skip this.

**Output:** A numbered implementation plan with sub-tasks that are independently executable and testable.

---

### Step 4 — Execution

Execute the implementation plan step by step using sub-agents where appropriate.

- Each sub-task gets its own agent with clear scope
- After each sub-task completes, verify the result before moving to the next step (see Step 5)

---

### Step 5 — Testing
Once all tasks are done, run a full check of the finished system against the wish list from the User Story. For anything with a visual or interactive component, do a manual walkthrough using a clean Chrome profile with no personal accounts logged in — this ensures the test reflects what a real user would experience.

---

### Step 6 — Wrapup

After the work is done, document what was learned so future work benefits from it.

- What worked well and should be repeated?
- What was harder than expected, and why?
- What would you do differently next time?
- Any new open questions that surfaced?

**Output:** An update to this file (or the relevant project file) with new learnings added.

---

## Document Structure

Each project or feature produces a set of documents that correspond to the process steps. Five document types are used:

| # | Document | Created in | Purpose |
|---|----------|------------|---------|
| 1 | **System document** | — | High-level reference for how a system works — architecture, design decisions, and ongoing learnings. This file is an example. |
| 2 | **User story document** | Step 1 | Captures the problem, pain points, wish list, and constraints agreed on with the user before any work begins. |
| 3 | **Design document** | Step 2 | Records what was found during research — options explored, alternatives considered, adversarial review outputs, and the agreed direction. |
| 4 | **Implementation plan document** | Step 3 | The numbered task breakdown used to execute the work — each task independently executable and testable. |
| 5 | **Feature document** | Step 6 | Post-ship summary of what was built, what was learned, and any open questions for future work. |
| 6 | **Change Log file** | As needed | Records changes made to skills, tools, or processes — what changed, why, and when. Named: `YYYY-MM-DD-[Thing-Changed]-[brief-description].md`. |

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-26 | Added Change Log file as document type 6 | First change log file created; needed a category in the document structure table |
| 2026-04-25 | File created | Documenting the AI agent workflow process based on initial research |
