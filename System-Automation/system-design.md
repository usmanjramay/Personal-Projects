# System Design

**Document type:** System
**Last updated:** 2026-04-27

> A living reference for how to use AI agents more effectively — updated as we learn and experiment.

---

## Open Items

1. [ ] Test the dual-source search workflow (Claude vs. Gemini/Perplexity) on a real task and document what differences emerge
2. [ ] Explore setting up a clean Chrome profile (no personal accounts) for final visual testing
3. [ ] **VS Code extension ignores `~/.claude/settings.json` allowlists (confirmed bug)** — the extension has its own permission layer that always prompts, regardless of what's in the allowlist. Two options: (A) enable `claudeCode.allowDangerouslySkipPermissions: true` + `claudeCode.initialPermissionMode: bypassPermissions` in VS Code settings — blunt but works; or (B) use the `claude` CLI in the integrated terminal with `/ide` to reconnect the diff viewer — granular and fully respects the allowlist. Multiple open GitHub issues; no fix yet as of April 2026.

---

## System Processes

Capabilities that support the work across all projects — not tied to a specific phase or workflow. More details on the infrastructure can be found in: [`claude-environments.md`](claude-environments.md), [`remote-vps-setup.md`](remote-vps-setup.md), [`second-brain-setup.md`](second-brain-setup.md), [`executive-assistant-agent.md`](executive-assistant-agent.md), [`google-accounts.md`](google-accounts.md).

### Quick Notes

For fast capture without wasting tokens: use a bash append command to write to `~/.claude/quick-notes.md` without reading the file first. This is the standard process documented in the global [`CLAUDE.md`](~/.claude/CLAUDE.md). Do not use the Read tool before appending, and do not use the Second Brain MCP for quick captures.

### Context Management

How Claude Code context files are structured — what lives globally vs. per-project, how documents are organized, and the principles behind those decisions. Covered in [`context-management.md`](context-management.md).

### Automation

All n8n automation workflows are managed through the `n8n-automation-skill`, which is global and has its own self-improvement loop. More details on how Claude Code runs across different environments (VS Code, web, CLI) and what's available in each are covered in [`claude-environments.md`](claude-environments.md).

### Self-Improvement

The `/review-n-learn` skill reviews any session to extract improvement ideas and update the system. Run it before closing or compacting a chat where something felt worth capturing — it works from in-session context at zero extra cost.

Skill file: [`~/.claude/skills/review-n-learn/SKILL.md`](~/.claude/skills/review-n-learn/SKILL.md)

The `/session-analytics` skill analyzes past Claude Code sessions to surface patterns — what kinds of tasks are taking the most tokens, where time is being spent, and what that reveals about how to work more efficiently. Run it periodically to get a data-driven view of usage across sessions.

Skill file: [`~/.claude/skills/session-analytics/SKILL.md`](~/.claude/skills/session-analytics/SKILL.md)

### Git Workflow

- **`git pull`** on session start → configured as a SessionStart hook in `settings.json`. Runs automatically — hooks are more reliable than a CLAUDE.md rule.
- **`git push`** after approved changes → rule in global `CLAUDE.md`. Done at end of task, not automatically, to preserve a review checkpoint. Auto-push via hook was rejected: risk of pushing untested changes.

### Token Optimization

Aimed at improving efficiency and making sure we're not wasting tokens — covering how to structure prompts, use context wisely, and avoid common sources of token bloat. Covered in [`token-optimization.md`](token-optimization.md).

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
| 2026-04-27 | Renamed Log-Files/ folder system-wide (was Change-Log/) | Folder name was too narrow — Log-Files/ can hold any kind of log, not just change logs |
| 2026-04-26 | Added System Processes section; renamed process to New Feature Process; removed Step 6 (Wrapup) | Wrapup is on-demand, not a fixed step; Quick Notes and Review and Learn are system-wide capabilities, not feature-specific |
| 2026-04-25 | File created | Documenting the AI agent workflow process based on initial research |
