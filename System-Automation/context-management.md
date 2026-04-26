# Context Management — How Claude Code Is Set Up

_Decisions made on 2026-04-18 during a dedicated session to optimize all context files._

## Why This File Exists

This is a reference for Usman — not for Claude. It captures what was decided about how Claude Code context files are organized and why, so you can remember the reasoning when revisiting these decisions later.

---

## Structure

### Global layout (`~/.claude/`)

Lives on the laptop. Applies to every Claude Code session started locally.

```
~/.claude/
├── CLAUDE.md                  # universal personal rules — short, imperative
├── settings.json              # hooks, permissions, env vars
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md           # the procedure (how to do X)
│       └── learnings.md       # gotchas captured over time
└── commands/
    └── <name>.md              # slash commands — always global, never per-project
```

**Note on cloud sessions:** Claude Code on the web clones the project repo only — it does not see `~/.claude/`. To get global skills/commands into cloud sessions, mirror `~/.claude/` into a dotfiles git repo. (Not set up yet — future task.)

### `personal-projects` monorepo layout

One Git repo for all loosely-coupled personal projects.

```
personal-projects/
├── CLAUDE.md                  # one line: follow structure.md for new files
├── structure.md               # tells Claude how to create files (agent-facing)
├── Everything-Else/
├── Home-Lab/
├── Housing-Project/
├── Job-Search/
└── System-Automation/
    └── user-learnings/        # this folder — for Usman's reference only
```

**Tracked separately:** `eol-context` and `eol-dev` (Experiment of Life) and `Perfect-Health` — each in their own folder under `~/Documents/Projects/`.

### Per-project layout

```
<project>/
├── CLAUDE.md                  # project-specific rules only
├── STATUS.md                  # current focus; rolling notes on recent decisions
└── specs/
    └── YYYY-MM-DD-<feature>.md   # one self-contained file per significant feature
```

---

## Key Decisions Made

### What belongs where

- **Global CLAUDE.md** — who Usman is, project locations, hard behavioral rules (skill invocations), environment notes. Kept very short.
- **Project CLAUDE.md** — only rules specific to that project. Never duplicates global.
- **structure.md** — agent-facing only. Tells Claude how to create new files. Not for Usman to read regularly.
- **Skills** — each skill owns its own learning loop via `learnings.md`. No per-project learnings files.
- **Memory** — project memory lives in `~/.claude/projects/.../memory/`. MEMORY.md is the index.

### What we deliberately removed

- **Tool Ecosystem table** — removed from global CLAUDE.md. MCP tool schemas are injected automatically every session; listing them again was redundant token spend.
- **Working Style bullets** — removed. Too generic to be useful to Claude; trust context.
- **"Review before edit" rule** — removed from CLAUDE.md. Rely on Claude's default behavior and task-by-task judgment.
- **Second-brain write confirmation rule** — removed from global CLAUDE.md. Covered by skill-level rules in `n8n-automation-skill`.

### Git workflow

- **`git pull`** on session start → configured as a SessionStart hook in `settings.json` (runs automatically, not a CLAUDE.md rule — hooks are more reliable).
- **`git push`** after approved changes → rule in global CLAUDE.md. Done at end of task, not automatically, to preserve a review checkpoint.
- Auto-push via hook was rejected: risk of pushing untested changes.

### Slash commands vs hooks

- **Slash commands** — user-triggered, Claude reads a `.md` file and follows instructions.
- **Hooks** — harness-triggered shell commands, run automatically at lifecycle events (SessionStart, PreToolUse, etc.). Claude doesn't decide whether they run.

### Memory system

- MEMORY.md deleted at some point by mistake — re-initialized 2026-04-18 as a blank index.
- Memory files live in `~/.claude/projects/-Users-usmanramay-Documents-Projects-Personal-Projects/memory/`.
- No memories have been captured yet as of 2026-04-18.

---

## Principles (for reference)

- **Optional, not mandatory.** Files described below exist when a project needs them.
- **One fact, one place.** Never duplicate content between global and project scopes.
- **Global = true for everything.** Project = specific to that project.
- **Skills own their own learning loop.** Lessons accumulate per-skill in `learnings.md`.
- **Token efficiency matters.** Every file loaded in CLAUDE.md costs tokens every session. Only load what Claude genuinely needs.

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
