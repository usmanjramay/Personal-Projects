# Structure

How files and folders are organized across all my Claude Code work.

## Principles

- **Optional, not mandatory.** Files described below exist when a project needs them. Small projects may have none of these files — that's fine.
- **One fact, one place.** Never duplicate content between global and project scopes.
- **Global = true for everything I do.** Project = specific to that project.
- **Skills own their own learning loop.** Lessons accumulate per-skill, not per-project.

## Global layout (`~/.claude/`)

Lives on my laptop. Applies to every Claude Code session started locally.

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

## `personal-projects` monorepo layout

One Git repo for all loosely-coupled personal projects.

```
personal-projects/
├── CLAUDE.md                  # rules shared across all personal projects
├── structure.md               # this file
├── Coaching-Certification/
├── Everything-Else/
├── Home-Lab/
├── Housing-Project/
├── Job-Search/
└── System-Automation/
```

**Tracked separately:** `eol-context` (Experiment of Life) — business project, shared with other people, keeps its own repo.

## Per-project layout (when the project needs it)

Only projects with meaningful ongoing work need these files. Tiny one-off projects can skip them entirely.

```
<project>/
├── CLAUDE.md                  # project-specific rules only — no duplication of global
├── STATUS.md                  # current focus; short rolling notes on recent
│                              # decisions, learnings, and changes
└── specs/
    └── YYYY-MM-DD-<feature>.md   # one self-contained file per significant feature
                                  # date = when the spec was finalized
```

## What we deliberately don't have

Listed here so they don't accidentally creep in:

- **No per-project `learnings.md`** — learnings live with the skill they relate to, in `~/.claude/skills/<name>/learnings.md`.
- **No per-project `decisions.md`** — short decision notes roll into `STATUS.md`. Promote to a standalone ADR only if the project genuinely outgrows that.
- **No per-project `CHANGELOG.md`** — `git log` + `STATUS.md` cover it.
- **No per-project `commands/`** — all slash commands are global.
- **No per-project `skills/`** — all skills are global.
- **No sub-folders inside `specs/`** — one flat file per feature until that stops scaling.

## Skill anatomy

Each skill is a folder under `~/.claude/skills/`:

- `SKILL.md` — the how-to. Loaded on demand when the skill's trigger matches.
- `learnings.md` — appended via triage as gotchas surface. The skill's `SKILL.md` can reference it so the skill improves over time.

**Triage pattern for lessons** (what to do when something surprises me during a session):
1. **Apply now** — fix the skill / rule in place.
2. **Capture** — append dated entry with context to the relevant `learnings.md`.
3. **Dismiss** — note why it's not worth keeping.

## Specs naming

- Filename: `YYYY-MM-DD-<feature-slug>.md`
- Date = when the spec was finalized (not when it was created).
- Self-contained: context, decisions, task list, current status all in one file.
- Lives in `<project>/specs/`.

## Conflict rules (global vs project)

When the same resource exists in both scopes:

| Resource | Behavior | Token cost |
|---|---|---|
| `settings.json` | Project overrides global on conflict | None (config) |
| Skill with same name | Project version wins | None (loaded on demand) |
| Slash command with same name | Project version wins | None (loaded on invocation) |
| `CLAUDE.md` content | Both files stack and load every message | **Duplication = paying twice** |

**Rule:** each fact lives in exactly one scope. Global for universally-true things; project for project-specific things. Never both.

## Scope reminder: cloud vs laptop

- **Laptop session:** has `~/.claude/` (global) + project repo.
- **Cloud session (Claude Code on the web):** has the project repo only.
- Consequence: if I only put a skill or command globally, cloud sessions won't have it until a dotfiles-sync mechanism is set up.
