# Second Brain — Setup

> Living reference for the infrastructure, data model, and design decisions behind this system.
> Update this document when the architecture changes, not when entries are added.

---

## Overview

A personal knowledge base built on Supabase. Entries are written by AI assistants (Claude, Cursor) or via automated pipelines (n8n, phone). All content is stored as markdown in Postgres with vector embeddings for semantic search. Access from AI clients is through a single MCP server hosted as a Supabase Edge Function.

---

## Supabase Project

- **Project ID**: `waeijzskhboqphxjscah`
- **Region**: default
- **URL**: `https://waeijzskhboqphxjscah.supabase.co`

### Environment Variables

| Variable | Used by | Notes |
|---|---|---|
| `SUPABASE_URL` | both functions | Auto-injected by Supabase runtime |
| `SUPABASE_SERVICE_ROLE_KEY` | both functions | Auto-injected. Bypasses RLS. |
| `OPENAI_API_KEY` | both functions | Set manually in Supabase dashboard → Edge Functions → Secrets |

---

## Tables

### `memories`

The main content store. One row per entry version (active entries plus their archived history).

| Column | Type | Notes |
|---|---|---|
| id | uuid | Primary key |
| slug | text | Unique identifier, e.g. `doc/home-lab/local-server/setup` |
| title | text | Human-readable title |
| content | text | Full markdown content |
| project | text | e.g. `home-lab`, `system-design` |
| topic | text | e.g. `local-server`, `second-brain` |
| category | text | `doc`, `log`, `capture`, or `digest` |
| date | date | Entry date (not created_at) |
| source | text | `ai`, `n8n`, `phone`, etc. |
| embedding | vector(1536) | OpenAI text-embedding-3-small |
| archived | bool | true = old version, not the live entry |
| is_version | bool | true = was archived by patch_sections |
| archived_at | timestamptz | When this version was archived |
| created_at | timestamptz | Row insertion time |

**RLS**: enabled. Service role key has full access. Anonymous role has no access.

**Active entry**: `archived = false AND is_version = false`. Always use both filters together.

**Version history**: when `patch_sections` runs, the current row is set `archived=true, is_version=true, archived_at=now()` and a new active row is inserted. Versions older than 30 days are deleted nightly by cron.

### `document_index`

A lightweight metadata index — one row per active entry, no content, no embeddings. Used by `get_taxonomy` to return the project/topic tree without scanning `memories`.

| Column | Type | Notes |
|---|---|---|
| slug | text | Primary key |
| title | text | |
| project | text | |
| topic | text | |
| category | text | |
| updated_at | timestamptz | Set on insert and patch |

Written to by `add_note` (INSERT) and `patch_sections` (ON CONFLICT UPDATE). Never written to directly.

**Why a separate table**: `memories` grows with every archived version and contains large content + embedding columns. Querying it for taxonomy would be slow at scale. `document_index` is always small and fast.

---

## Embedding Pipeline

New embeddings are generated asynchronously, not at write time.

**Trigger**: `embed_memories` (ON INSERT OR UPDATE on `memories`) calls `util.queue_embeddings()`, which pushes a job into the pgmq queue `embedding_jobs`. The job payload is `{ id, content }`.

**Worker**: the `embed` Edge Function reads up to 10 jobs per invocation, calls OpenAI, and writes the vector back to `memories.embedding`. It then deletes the job from the queue.

**Why async**: generating embeddings via OpenAI adds ~300–500ms of latency. Doing it synchronously inside `add_note` would make every write feel slow. The queue lets writes return immediately and embeddings catch up within a minute.

**Why pgmq**: already available in Supabase, no external queue needed. Supports visibility timeouts so a job that fails mid-processing becomes visible again after 60 seconds rather than being lost.

---

## Edge Functions

### `second-brain-mcp` (v7)

The MCP server. Handles all AI client connections via Streamable HTTP (JSON-RPC 2.0 over POST).

**URL**: `https://waeijzskhboqphxjscah.supabase.co/functions/v1/second-brain-mcp`

**Auth**: dual method — Bearer header for Cursor/Claude Desktop, `?key=` query param for Claude.ai connector which only accepts a URL.

**Why not Supabase JWT**: AI MCP clients can't do the token-obtain flow. A static API key is simpler and sufficient for personal use.

**Why no resources**: resources would send full entry content on every message, wasting tokens. Tools on demand is the correct pattern.

### `embed` (v2)

The embedding worker. Called by cron every minute. Always uses `SUPABASE_SERVICE_ROLE_KEY` to bypass RLS when writing embeddings back.

**Why SECURITY DEFINER helpers**: pgmq lives in its own schema. Two `SECURITY DEFINER` wrapper functions (`read_embedding_jobs`, `delete_embedding_job`) in the `public` schema allow the Edge Function to interact with pgmq via the Supabase client.

---

## MCP Tools

| Tool | Mode | What it does |
|---|---|---|
| `get_taxonomy` | — | Returns projects/topics from `document_index` + slug templates. Call before writing or browsing. |
| `add_note` | — | Inserts a new entry. Triggers embedding automatically. |
| `get_note` | — | Fetches full content of one entry by slug. |
| `search` | browse | No query: SQL filter by project/topic/category, newest first. Use when user names a project/topic. |
| `search` | semantic | With query: vector similarity search. Falls back to keyword (`ilike`) if nothing found. |
| `patch_sections` | — | Replaces or appends sections atomically. Archives old version, inserts new active version. |

**Search decision rule**: if the user names a project, topic, or category → browse mode (omit `query`). If the user asks a conceptual question → semantic mode (provide `query`).

---

## Stored Procedures

**`add_note`** — inserts into `memories`, upserts into `document_index`. Embedding trigger fires automatically.

**`patch_sections`** — takes an array of `{ heading, new_content }` patches. Applies all patches in memory, then archives the old row and inserts a new active row. One version per call, not per section.

**`get_taxonomy`** — reads `document_index`, returns `{ projects: [{ project, topics[] }], categories[] }`. Categories are hardcoded since they are schema-level constants.

**`search_memories`** — filters `archived=false AND is_version=false`, computes cosine similarity, returns rows above threshold ordered by similarity. Threshold in the Edge Function is `0.3` (permissive) to handle short/single-word queries.

**`read_embedding_jobs` / `delete_embedding_job`** — SECURITY DEFINER wrappers around pgmq. Bridge the schema gap between the Edge Function's Supabase client and the pgmq schema.

---

## Cron Jobs

**`process-embedding-jobs`** — every minute (`* * * * *`). Calls `embed` Edge Function with `{ batch_size: 10 }`. Processes up to 10 pending embeddings per minute. Returns immediately if queue is empty.

**`delete-old-archives`** — nightly at 3am (`0 3 * * *`). Deletes archived versions older than 30 days. Prevents unbounded table growth as entries get patched over time. 30 days is enough to recover from a bad patch.

---

## Client Connections

| Client | Transport | Auth |
|---|---|---|
| Claude.ai | MCP connector (HTTP) | `?key=` in URL |
| Cursor | `~/.cursor/mcp.json` | Bearer header |
| Claude Desktop | Not connected | Removed to avoid duplicate with Claude.ai |
| n8n | Supabase REST API | Service role key |
| Phone | n8n webhook → REST | n8n handles auth |

**Why n8n bypasses MCP**: n8n writes structured data with known taxonomy values. No AI tool-calling overhead needed. Embeddings still generated automatically via trigger.
