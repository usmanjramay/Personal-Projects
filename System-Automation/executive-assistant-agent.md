# Executive Assistant (EA) Agent

**Status:** Testing
**Last updated:** 2026-04-19

> A personal AI agent on the Hostinger VPS. Send a WhatsApp message, it executes the task autonomously using Claude Code CLI, and replies via WhatsApp. Conversation persists across messages using `--resume`.

---

## User Story

**Pain Points**
1. No easy way to capture a thought, idea, or task quickly when on the move — most communication happens via WhatsApp, but opening separate apps or systems creates enough friction to lose the thought.
2. For tasks that require research, browsing, comparison, or iteration (finding flights, researching purchases, setting up services), the human is still doing most of the legwork — AI conversations are transactional with no memory across sessions.
3. Security concern: running an autonomous agent on a personal computer with personal accounts is risky — if it makes a mistake, it could affect real accounts, data, or systems.

**Wish List**
1. Send a WhatsApp message with a task or thought; receive a finished result or confirmation with no further input required
2. Follow-up messages continue the same conversation naturally — no re-briefing needed
3. Complex tasks that need parallel research are handled internally; one consolidated reply
4. If a message arrives while the agent is mid-task, it queues and processes in order — no collision
5. Sending `/done` closes the session cleanly: dashboard updated, learnings saved, summary sent
6. Any failure surfaces as a plain-language WhatsApp message — never silent
7. The agent operates from its own accounts and identity, not Usman's personal ones

_This list is the success criteria. Items 1–6 are confirmed. Item 7 is partially met (email identity exists; social media accounts pending)._

**Potential Risks**
1. No progress updates during long turns in v1 — user waits silently with no feedback
2. Session auto-retires after 2 days of inactivity (evaluated at next message, not on a background timer)

---

## How It Works

### Components

| Component | Where | Purpose |
|-----------|-------|---------|
| `CoS: Inbound` workflow | n8n (see @n8n_setup.md) | WhatsApp trigger → SSH → fires `cos-turn.sh` in background |
| `CoS: Outbound` workflow | n8n | Webhook → sends Claude's reply to WhatsApp |
| `cos-turn.sh` | VPS `/home/agent/bin/` (see @remote_vps.md) | Lock, resume/start session, run Claude, post reply, unlock |
| `CLAUDE.md` (system prompt) | VPS `/home/agent/chief-of-staff/` | Role, behavior rules, sub-agent usage, `/done` discipline |
| `dashboard.md` | VPS `/home/agent/chief-of-staff/` | Claude's own task scratchpad |
| `.cos-session` | VPS `/home/agent/` | Current session ID (plaintext) |
| `.cos-session.lock` | VPS `/home/agent/` | flock target — enforces serial turn processing |
| Claude Code CLI | VPS | The brain. Native installer, OAuth from Usman's Max plan. |
| Agent email | `sophia@ptriconsulting.com` | EA's own identity for sign-ups, verifications, and outbound emails |

### End-to-end flow

```
You → WhatsApp message
         ↓
CoS: Inbound (filters to Usman's number only)
         ↓
SSH to VPS: nohup cos-turn.sh "<msg>" &   (returns 200 immediately)
         ↓
cos-turn.sh:
  1. Acquire flock (queues if another turn is running)
  2. If "/done" → append wrap-up directive to message
  3. .cos-session exists and <2 days old → claude --resume $SESSION_ID -p "<msg>"
     Otherwise → claude -p "<msg>" (new session)
  4. Persist session ID to .cos-session
  5. If "/done" → clear .cos-session on success
  6. POST Claude's reply to CoS: Outbound webhook
  7. Release lock
         ↓
CoS: Outbound → WhatsApp reply to you
```

### Session lifecycle

- **Start:** First message ever, after `/done`, or after 2-day inactivity
- **Continue:** Every message within the 2-day window resumes the current session
- **Close:** `/done` — Claude updates dashboard, saves learnings, sends summary, session cleared
- **Failure:** Non-zero Claude exit → plain-language error + stderr tail sent to WhatsApp

### Agent identity

The EA has its own email address (`sophia@ptriconsulting.com`) which it uses to create accounts on external services and send emails on Usman's behalf. This keeps all agent activity separate from Usman's personal accounts.

### Architecture & Design Rationale

This minimal n8n architecture (single long-lived Claude session) was chosen because:

- **Aligned with the tool.** Claude Code CLI is designed for exactly this invocation pattern — turn-based, stateless process, session state preserved by `--resume`.
- **No fragile middleware.** No `pexpect`, no PTY parsing, no always-running Python service.
- **Inside the subscription.** Uses the OAuth token from Usman's Max plan. Zero additional API cost.
- **Expressive.** Claude gets the full context of the ongoing conversation every turn and can choose whatever interaction shape fits (ask, plan, execute, summarize).
- **Parallelism via built-ins.** Sub-agents spawn inside a turn via the Task tool, not via tmux.

The original three-agent architecture (Planner → Executor → Reviewer) was rigid: the same pipeline was overkill for quick questions and had no conversation continuity across turns. Each task was sliced into fixed phases with separate one-shot Claude invocations, so the agent had no memory of previous phases.

### VPS Setup

#### Directory Layout

```
/home/agent/
├── bin/
│   └── cos-turn.sh              # thin wrapper invoked by n8n via SSH
├── chief-of-staff/
│   ├── CLAUDE.md                # system prompt for the agent
│   └── dashboard.md             # Claude's own task tracker
├── memory/
│   └── whatsapp-token.txt       # permanent WhatsApp system user access token (chmod 600)
├── workdir/                     # task scratch space; files >7 days auto-deleted
├── .cos-session                 # current session ID (plaintext)
├── .cos-session.lock            # flock target for concurrency control
└── logs/
    └── cos-turn-YYYY-MM-DD.log  # per-day rolling log of every turn
```

#### Claude Installation Audit

Before deploying, verify the VPS runs the 2026 Claude Code native installer:

- Single binary, likely `~/.claude/local/claude` or `/usr/local/bin/claude`.
- No residual `/usr/bin/claude` or `/bin/claude` symlinks pointing at a global npm install.
- `claude --version` returns a modern version.
- OAuth token in `/home/agent/.profile` loads correctly.

If duplicates exist, remove npm-backed symlinks only after confirming the native installer works with `claude --version` and a successful no-op invocation.

#### `cos-turn.sh` Responsibilities

The thin shell wrapper handles:

1. Accept `$1` (base64-encoded message, may include reply-context prefix) and `$2` (optional WhatsApp media ID).
2. Decode `$1` to get the message text.
3. **If `$2` is non-empty (media message):** read WhatsApp token from `/home/agent/memory/whatsapp-token.txt`, call the WhatsApp API to get the media download URL, download the file to `/home/agent/workdir/photo-<timestamp>.jpg`, prepend `[Image: <path>]` to the message. On any failure (token missing, API error, empty file), prepend a plain-language error note instead.
4. Acquire `flock` on `.cos-session.lock` (blocks if another turn is running; processes messages strictly in arrival order).
5. Detect `/done` prefix; if present, augment the message with the wrap-up directive before passing to Claude.
6. Decide whether to resume: check `.cos-session` exists and is <2 days old.
   - If yes: `claude --resume $SESSION_ID -p "<msg>"`
   - If no: `claude -p "<msg>"` (new session)
7. Parse Claude's JSON output: extract `.result` (reply text) and `.session_id`.
8. Persist session ID to `.cos-session` (and optionally mirror to Supabase for recovery).
9. If `/done`: clear `.cos-session` on success.
10. POST `{reply: "<result>"}` to the CoS: Outbound webhook.
11. On Claude non-zero exit: POST a plain-language error + last 20 lines of stderr to WhatsApp.
12. Release lock. Append every turn to the daily log (`/home/agent/logs/cos-turn-YYYY-MM-DD.log`).

**Important:** Messages are passed safely via SSH (not as unquoted shell arguments) to avoid command-injection and quoting issues with user text containing quotes, backticks, newlines, or emojis.

#### `CLAUDE.md` System Prompt Outline

The system prompt file defines:

- **Role:** Executive Assistant — receive tasks from Usman via WhatsApp, execute autonomously, report results concisely.
- **Communication style:** Under ~2000 characters per reply; never dump raw logs or command output; push detail to `dashboard.md`.
- **Tools:** Playwright browser scripts in `/home/agent/scripts/`; email identity `sophia@ptriconsulting.com` with credentials at `/home/agent/memory/email-credentials.md`; Task tool for sub-agents and parallelism.
- **`dashboard.md` discipline:** On any turn that touches a task, update it. Schema: Active, Paused, Done, Open Questions.
- **`/done` closing work:** Update `dashboard.md`, save important learnings to the second brain, return a concise summary.
- **Destructive operations forbidden:** Never `rm -rf`, never modify `/etc`, never force-push to shared repos.
- **Failure reporting:** What was attempted → what went wrong (specific error) → what recovery was tried.
- **Directory rule:** All task files (scripts, screenshots, data) go to `workdir/`, never to `chief-of-staff/`. Files in `workdir/` older than 7 days are auto-deleted by `cos-turn.sh`.

### `/done` Semantics

The point of `/done` is not "forget everything" — it's "close this session properly." This gives the user a single, memorable command that ends a session with the closing work they want done (memory writes, file updates, dashboard cleanup, learnings captured).

Behavior:

1. User sends `/done` (optionally with trailing context, e.g. `/done thanks, that's all for tonight`).
2. `cos-turn.sh` detects the `/done` prefix and **appends a wrap-up directive** to the message before passing to Claude.
3. Claude runs the turn normally — with full conversation context — and does the closing work.
4. On Claude exit 0, `cos-turn.sh` clears `.cos-session`. The next message starts fresh.

### n8n Workflows

#### `CoS: Inbound`

- **Trigger:** WhatsApp Trigger node, credential `Send WhatsApp Personal` (Usman's personal number).
- **Filter:** Only process messages from Usman's own number (`352621486096`). Drop everything else.
- **Encode Message node:** Detects message type (text, image, document, video, audio). Extracts media ID and caption/text. If the message is a WhatsApp reply (using the reply function), prepends `[Replying to: <message_id>]` to the text. Base64-encodes the final message.
- **Action:** SSH node (`Sophia's VPS SSH` credential). Command: `nohup /home/agent/bin/cos-turn.sh <msg_b64> <media_id> >> /home/agent/logs/cos-turn-$(date +%Y-%m-%d).log 2>&1 &`
  - `$1` = base64-encoded message (may include reply context prefix)
  - `$2` = WhatsApp media ID (empty string for text-only messages)
  - Background dispatch so SSH returns immediately (webhook responds 200 without waiting on Claude).
- **Response:** Immediate 200. No waiting on Claude.

#### `CoS: Outbound`

- **Trigger:** Webhook, POST, path `/webhook/cos-outbound`.
- **Body schema:** `{ "reply": "<text>" }`.
- **Action:** WhatsApp Send (native node) using `Send WhatsApp Personal` credential. Phone number ID `1020770907789933`, recipient `352621486096`, text body `{{ $json.reply }}`.
- **Response:** 200 on success. Log failures; retry once.

#### Existing Workflows (April 14 Architecture)

All Chief of Staff workflows created in the original three-agent design — Task Intake, Planner, Executor, Reviewer, Delivery, Retrospective, Weekly Review, Email Watcher, n8n Backup — remain **untouched and active** during the validation period. They are not deleted until at least one week of reliable usage with the new system confirms we don't need to roll back.

### Operational Rules

| Concern | Rule |
|---------|------|
| **Concurrency** | `flock` on `.cos-session.lock`. Turns processed in arrival order. |
| **Session retirement** | `/done` (explicit) or 2-day inactivity (evaluated when the next message arrives — never in the background). |
| **Long turns** | No progress pings in v1. User waits silently during multi-minute turns. |
| **Failure surfacing** | `cos-turn.sh` posts plain-language error + stderr tail to outbound webhook on any non-zero Claude exit. |
| **Destructive ops** | Forbidden by system prompt. Weekly VPS snapshot is the safety net. |
| **Secrets** | OAuth token in `.profile`. No credentials in session file, dashboard, or memory files. |
| **Reply size** | System prompt caps replies at ~2000 chars. Overflow goes to `dashboard.md`. |
| **Non-user messages** | WhatsApp trigger filters by sender ID before SSHing. |
| **Logging** | Per-day rolling log in `/home/agent/logs/`. No rotation policy in v1. |

### Success Criteria

This system is successful when:

1. Usman sends a WhatsApp message. Agent's reply arrives. Under the hood the session was resumed, one Claude turn ran, and the agent is ready for the next message.
2. Usman sends a second message 20 minutes later. Claude knows what was discussed and continues naturally — no re-briefing needed.
3. Usman sends a complex task ("research X and Y"). Claude spawns sub-agents, synthesizes, and replies once with the combined result.
4. Usman sends a message while Claude is mid-turn on a previous one. The second message queues and is processed after the first turn completes.
5. Usman sends `/done`. Claude updates `dashboard.md`, writes any important memory, and replies with a session summary. The next message starts a new session.
6. On any failure (Claude error, OAuth expiry, SSH hiccup, rate limit), Usman sees a plain-language error via WhatsApp — never silence.
7. All existing n8n workflows continue to exist untouched throughout the validation week.

### Deferred Features (Explicitly Not in v1)

| Feature | Defer until |
|---------|-------------|
| Progress pings on long turns | User reports the silence is a problem |
| True background worker processes (multi-day tasks) | User has a concrete multi-day task |
| Extra slash commands beyond `/done` | User asks for one |
| Web UI dashboard | User wants one |
| Deletion of old n8n workflows | ≥1 week of reliable new-system usage |
| Separate sessions per task-thread | User finds a single long session gets confused across topics |
| Retrospective / weekly self-improvement loop | Post-v1; re-evaluate whether still worth building |

### Open Implementation Decisions

- **Supabase mirror shape.** Store session ID as a `memories` row with category `log` and slug `cos/session/<date>`, or add a dedicated `sessions` column/table. Lean toward reusing `memories` for simplicity.
- **Claude CLI output parsing.** Use `--output-format json`, read `.result` and `.session_id`. Verify field stability during build — if the schema shifts, fall back to text output + `--print-session-id` or equivalent.
- **Lock timeout.** Should `flock` wait indefinitely or time out after N seconds? Default to wait indefinitely in v1; revisit if we see stuck locks.
- **Rate-limit handling.** If Claude returns a rate-limit error, `cos-turn.sh` surfaces it verbatim. Future: detect and auto-retry after the window.
- **Message passing to the shell.** WhatsApp messages can contain quotes, backticks, newlines, and emojis. Passing them as a raw shell argument risks quoting issues and command-injection. Decide during build whether to base64-encode the message in the SSH command and decode in `cos-turn.sh`, or pipe it via `stdin`. Do not pass user text as an unquoted shell argument.

---

## Current Status

Working end-to-end. Wish list items 1–6 confirmed. Item 7 (own accounts) partially met — email identity in place, social media accounts not yet created.

Old April 14 n8n workflows (three-agent architecture) still active — kept during 1-week validation period before deletion.

---

## Open Tasks and Ideas

- [x] Delete old April 14 n8n workflows (Task Intake, Executor, Reviewer, Delivery, Retrospective, Email Watcher, Weekly Review, n8n Backup, Agent: Send WhatsApp) and remove associated VPS files from the old three-agent architecture (`prompts/`, `schemas/`) — _completed 2026-04-21_
- [x] Photo/media support: `CoS: Inbound` n8n workflow updated to detect image/video/document/audio and pass media_id to VPS — _completed 2026-04-21_
- [x] Reply-context support: when Usman uses WhatsApp's reply function, the message is prefixed with `[Replying to: <id>]` so the EA knows which message is being referenced — _completed 2026-04-21_
- [ ] **VPS setup required:** add the image download block to `cos-turn.sh` (Step 2 from the implementation session), and store the WhatsApp access token at `/home/agent/memory/whatsapp-token.txt` (Step 1)
- [ ] Build a French phone agent — Usman lives in Luxembourg and frequently needs to make calls in French but does not speak the language. The agent should be able to make outbound calls on his behalf, conduct the conversation in French, and report back a summary. Needs research into the right platform (e.g. Bland.ai, Retell.ai, or similar) and integration with the EA so Usman can trigger a call via WhatsApp.
- [ ] Create social media accounts for the EA under its own identity
- [ ] Decide on a process for the EA sending emails on Usman's behalf — what authorization looks like, what guardrails are needed
- [ ] Give the EA access to the Old iPhone as an air-gapped browser for bot-resistant web tasks (flight search, etc.) — see [Ideas/Old-Iphone-Agent](Ideas/Old-Iphone-Agent) for full spec and implementation checklist
- Idea: progress ping after N minutes on long turns (currently user waits silently)
- Idea: extra slash commands beyond `/done` (e.g. `/pause`, `/status`)
- Idea: web UI dashboard once the system is mature

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-19 | Renamed from "Chief of Staff" to "EA (Executive Assistant)" | More accurate name for the role |
| 2026-04-18 | Redesigned to minimal n8n architecture — single long-lived Claude session, n8n reduced to thin messaging layer | Original three-agent design was rigid: same pipeline was overkill for quick questions, no conversation continuity across turns |
| 2026-04-14 | Initial build — three-agent architecture (Planner → Executor → Reviewer) orchestrated by n8n | First version of the autonomous agent concept |
