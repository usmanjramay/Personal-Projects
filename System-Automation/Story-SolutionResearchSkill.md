# Solution Research Skill — Improvement

**Status:** Planning
**Last updated:** 2026-04-26
Document Type: Story
Generated on: 2026-04-26
Time invested: ~45 minutes

> Improve the solution-research skill from a first-draft baseline to a rigorous, standardized process that eliminates the two core failure modes: missing a better solution and hitting surprise problems during implementation.

---

## User Story

### What We Are Building

A revised version of the `/solution-research` skill with five targeted improvements to the existing process. The core phase structure (surface questions → search → adversarial review → agree direction → produce Design document) is sound and stays intact. The changes close five specific gaps: capturing existing technical context before research begins, replacing the hard-coded optimization priority with three balanced evaluation factors and a strong MVP-first default, making Phase 3 genuinely adversarial by spinning off separate sub-agents explicitly instructed to find flaws, adding a continuous feedback loop that can redirect the process at any point when direction shifts, and expanding the Design document output to include acceptance criteria and a risk register for the agreed solution. The skill is used by Usman and his two junior developers — it must be self-contained and consistent enough to produce the same quality of output regardless of who runs it.

### Minimum Acceptance Criteria

The following must all be true for this improvement to be considered successful:

1. No manual Google search by anyone on the team surfaces a flaw or a better option that the skill missed — if that happens, the skill has failed
2. The risks and problems surfaced for the chosen solution are comprehensive enough that no surprise issues emerge during implementation — everything foreseeable was caught, accounting for the project's specific infrastructure and context
3. All proposed options and implementation details are current and not outdated at the time the skill is run

### Good to Have

1. At no point in the foreseeable future does a chosen solution feel like a mistake or a missed opportunity

---

## Background

### Where We Started

Usman described a recurring pattern: pre-research before building is ad hoc — varying between team members and over time — causing expensive rework when better solutions are discovered post-build, or when unforeseen implementation issues arise. The solution-research skill had just been created as a structured first attempt to fix this, based on intuition rather than a tested process. The goal of this session was to stress-test the skill before putting it into real use, rather than discovering its gaps through failed runs.

### Assumptions We Challenged

| Assumption | Status | What We Found |
|---|---|---|
| The existing technical environment is captured before research starts | ❌ Invalidated | No step existed for this — options could be incompatible with the existing system from the start |
| The hard-coded optimization order (ease/speed first, cost second) is always correct | ❌ Invalidated | Three balanced factors — quality, effort, cost — are always a trade-off; no fixed order applies universally |
| Phase 3 is genuinely adversarial | ❌ Invalidated | Three analytical sections written in one session by the same AI carries the same blind spots as the research phase |
| The skill has a feedback loop if direction changes mid-process | ❌ Invalidated | The skill only moves forward; no path back was defined |
| "Test it first" is strong enough as an option to consider | ❌ Invalidated | Needs to be a default bias — prefer existing paid services to validate quickly, build custom only once proven |
| User story as primary input is sufficient context for Phase 1 | ✅ Held up | The story provides enough context; Phase 1 questions can be surfaced from it |
| Dual-source search (Claude + Gemini/Perplexity) adds value | ✅ Held up | Keep manual for now; automate only once value is confirmed through real runs |
| The Design document is the right end artifact | ✅ Held up | It feeds into implementation planning (superpowers skill); handoff is covered |
| Standardization is itself a core goal | ✅ Held up | Consistency across users and over time is as important as finding the right answer in any single run |

### What We Narrowed Down To — and Why

Five targeted changes to the existing skill rather than a rewrite. The phase structure is valid — the gaps are specific and addressable without disrupting the overall flow. This approach also preserves the skill as a living document: if the first runs reveal further gaps, those can be patched without re-architecting.

The five changes:

1. **Existing context step** — Before Phase 1, read any provided context files. If insufficient, ask: *"What code or infrastructure exists that this change will impact or be impacted by?"* with a brief explanation of why it matters for the research. Only ask if not already answered by provided context.

2. **Balanced evaluation factors** — Remove the hard-coded priority order. State three equal factors upfront: quality (how well it solves the problem), effort (ease and speed of implementation and maintenance), cost. Default to an MVP-first bias — prefer existing paid services to validate quickly, then build custom only if validated. Inform the user and ask if they have a specific preference for this case.

3. **Genuinely adversarial Phase 3** — Replace the three analytical sections with three actual separate sub-agent calls. Each agent receives the options and is explicitly instructed: find flaws, do not summarize or validate. One agent per angle: (a) requirements gaps, (b) missing or better alternatives, (c) risks and inversion. The main agent scores the options afterward, informed by all three adversarial reports — sub-agents do not score.

4. **Continuous feedback loop** — At any point during the process, if research or conversation suggests the direction has shifted significantly, the skill stops, assesses how far back the process needs to re-run, and proposes this to the user before proceeding. Not limited to phase boundaries.

5. **Expanded Design document** — Add to the end of the output: key risks for the agreed plan, what can go wrong, acceptance criteria (what done looks like), and specific things to verify that the solution is working correctly.

### What We Decided Not to Build

- **Automated Gemini/Perplexity search**: The dual-source step stays manual. Run it a few times to validate whether it adds meaningful value before spending time automating it — especially since automation would require setup on each team member's machine.
- **Sub-agent scoring in Phase 3**: Sub-agents stay purely adversarial. Having them score as well would split their focus. The main agent scores, with the benefit of all adversarial input.
- **Full skill rewrite**: The existing phase structure is sound. Only the five gaps above need closing.

---

## Current Status

Story defined. Skill not yet updated.

---

## Open Tasks and Ideas

- [ ] Rewrite `/solution-research/SKILL.md` with the five improvements defined above
- [ ] Run the updated skill on a real problem and note what works and what doesn't
- Idea: Once the skill has been run a few times, build a skill change log / learning tracker in this project to track assumptions, issues, and changes over time across all skills

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-26 | Story created | First-principles session to stress-test and improve the skill before real use |
