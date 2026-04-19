# Markdown File Structure — Reference & Template

_Use this when creating any new living reference document._
**Last updated:** 2026-04-19

---

## Critical Assumptions

_Last reviewed: 2026-04-19_

| Assumption | Value | If this changes... |
|------------|-------|--------------------|
| Claude Code Read tool default line limit | 2000 lines | Re-evaluate Section Map threshold |
| `@filename` syntax followed in regular .md files | Yes, on demand | Re-evaluate cross-referencing approach |
| Claude Code version | April 2026 (claude-sonnet-4-6) | Re-verify reading mechanics |
| File format | Plain Markdown (.md) | Different formats may parse differently |

---

# Part 1: General Guidance

_These rules apply to every markdown file created across all projects._

---

## Key Principles

- **Status at the top, always.** Anyone opening this file should know the current state in 5 seconds.
- **Living document, not a snapshot.** Update in place. Git history covers the archive.
- **Include all relevant details** Do not miss any technical details, that may leave a gap in understanding.
- **One file per system.** Don't split unless the file genuinely outgrows 2000 lines.
- **Change log = major decisions only.** Small edits belong in git history, not here.
- **Wish List = success criteria.** Write it as outcomes that will be true when done, not as tasks.
- **Critical Assumptions are optional.** Only add them when a change in the assumption would require updating this doc or the system.

---

## Formatting Rules

- Single H1 (`#`) — document title only
- H2 (`##`) for main sections, H3 (`###`) for subsections — no skipping levels
- Tables for structured data (more accurate for LLMs than bullet prose)
- Code blocks with language tags (` ```bash `, ` ```json `)
- No raw HTML — pure Markdown parses more reliably
- Consistent heading names — don't rename sections across edits

---

## Cross-Referencing Related Files

Use `@path/to/file` syntax inline, near the content it relates to. Claude follows these on demand.

```markdown
For VPS access details, see @remote_vps.md.
```

- Place references inline — not in a separate "See also" section at the bottom
- Claude does NOT auto-discover relationships — you must reference files explicitly
- Don't over-reference: only link files Claude would genuinely need for the current task

---

## How Claude Reads Files

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

---

## Critical Assumptions (when to include)

Only add a Critical Assumptions section when:
- The doc or the system it describes depends on something external that could change
- A change in that assumption would require someone to update this file or the system

Include "Last reviewed: YYYY-MM-DD" so it's clear when the assumptions were last verified.

---

# Part 2: Standard Template for Features and Specs

_Use this structure whenever writing a spec, building a new feature, or documenting a new system or tool. It is not required for general reference files, setup docs, or one-off notes._

---

## Status Values

Use exactly one. Definitions:

| Status | Meaning |
|--------|---------|
| **Planning** | Idea is being defined; no work has started |
| **Execution** | Active work in progress |
| **Testing** | Build is done but open tasks remain; validating against the wish list |
| **Complete** | All wish list items confirmed; no remaining open tasks |

> Note: remain in **Testing** as long as any task on the open list is unresolved. Only move to **Complete** when the wish list is fully verified.

---

## Mandatory vs. Flexible Sections

| Section | Required? | Notes |
|---------|-----------|-------|
| Status header | **Mandatory** | Use predefined status values above |
| One-line summary | **Mandatory** | One sentence describing current state |
| User Story | **Mandatory** | Pain Points + Wish List always; Potential Risks optional |
| Current Status | **Mandatory** | Specific — not "mostly done" |
| Open Tasks and Ideas | **Mandatory** | Even if empty |
| Change Log | **Mandatory** | Major decisions only |
| Critical Assumptions | **Optional** | Only if a real external dependency exists |
| How It Works | **Flexible** | Include when the system needs explanation; format is open |
| Section Map | **Conditional** | Only if file exceeds 300 lines |

---

## User Story — Detailed Guidance

The User Story section defines why this feature or system exists. It should be written before any work begins and updated as understanding evolves. It has three parts:

### Pain Points _(mandatory, numbered)_

What specific problems or friction exist today? Write from the perspective of the person experiencing them. Number them so they can be referenced later.

Good: *"1. I have no way to capture a thought quickly when I'm on the move without opening an app."*
Bad: *"The current system is limited."*

Each pain point should describe:
- Who experiences it
- When or in what situation
- What the cost or consequence is

### Wish List _(mandatory, numbered)_

What would the user ideally want? Write each item as an outcome — something that will be true when the work is done — not as a task to complete.

Good: *"1. I can send a WhatsApp message and receive a finished result with no further input."*
Bad: *"1. Build a WhatsApp integration."*

**This list is the success criteria.** During testing, go through each item and confirm it is true. When all items are confirmed, the status moves to Complete.

### Potential Risks _(optional, numbered)_

Explicit constraints, non-goals, or risks worth flagging upfront. Include only if there's something specific to watch out for — not a generic list of things that could go wrong. Number them so they can be referenced.

Good: *"1. The agent must not use Usman's personal accounts — only its own."*
Bad: *"1. Security could be a concern."*

---

## Full Template

```markdown
# [System / Feature / Project Name]

**Status:** [Planning | Execution | Testing | Complete]
**Last updated:** YYYY-MM-DD

> One sentence: what is this and what does it do right now.

---

## Critical Assumptions _(optional — only include if meaningful)_

_Last reviewed: YYYY-MM-DD_

| Assumption | Value | If this changes... |
|------------|-------|--------------------|
| [Assumption] | [Current value] | [What to update in this doc or system] |

---

## Section Map _(only include if file exceeds 300 lines)_

| Section | What it contains |
|---------|-----------------|
| User Story | Pain points, wish list, risks |
| How It Works | Architecture and flow |
| Current Status | What's working today |
| Open Tasks and Ideas | To-dos and future thinking |
| Change Log | Major decisions over time |

---

## User Story

**Pain Points**
1. [Specific problem — who, when, what cost]
2. [Next problem]

**Wish List**
1. [Outcome that will be true when done]
2. [Next outcome]

_This list is the success criteria. Confirm each item during testing before moving to Complete._

**Potential Risks** _(optional)_
1. [Specific constraint or non-goal]

---

## How It Works _(flexible — include when the system needs explanation)_

No fixed format. Use what best explains the system:
- Components table for multi-part systems
- Numbered flow for request/response pipelines
- Diagram description for branching logic

Keep it concise — but do not omit any relevant techincal details.

---

## Current Status

[What is actually working today. Specific — not "mostly done" but "X works, Y is broken, Z not started."]

---

## Open Tasks and Ideas

- [ ] [Task] — _context or deadline if relevant_
- Idea: [future possibility, not committed]

---

## Change Log

Most recent first. Major decisions only — not every small edit.

| Date | Change | Why |
|------|--------|-----|
| YYYY-MM-DD | What changed | Reason |
```
