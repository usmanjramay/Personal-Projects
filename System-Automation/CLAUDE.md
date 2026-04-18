# System Automation — Second Brain

## What This Is

A personal knowledge management system that preserves thinking, decisions, and research across AI tools (Claude Desktop, Cursor, Gemini). The core problem is **output loss** — conversations get lost. The solution is **context injection at conversation start** via MCP.

**Stack:** Supabase (PostgreSQL + pgvector, Frankfurt) → n8n (automation) → MCP server (Supabase Edge Function)

## Current Focus

- MCP server migration **completed** (2026-03-23): now running as Supabase Edge Function (`second-brain-mcp`). VPS MCP server decommissioned.
- Phone capture workflow (`zwAM6zuOkwFsXz60`) is working end-to-end
- Tech debt: project registry hardcoded in Build Prompt node — replace with `get_taxonomy` tool call
- iWhisper dictation system built: Hammerspoon + Apple Shortcuts for dictation and AI text cleanup (double Ctrl = improve text, Ctrl+Option+I = dictation toggle). Code at `~/Documents/iWhisper/`
- RLS enabled on `memories` and `document_index` tables
- Pending: populate embeddings, Gemini routing logic

## Key Decisions

- Separation of concerns: project docs → Google Drive → n8n → Supabase; conversation logs → MCP `add_note` directly
- Black-box UX reduces perfectionism friction (Wispr Flow's value was hiding errors, not accuracy). iWhisper replaces Wispr Flow.
- MCP server is a Supabase Edge Function (URL: `https://waeijzskhboqphxjscah.supabase.co/functions/v1/second-brain-mcp`). VPS retained for n8n only.
- Categories: `doc`, `log`, `capture`, `digest`
- Embeddings: OpenAI text-embedding-3-small (1536 dimensions), async via `embed` edge function

## Reference Docs

- Full infrastructure and data model: see `second_brain_setup.md`
- n8n workflow configuration and node rules: see `n8n_setup.md`
- VPS access and legacy MCP server: see `remote_vps.md`
- High-level project summary: see `second-brain-context.md`

## Do Not

- Never use the native Supabase n8n node (triggers pgmq permission errors)
- Never use `$vars.*` in n8n — free plan has no Variables feature
- Never make cross-branch node references (`$('NodeName')`) in n8n Code nodes
- Never write to the second-brain without showing proposed content for confirmation first
