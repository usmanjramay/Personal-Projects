# Second Brain — Project Context Summary

*Summarized from memory.md on 2026-04-01*

## What This Project Is

Usman is building a personal "second brain" knowledge management system to preserve useful thinking, decisions, and research across multiple AI tools (Claude Desktop, Cursor, Gemini). The core insight: the problem is **output loss**, not AI-to-AI memory sharing. The solution is **context injection at conversation start** rather than inter-system communication.

**Stack:** Supabase (PostgreSQL + pgvector, Frankfurt region) + n8n for automation.

---

## Current State

- **MCP server migration in progress**: Moving from a VPS-hosted Express/TypeScript MCP server to a Supabase Edge Function-based MCP server. Migration plan (8 phases) written and reviewed.
- **Active infrastructure**: VPS at Hostinger (kept for n8n only post-migration), Supabase project `waeijzskhboqphxjscah`, n8n at `n8n.srv1016866.hstgr.cloud`
- **Phone capture workflow** (`zwAM6zuOkwFsXz60`) is working end-to-end; project registry is hardcoded in Build Prompt node — identified as tech debt to replace with a `get_taxonomy` MCP tool call post-migration
- **Dictation workflow**: Built a Wispr Flow replacement using macOS built-in dictation + AI text cleanup shortcut. Root issue: real-time error visibility triggers perfectionism. Resolution direction not yet selected.

---

## On the Horizon

- Complete MCP server migration to Supabase Edge Function; cut over after parallel testing confirmed
- Replace hardcoded project registry in n8n Build Prompt node with `get_taxonomy` tool call
- Enable RLS on `memories` and `document_index` tables (currently disabled)
- Populate embeddings (currently 0 populated on existing rows)
- Resolve dictation UX: recreate Wispr Flow's "black box" pattern
- Gemini routing logic in n8n: start in ask-when-uncertain mode, move toward notify-and-trust once reliable

---

## Key Principles

- **Separation of concerns over clever unified solutions**: Project/strategy docs live in Google Drive (Drive → n8n → Supabase pipeline); conversation logs and captures go directly via MCP `add_note`; never mix these paths
- **Slugs in frontmatter, not filenames**: Ensures slugs survive file reorganization
- **MCP tool schemas are injected into context on every message**: Keep tool count and schema size lean. `get_taxonomy` replaces hardcoded enum lists
- **Black-box UX reduces perfectionism friction**: Wispr Flow's value was hiding transcription errors during dictation, not just accuracy
- **Review before edit**: Full review and approval before any file changes are made
- **Always show proposed second-brain writes for explicit confirmation** before executing
- **n8n write notifications on every write** for auditability

---

## Working Style

- Prefers staying in problem space and challenging assumptions before jumping to solutions; uses first-principles decomposition
- Pushes back on over-engineering: questions redundant tools, unnecessary enums, duplicate metadata, and runtime fetches that could be hardcoded with a sync workflow
- Works iteratively with structured approval gates (review → approve → execute)
- Prefers separate dated log entries over combined entries

---

## Tools & Resources

| Tool | Role |
|------|------|
| Supabase | PostgreSQL + pgvector, Edge Functions, service role key auth, RLS |
| n8n | Automation workflows |
| MCP (5 tools) | `add_note`, `get_note`, `search`, `patch_sections`, `get_taxonomy` |
| OpenAI | Embeddings and AI classification in n8n |
| Google Drive | Source of truth for project/strategy documents |
| Claude Desktop, Cursor, Gemini | Primary AI clients |
| macOS dictation + Shortcuts | Current Wispr Flow replacement (under review) |

**Key n8n workflow IDs:**
- Phone capture: `zwAM6zuOkwFsXz60`
- Update Workflow: `IHhN14A7NaCHeocZ`
- Test Webhook: `w4mhvZqpQwt8yyAL`
- Supabase Query: `cqS9UPhw014jAPWS`

---

## n8n Critical Technical Rules

- `openAiApi` credential is blocked in generic HTTP Request nodes — always use the native OpenAI node
- Native Supabase n8n node must never be used (triggers pgmq permission errors) — use HTTP Request with `predefinedCredentialType: supabaseApi` and `neverError: true`
- IF nodes must use `typeValidation: "loose"` with integer flags (1/0), not booleans
- Cross-branch node references (`$('NodeName')`) crash workflows — use only `$input.first()` or guaranteed ancestor nodes in Code nodes
- Free n8n plan has no Variables feature — `$vars.*` fails silently; hardcode all values in Code nodes
- When creating n8n workflows via API, always set `availableInMCP: true` in settings
