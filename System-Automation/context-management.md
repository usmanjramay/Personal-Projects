# Context Management

**Document type:** System
**Last updated:** 2026-04-27

> Reference file for how context is structured and maintained across Claude Code sessions in this project. The goal of context management is for Claude to always have the right context for every conversation — without Usman having to do anything manually.

---

## Section Map

| Section                | Line |
|------------------------|------|
| Open Items             | ~22  |
| Structure              | ~28  |
| Document Types         | ~47  |
| Writing Markdown Files | ~65  |
| Change Log             | ~197 |

---

## Open Items

_No open items._

---

## Structure

### File Organization

- **Global (`~/.claude/`):** Applies to every Claude Code session — contains `CLAUDE.md` (universal rules), `settings.json` (hooks and permissions), skills, and slash commands.
- **Per-project:** Each project has a `CLAUDE.md` plus one main file that describes the project's structure, tracks open items, and summarizes current status. This main file is always the first to read when starting work on a project, and it contains references to all other files in that project.

### How Claude Reads Files

**Claude can read specific line ranges.** The Read tool accepts `offset` (start line) and `limit` (number of lines), so only the relevant slice of a file needs to be loaded. The standard pattern is:

1. **Grep to locate** — run `grep -n "<heading or keyword>" <file>` to find the exact line number.
2. **Read the slice** — call Read with `offset` set to just before that line.
3. **Edit precisely** — make the targeted change without loading the rest of the file.

This is why consistent headers matter: they are what grep matches on. Live documents (system and feature documents) should include a **Section Map** near the top so Claude can navigate directly to any section without a grep step.

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
| 7 | **Log file** | As needed | Records notable events related to a skill, tool, or process — changes, issues, or other events worth tracking over time. Can be a Change Log, Issue Log, or any other log type. Saved in `Log-Files/` subfolder. Named: `YYYY-MM-DD-[Thing-Changed]-[brief-description].md`. |

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

- Open with two metadata lines: **Document type** and **Last updated**.
- Follow with a blockquote summary (`>`) — one or two sentences on what this file is and why it exists.

**Live files (system documents and feature documents) only:**

- Include an **Open Items** section with a numbered list of outstanding tasks, questions, or decisions.
- Include a **Critical Assumptions** section only when the system or feature depends on something external that could change and would require updating this file if it did.
- Include a **Section Map** near the top listing each section and its approximate line number, so Claude can navigate without reading the full file first.
- Include a **Change Log** at the end — major changes only. Small corrections belong in git history. If a change is significant enough to warrant more detail than a single row, create a log file in `Log-Files/` named `YYYY-MM-DD-[Thing-Changed]-[brief-description].md` and reference it from this table.

### Cross-Referencing Related Files

Use `@path/to/file` syntax inline, near the content it relates to — not in a separate "See also" section at the bottom. Claude does not auto-discover relationships; you must reference files explicitly. Only link files Claude would genuinely need for the current task.

```markdown
For VPS access details, see @remote_vps.md.
```

---

### When Editing Existing Files

Whenever editing any markdown file in this project, check and update it to match the templates below — without losing any existing information. Specifically:

1. **Header (all files):** Ensure the file opens with an H1 title, a context summary, and the `Document type` and `Last updated` metadata lines. If the document type is unclear, ask before proceeding.
2. **Living documents (System and Feature types):** Also ensure a **Section Map**, **Open Items**, and **Change Log** section are present and follow Template 2. If any are missing, add them.
3. **Critical Assumptions:** This section is optional. Add it if the file describes something that depends on an external factor that could change. If it's already present, make sure it follows the template format. If it's unclear whether it applies, ask.

Do not restructure or rename existing sections — just bring the header and required sections into alignment with the template.

---

### Templates

**Template 1 — All files (mandatory header)**

Every file, regardless of type, must open with this header:

```markdown
# [File / Feature / Project Name]

**Document type:** [System | User Story | Research | Design | Implementation Plan | Feature | Log File]
**Last updated:** YYYY-MM-DD

> One or two sentences on what this file is and why it exists.
```

---

**Template 2 — Living documents (system and feature documents)**

In addition to the mandatory header above, living documents include the following sections:

```markdown
# [File / Feature / Project Name]

**Document type:** [System | Feature]
**Last updated:** YYYY-MM-DD

> One or two sentences on what this file is and why it exists.

---

## Section Map

| Section               | Line |
|-----------------------|------|
| Open Items            | ~00  |
| Critical Assumptions  | ~00  |
| How It Works          | ~00  |
| Change Log            | ~00  |

---

## Open Items

Valid statuses: **[Idea]**, **[In-Progress]**, **[Limitation]**, **[Blocked]**

1. **[In-Progress]** Migrate authentication flow to the new provider
   - a. Token exchange step is complete
   - b. Redirect URL still needs updating in the provider dashboard
   - c. Test coverage for edge cases not yet written

2. **[Idea]** Add rate limiting to the webhook endpoint
   - a. Evaluate whether this belongs in the gateway or in the handler
   - b. No timeline yet — revisit after initial launch

---

## Critical Assumptions

_Only include if this system or feature depends on something external that could change and would require updating this file._

| Assumption | Current value | If this changes... |
|------------|--------------|-------------------|
| [Assumption] | [Value] | [What to update] |

---

## How It Works

_No fixed format. Include whatever information best explains this system or feature: architecture tables, numbered flows, component descriptions, diagrams, etc. Do not omit relevant technical details._

---

## Change Log

Most recent first. Major changes only — small corrections belong in git history. If a change is significant enough to warrant more detail, create a file for it in `Log-Files/` and reference it from this table.

| Date       | Change         | Why      |
|------------|----------------|----------|
| YYYY-MM-DD | [What changed] | [Reason] |
```

---

## Change Log

| Date       | Change                                                              | Why                                                                 |
|------------|---------------------------------------------------------------------|---------------------------------------------------------------------|
| 2026-04-27 | Renamed Change-Log/ folder to Log-Files/; updated document type 7 from "Change Log file" to "Log file"; updated all references to reflect that log files can be different types (change logs, issue logs, etc.) | Folder name was too narrow — Log-Files/ can hold any kind of log, not just change logs |
| 2026-04-27 | Merged How Claude Reads Files into Structure; moved When Editing Existing Files before Templates | Improve logical flow of the document |
| 2026-04-27 | Added templates, cross-referencing section, When Editing Existing Files rule, and applied new template to this file | Consolidate all markdown writing standards in one place             |
