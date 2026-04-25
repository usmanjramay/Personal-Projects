# AI Agent System Design

**Status:** Planning
**Last updated:** 2026-04-25

> A living reference for how to use AI agents more effectively — updated as we learn and experiment.

---

## Research

Key learnings from research so far:

1. **Exploration first.** Before any implementation, there needs to be an exploration run — surveying the problem space, existing solutions, and constraints before committing to a path.
2. **More conversation upfront.** There is likely meaningful overlap between the exploration/research phase and the user story phase. More back-and-forth with the user before acting reduces wasted work downstream.
3. **Adversarial reviews add the most value.** Multiple adversarial agents challenging the solution from different angles catches problems early:
   - One checks whether the requirements from the user story are actually being met.
   - One checks whether better solutions exist that were missed.
   - One checks what could go wrong with the proposed solution.
4. **Dual-source search.** Run the same search once through Gemini or Perplexity and once through Claude, then compare results. This catches blind spots and improves quality — this step is important and should not be skipped.

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

Before any research or work begins, sharpen the problem through structured questioning. The goal is to get clarity on what the task is actually about and what a good outcome looks like.

**Skill options to use for this step (try each once and see what works best):**
- `/first-principles` — Socratic breakdown; good for ambiguous or complex problems
- `/office-hours` — YC-style challenge mode; good when the idea needs stress-testing
- No "Grill Me" skill exists yet, but worth adding if the above don't fully serve this step
- GStack review skills (`/design-review`, `/devex-review`) — likely not the right fit here; they are designed for reviewing built things, not sharpening problem definitions

**Output of this step:** A clear user story with pain points, wish list, and any known constraints — written before any research begins.

---

### Step 2 — Research

With the user story defined, run a structured research phase to explore the problem space before committing to a solution.

**How to run this step:**
1. Run an exploration pass — survey existing tools, approaches, and prior art
2. Run the same search through Claude and through Gemini or Perplexity separately, then compare the results — note where they diverge
3. Bring findings back to the user for a short conversation before moving forward; there may be things worth discussing before locking in a direction

**Output of this step:** A summary of what exists, what looks promising, and what was considered but ruled out — plus any open questions for the user.

---

### Step 3 — Adversarial Review

Before finalizing a direction, run three separate adversarial agents to challenge the proposed solution:

| Agent | Question it answers |
|-------|---------------------|
| Requirements check | Are all the requirements from the user story actually being met by this solution? |
| Better solutions | Are there meaningfully better alternatives that were missed or dismissed too quickly? |
| Risk check | What could go wrong with this solution, and how serious are those risks? |

These can run in parallel. Bring the outputs back to the user before moving to implementation.

---

### Step 4 — Implementation Plan and Task Breakdown

Once the user story is finalized and the research direction is agreed upon, build a concrete implementation plan.

**Structure:**
- Break the work into discrete steps that can each be executed and tested independently
- Each step should be small enough to be handed to a sub-agent with clear inputs and expected outputs
- Follow the pattern that superpowers uses: each task verifiable before the next one starts

**Output of this step:** A numbered implementation plan with sub-tasks that are independently executable and testable.

---

### Step 5 — Execution

Execute the implementation plan step by step using sub-agents where appropriate.

- Each sub-task gets its own agent with clear scope
- After each sub-task completes, verify the result before moving to the next step (see Testing Layer 1 below)

---

### Step 6 — Testing

Testing happens in two layers:

**Layer 1 — Per-task verification (during execution)**
After each sub-task in the implementation plan, verify the result is correct before proceeding. This is the same pattern superpowers uses for sub-agent output verification — do not skip it.

**Layer 2 — Final end-to-end check (after all tasks complete)**
Once all tasks are done, run a full check of the finished system against the wish list from the User Story. For anything with a visual or interactive component, do a manual walkthrough using a clean Chrome profile with no personal accounts logged in — this ensures the test reflects what a real user would experience.

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-25 | File created | Documenting the AI agent workflow process based on initial research |
