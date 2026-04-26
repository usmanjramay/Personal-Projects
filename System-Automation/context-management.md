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

Each feature gets its own subfolder. The user story, research, design, implementation plan, and feature documents for that feature all live in that subfolder. Only the **system document** and the **feature document** are live documents — they evolve over time. The others are created once and should not generally change after that.

| # | Document | Created by | Purpose |
|---|----------|------------|---------|
| 1 | **System document** | Usman (manually) | High-level reference for how a system works — architecture, design decisions, and ongoing learnings. |
| 2 | **User story document** | `/generate-story` skill | Captures the problem, pain points, wish list, and constraints agreed on with the user before any work begins. |
| 3 | **Research document** | `/solution-research` skill | Surveys the problem space and available options — tools, approaches, alternatives, and adversarial review outputs gathered before committing to a direction. |
| 4 | **Design document** | `/superpowers:brainstorm` skill | Records the agreed direction coming out of research and brainstorming — what to build, key decisions, and why. |
| 5 | **Implementation plan document** | `/superpowers:write-plan` skill | The numbered task breakdown used to execute the work — each task independently executable and testable. |
| 6 | **Feature document** | `/superpowers:requesting-code-review` skill | Triggers a review checklist that compares the implementation against the plan, highlighting discrepancies to document or fix. Post-ship summary of what was built, what was learned, and open questions for future work. |
| 7 | **Change Log file** | As needed | Records changes made to skills, tools, or processes — what changed, why, and when. Saved in `Change-Log/` subfolder. Named: `YYYY-MM-DD-[Thing-Changed]-[brief-description].md`. |

---

## Writing Markdown Files

### Formatting Rules

- Single H1 (`#`) — document title only
- H2 (`##`) for main sections, H3 (`###`) for subsections — no skipping levels
- Tables for structured data (more accurate for LLMs than bullet prose)
- Code blocks with language tags (` ```bash `, ` ```json `)
- No raw HTML — pure Markdown parses more reliably
- Consistent heading names — don't rename sections across edits
- **Headers are especially important** — Claude locates a section by grepping for its heading, then reads just those lines. Clear, consistent headers make targeted reads efficient.

### Key Principles

**All files:**

- Open with a brief context summary — one or two sentences on what this file is and why it exists.
- Follow with two metadata lines: **Document type** and **Last updated**.

**Live files (system documents and feature documents) only:**

- Include an **Open Items** section with a numbered list of outstanding tasks, questions, or decisions.
- Include a **Critical Assumptions** section only when the system or feature depends on something external that could change and would require updating this file if it did.
- Include a **Section Map** near the top listing each section and its approximate line number, so Claude can navigate without reading the full file first.
- Include a **Change Log** at the end — major changes only. Small corrections belong in git history. When a Change Log file in `Change-Log/` is relevant to this system or feature, reference it from this section.

### How Claude Reads Files

**Claude can read specific line ranges.** The Read tool accepts `offset` (start line) and `limit` (number of lines), so only the relevant slice of a file needs to be loaded. The standard pattern is:

1. **Grep to locate** — run `grep -n "<heading or keyword>" <file>` to find the exact line number.
2. **Read the slice** — call Read with `offset` set to just before that line.
3. **Edit precisely** — make the targeted change without loading the rest of the file.

This is why consistent headers matter: they are what grep matches on. Live documents (system and feature documents) should include a **Section Map** near the top so Claude can navigate directly to any section without a grep step.
