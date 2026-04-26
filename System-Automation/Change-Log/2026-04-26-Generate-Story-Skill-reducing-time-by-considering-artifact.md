# Change Log — Generate Story Skill: Reducing Time by Considering Existing Artifact

**Status:** Complete
**Last updated:** 2026-04-26

> Two changes made to the generate-story skill to reduce session time while preserving quality of output.

---

## What Changed

### 1. Pre-read existing artifacts before asking any questions

When the user provides an existing artifact to analyze (a skill file, process doc, design doc, etc.), the skill now reads it in full before asking any questions. It then surfaces all identified assumptions and gaps in a single structured message — ranked by impact — and asks the user to react to the full picture at once, rather than surfacing and discussing each assumption individually.

**Before:** AI read the artifact, then started asking about assumptions one at a time (sequential loops: surface → user responds → AI reframes → next). For a session with 7 assumptions, this produced ~25 minutes of overhead.

**After:** AI reads artifact → produces one structured message with all assumptions and gaps → user reacts to the full list → one adjustment round → proceed to solution. Target: Phase 2 compresses from ~25 minutes to ~10 minutes.

### 2. General efficiency principle added

A standing instruction added near the top of the skill: wherever multiple questions or assumptions can be addressed together, club them into one message rather than asking sequentially. The goal is to reach the same output quality in less of the user's time.

---

## Why

Session analysis showed that the generate-story skill consumed ~45 minutes for a skill audit session that should have taken ~25 minutes. Half the time (Phase 2) was spent on sequential assumption-by-assumption conversation when the AI already had all the information it needed to front-load the full analysis. The sequential structure is appropriate for open-ended problems with no existing artifact, but creates unnecessary overhead when an artifact is available.

---

## Applies To

File: `/Users/usmanramay/.claude/skills/generate-story/SKILL.md`
