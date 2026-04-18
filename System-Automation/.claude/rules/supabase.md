---
description: "Supabase database, edge functions, and MCP server rules"
globs:
  - "second_brain_setup.md"
  - "**/*supabase*"
  - "**/*edge*function*"
  - "**/*mcp*"
---

# Supabase & MCP Server Rules

## Project

- Project ID: `waeijzskhboqphxjscah`
- URL: `https://waeijzskhboqphxjscah.supabase.co`
- Region: Frankfurt (default)

## Tables

### `memories` — Main content store
One row per entry version (active + archived history).

Key columns: `id` (uuid), `slug` (text, unique identifier), `title`, `content` (markdown), `project`, `topic`, `category` (doc/log/capture/digest), `date`, `source` (ai/n8n/phone), `embedding` (vector 1536), `archived` (bool), `is_version` (bool)

### `document_index` — Lightweight metadata index
For fast taxonomy queries without loading full content.

Key columns: `id` (uuid), `slug` (text), `title`, `project`, `topic`, `category`, `date`, `source`, `summary` (text)

## Edge Functions

- `second-brain-mcp` (v7) — The MCP server (Streamable HTTP). Handles AI client tool calls.
  - URL: `https://waeijzskhboqphxjscah.supabase.co/functions/v1/second-brain-mcp`
  - Auth: service role key (auto-injected by Supabase runtime)
  - Clients: Claude Desktop, Claude.ai, Cursor, Gemini (all connect directly)
- `embed` (v2) — Async embedding worker. Triggered after writes. Uses OpenAI text-embedding-3-small.

## Database Procedures (created during migration)

- `add_note` — stored procedure for note creation
- `patch_sections` — stored procedure for atomic section updates
- `get_taxonomy` — stored procedure for taxonomy queries
- RLS enabled on both tables (service_role has full access policies)

## MCP Tools (5 total)

1. `get_taxonomy` — Returns projects/topics tree + slug templates
2. `add_note` — Creates new entries (triggers embeddings automatically)
3. `get_note` — Fetches full entry content by slug
4. `search` — Browse mode (SQL) or semantic mode (vector similarity)
5. `patch_sections` — Updates sections atomically, creates versions

## Environment Variables

| Variable | Used by | Notes |
|---|---|---|
| `SUPABASE_URL` | both functions | Auto-injected by Supabase runtime |
| `SUPABASE_SERVICE_ROLE_KEY` | both functions | Auto-injected. Bypasses RLS. |
| `OPENAI_API_KEY` | both functions | Set manually in dashboard → Edge Functions → Secrets |
