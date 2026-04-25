# Token Optimization — Claude Code

**Last updated:** 2026-04-25

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

## Open Questions

1. **Does compaction impact the session summary recorded in local files?** If compaction rewrites the conversation, does it affect what gets saved to disk — and is any context permanently lost?

2. **What is the most token-efficient way to take quick notes?** Can Claude append to a file without reading the full file first, or is it better to use the Second Brain MCP for this? The Read tool default of 2000 lines may make file-appending expensive for longer files.

3. **What is the benefit of using custom agents over custom skills?** For example, if a skill can guide Claude through a task, when would a custom agent be preferable — and does that have a different token profile?
