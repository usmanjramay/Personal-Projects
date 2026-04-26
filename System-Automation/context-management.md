# Context Management

## Why This File Exists

The goal of context management is for Claude to always have the right context for every conversation — without Usman having to do anything manually. This file tracks how that system is set up so it can be improved over time.

---

## Structure

- **Global (`~/.claude/`):** Applies to every Claude Code session — contains `CLAUDE.md` (universal rules), `settings.json` (hooks and permissions), skills, and slash commands.
- **Per-project:** Each project has a `CLAUDE.md` plus one main file that describes the project's structure, tracks open items, and summarizes current status. This main file is always the first to read when starting work on a project, and it contains references to all other files in that project.

---

## Document Types

Each project or feature produces a set of documents that correspond to the New Feature Process steps. See [system-design.md](system-design.md) for the process.

| # | Document | Created in | Purpose |
|---|----------|------------|---------|
| 1 | **System document** | — | High-level reference for how a system works — architecture, design decisions, and ongoing learnings. |
| 2 | **User story document** | Step 1 | Captures the problem, pain points, wish list, and constraints agreed on with the user before any work begins. |
| 3 | **Design document** | Step 2 | Records what was found during research — options explored, alternatives considered, adversarial review outputs, and the agreed direction. |
| 4 | **Implementation plan document** | Step 3 | The numbered task breakdown used to execute the work — each task independently executable and testable. |
| 5 | **Feature document** | Step 5 | Post-ship summary of what was built, what was learned, and any open questions for future work. |
| 6 | **Change Log file** | As needed | Records changes made to skills, tools, or processes — what changed, why, and when. Saved in `Change-Log/` subfolder. Named: `YYYY-MM-DD-[Thing-Changed]-[brief-description].md`. |

---

## Writing Markdown Files

### Key Principles

- **Status at the top, always.** Anyone opening this file should know the current state in 5 seconds.
- **Living document, not a snapshot.** Update in place. Git history covers the archive.
- **Include all relevant details.** Do not miss any technical details that may leave a gap in understanding.
- **One file per system.** Don't split unless the file genuinely outgrows 2000 lines.
- **Change log = major decisions only.** Small edits belong in git history, not here.
- **Wish List = success criteria.** Write it as outcomes that will be true when done, not as tasks.
- **Critical Assumptions are optional.** Only add them when a change in the assumption would require updating this doc or the system.

### Formatting Rules

- Single H1 (`#`) — document title only
- H2 (`##`) for main sections, H3 (`###`) for subsections — no skipping levels
- Tables for structured data (more accurate for LLMs than bullet prose)
- Code blocks with language tags (` ```bash `, ` ```json `)
- No raw HTML — pure Markdown parses more reliably
- Consistent heading names — don't rename sections across edits

### How Claude Reads Files

**Claude reads reactively, not predictively.** It cannot inspect file size before reading. It calls the Read tool with default parameters, receives the content, and then decides what to do next.

- **Under 2000 lines** → Claude reads the entire file in one call. A Section Map provides zero reading efficiency benefit.
- **Over 2000 lines** → Claude reads lines 1–2000, sees truncation, and may request more using `offset`. A Section Map at the top IS useful here.

**Section Map guidance by file length:**

| File length | Section Map | Action |
|-------------|-------------|--------|
| Under 300 lines | Not needed | No special action |
| 300–2000 lines | Recommended | Helps Claude navigate conceptually |
| Over 2000 lines | Required | Enables efficient partial reads |
| Over 500 lines | — | Consider splitting into multiple files |

### Critical Assumptions (when to include in a doc)

Only add a Critical Assumptions section to a doc when:
- The doc or the system it describes depends on something external that could change
- A change in that assumption would require someone to update this file or the system

Include "Last reviewed: YYYY-MM-DD" so it's clear when the assumptions were last verified.

**Assumptions for `markdown-file-structure.md` specifically:**

_Last reviewed: 2026-04-19_

| Assumption | Value | If this changes... |
|------------|-------|--------------------|
| Claude Code Read tool default line limit | 2000 lines | Re-evaluate Section Map threshold |
| `@filename` syntax followed in regular .md files | Yes, on demand | Re-evaluate cross-referencing approach |
| Claude Code version | April 2026 (claude-sonnet-4-6) | Re-verify reading mechanics |
| File format | Plain Markdown (.md) | Different formats may parse differently |
