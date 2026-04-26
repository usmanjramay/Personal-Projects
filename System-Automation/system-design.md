# AI Agent System Design

**Status:** Planning
**Last updated:** 2026-04-25

> A living reference for how to use AI agents more effectively — updated as we learn and experiment.

---

## Open Items

1. [ ] What is the best research skill available these days?
2. [ ] Decide which skill works best for the User Story step (first-principles, office-hours, or another) — run each once and compare
3. [ ] Test the dual-source search workflow (Claude vs. Gemini/Perplexity) on a real task and document what differences emerge
4. [ ] Explore setting up a clean Chrome profile (no personal accounts) for final visual testing
5. [ ] Define what "adversarial review" prompts look like in practice — write reusable templates for each type
6. [ ] Consider whether the adversarial review agents should run in parallel or sequentially
7. [ ] **VS Code extension ignores `~/.claude/settings.json` allowlists (confirmed bug)** — the extension has its own permission layer that always prompts, regardless of what's in the allowlist. Two options: (A) enable `claudeCode.allowDangerouslySkipPermissions: true` + `claudeCode.initialPermissionMode: bypassPermissions` in VS Code settings — blunt but works; or (B) use the `claude` CLI in the integrated terminal with `/ide` to reconnect the diff viewer — granular and fully respects the allowlist. Multiple open GitHub issues; no fix yet as of April 2026.

---

## System Processes

Capabilities that support the work across all projects — not tied to a specific phase or workflow.

### Quick Notes

For fast capture without wasting tokens: use a bash append command to write to `~/.claude/quick-notes.md` without reading the file first. This is the standard process documented in the global [`CLAUDE.md`](~/.claude/CLAUDE.md). Do not use the Read tool before appending, and do not use the Second Brain MCP for quick captures.

### Self-Improvement

The `/review-n-learn` skill reviews any session to extract improvement ideas and update the system. Run it before closing or compacting a chat where something felt worth capturing — it works from in-session context at zero extra cost.

Skill file: [`~/.claude/skills/review-n-learn/SKILL.md`](~/.claude/skills/review-n-learn/SKILL.md)

### Automation

All n8n automation workflows are managed through the `n8n-automation-skill`, which is global and has its own self-improvement loop. More details on how Claude Code runs across different environments (VS Code, web, CLI) and what's available in each are covered in [`claude-environments.md`](claude-environments.md).

### Git Workflow

- **`git pull`** on session start → configured as a SessionStart hook in `settings.json`. Runs automatically — hooks are more reliable than a CLAUDE.md rule.
- **`git push`** after approved changes → rule in global `CLAUDE.md`. Done at end of task, not automatically, to preserve a review checkpoint. Auto-push via hook was rejected: risk of pushing untested changes.

### Context Management

How Claude Code context files are structured — what lives globally vs. per-project, how documents are organized, and the principles behind those decisions. Covered in [`context-management.md`](context-management.md).

---

## New Feature Process

### Step 1 — User Story

Get clarity on what the task is actually about and what a good outcome looks like. Covered by the `/generate-story` skill. Output: a User Story document.

---

### Step 2 — Research

Survey the problem space, research available tools and approaches, then run adversarial reviews to stress-test the options before agreeing a direction. Covered by the `/solution-research` skill. Output: a Design document.

---

### Step 3 — Implementation Plan

Break the agreed direction into discrete, independently testable tasks. Currently using the `superpowers:brainstorming` skill. Output: an Implementation Plan document.

---

### Step 4 — Execution

Execute the implementation plan step by step, verifying each task before moving to the next. Currently using the `superpowers:executing-plans` skill.

---

### Step 5 — Testing

Full check of the finished system against the wish list from the User Story. *Still in progress — no standardized skill yet.*

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-26 | Added System Processes section; renamed process to New Feature Process; removed Step 6 (Wrapup) | Wrapup is on-demand, not a fixed step; Quick Notes and Review and Learn are system-wide capabilities, not feature-specific |
| 2026-04-26 | Change Log file updated to specify Change-Log/ subfolder | Corrected location after user moved first change log to subfolder |
| 2026-04-26 | Added /review-n-learn skill to system | Need a standard way to extract learnings from sessions and convert to system improvements |
| 2026-04-26 | Added Change Log file as document type 6 | First change log file created; needed a category in the document structure table |
| 2026-04-25 | File created | Documenting the AI agent workflow process based on initial research |
