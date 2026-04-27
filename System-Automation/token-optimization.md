# Token Optimization — Claude Code

**Last updated:** 2026-04-26

> Reference for understanding where tokens are spent in Claude Code and how to reduce unnecessary cost.

---

## How Tokens Are Spent

| Area | Cost | Notes |
|------|------|-------|
| **System prompt + system tools** | ~10k tokens | Claude Code's own operating system; always present, unavoidable |
| **Skill descriptions** | Variable | All skill descriptions load every session — see Levers below |
| **MCP + deferred tools** | Names only | Only tool names load by default; full schema loads only if a tool is called |
| **Cache miss (after 5-min TTL)** | Full input cost | Everything in context is re-billed, not just the new message |
| **Compaction** | ~150k in + 5–8k out | On a 150k-token conversation, compaction is a significant one-time hit |

---

## Key Learnings

### 1. Cache TTL is 5 minutes

The default prompt cache TTL is 5 minutes. If the conversation goes silent for more than 5 minutes before a new message is sent, the cache has expired. The next message pays full input token cost for the entire context — not just the new message.

### 2. Compaction is expensive

Compaction summarizes the conversation when context gets large. On a 150k-token conversation, that's ~150k tokens in plus ~5–8k tokens out. It's a significant cost event, not a background operation.

### 3. Skill descriptions always load — three levers to reduce this

All skill descriptions are loaded into every session. The available levers are:

- **Limit skills to relevant projects only** — scope skills so they don't load globally
- **Trim description text** — shorter descriptions = fewer tokens per session
- **Set `disable-model-invocation: true`** on specific skills or tools that are never used

### 4. System prompt and system tools are fixed overhead

Claude Code's own operating system (system prompt, built-in tools) costs ~10k tokens. This is unavoidable — it's present in every session regardless of conversation length.

### 5. MCP and deferred tools are efficient by default

MCP tools and system tools marked as deferred load only their names into context. The full schema is only loaded when the system actually needs to call one. This means having many MCP tools doesn't necessarily cost much — unless they get invoked.

---

### 6. In-session context does not include timestamps

When reviewing a conversation from in-session context, message content is fully available — but timestamps are not. Timestamps only exist in the JSONL session file on disk (`~/.claude/projects/[project-hash]/[session-id].jsonl`).

**Practical implication:** For qualitative review (what happened, how many rounds, what felt wrong), in-session context is sufficient and reading the JSONL is unnecessary cost. Only read the JSONL if precise timing per phase is explicitly needed.

**Also note:** If `/compact` has been run, in-session context no longer contains the full message history — only the compact summary. At that point, the JSONL is the only way to recover the detailed session content.

**Recommended practice:** Before running `/compact` or closing a chat where something felt worth capturing, run the `/review-n-learn` skill first. This extracts learnings while the full conversation is still in context — the cheapest possible moment to do it.

---

### 7. Compaction does not affect JSONL session files on disk

When compaction runs, it rewrites the in-session context — but the JSONL file on disk (`~/.claude/projects/[project-hash]/[session-id].jsonl`) is not modified. The full message history, including timestamps, is always preserved in the JSONL regardless of whether compaction has run. No context is permanently lost from disk.

### 8. Bash append is the most token-efficient way to take quick notes

The most token-efficient note-taking method is a bash append command — no file read required. This avoids loading the file contents into context before writing.

The file in use is `~/.claude/quick-notes.md`. This is documented in the global CLAUDE.md (`~/.claude/CLAUDE.md`) as the standard process: *"When asked to take a note, append it to `~/.claude/quick-notes.md` without reading the file first."*

Do not use the Read tool before appending, and do not use the Second Brain MCP for quick captures — those paths cost more tokens for no added benefit.

---

## Open Questions

1. **What is the benefit of using custom agents over custom skills?** For example, if a skill can guide Claude through a task, when would a custom agent be preferable — and does that have a different token profile?
