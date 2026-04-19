# Chief of Staff Agent — Minimal n8n Redesign

**Date:** 2026-04-18
**Status:** Draft — pending user review
**Author:** Usman Ramay + Claude
**Supersedes (partially):** `2026-04-14-vps-chief-of-staff-agent-design.md` — the VPS infrastructure (agent user, Patchright, email identity, backups) stays; the n8n orchestration and three-agent architecture are being replaced by the design below.

## Problem

The current Chief of Staff system treats n8n as the orchestrator. Each task is sliced into three fixed phases (Planner → Executor → Reviewer), each phase runs as a separate one-shot `claude -p` invocation with no memory of the others, and the interaction shape is hard-wired in n8n nodes. This produces a rigid, transactional experience: the same pipeline is overkill for a quick question and underpowered for a multi-step task that needs replanning. The user cannot have a flowing conversation with the agent — every WhatsApp message starts a fresh amnesiac Claude session.

## Solution

Collapse the three specialized roles into **one long-lived Claude Code CLI session** with a strong system prompt. Let Claude decide the shape of each interaction — plan when needed, ask when needed, execute, self-review, report back. Claude uses its own built-in sub-agents (Task tool) for parallel work within a turn. **n8n is reduced to a thin messaging layer** — WhatsApp in, WhatsApp out. Session continuity comes from `claude --resume <session-id>`; the conversation is preserved by Anthropic, not by a persistent local process.

## Why this approach

- **Aligned with the tool.** Claude Code CLI is designed for exactly this invocation pattern — turn-based, stateless process, session state preserved by `--resume`.
- **No fragile middleware.** No `pexpect`, no PTY parsing, no always-running Python service.
- **Inside the subscription.** Uses the OAuth token from Usman's Max plan. Zero additional API cost.
- **Expressive.** Claude gets the full context of the ongoing conversation every turn and can choose whatever interaction shape fits (ask, plan, execute, summarize).
- **Parallelism via built-ins.** Sub-agents spawn inside a turn via the Task tool, not via tmux.

## Architecture

### Components

| Component | Where | Purpose |
|-----------|-------|---------|
| `CoS: Inbound` workflow | n8n | WhatsApp trigger → fires `cos-turn.sh` on VPS |
| `CoS: Outbound` workflow | n8n | Webhook Claude calls to deliver reply to WhatsApp |
| `cos-turn.sh` | VPS, `/home/agent/bin/` | Thin shell wrapper: lock, resume/start session, run Claude, callback, unlock |
| Chief of Staff system prompt | VPS, `/home/agent/chief-of-staff/CLAUDE.md` | Role, behavior rules, sub-agent usage, closing discipline |
| `dashboard.md` | VPS, `/home/agent/chief-of-staff/` | Claude's scratchpad: in-flight, pending, completed tasks |
| Session ID file | VPS, `/home/agent/.cos-session` | Current session ID (absent = next turn starts fresh) |
| Session ID mirror | Supabase `memories` table | Redundant copy for recovery |
| Claude Code CLI | VPS, native installer | The brain |

### End-to-end turn

```
You send WhatsApp message
         │
         ▼
CoS: Inbound (WhatsApp Trigger, filtered to Usman's number)
         │
         ▼
SSH to VPS: nohup cos-turn.sh "<msg>" &   (webhook returns 200 immediately)
         │
         ▼
cos-turn.sh:
  1. flock on .cos-session.lock (queues if another turn in flight)
  2. If message starts with "/done" → append wrap-up directive to the message
  3. If .cos-session exists and session is <2 days old
        → claude --resume $SESSION_ID -p "<msg>"
     Else
        → claude -p "<msg>"  (new session; capture session_id from JSON output)
  4. Persist session ID to .cos-session + Supabase
  5. If message was "/done" → clear .cos-session on success
  6. POST Claude's final reply to CoS: Outbound webhook
  7. Release lock
         │
         ▼
CoS: Outbound
         │
         ▼
WhatsApp message back to you
```

### Concurrency model

`flock` on `/home/agent/.cos-session.lock`. A message arriving during a turn waits for the lock. Turns are processed strictly in arrival order. If you send three messages in quick succession, they run serially on the same session, not in parallel — because they'd all mutate the same conversation.

### Session lifecycle

- **Start:** Only when Usman sends a WhatsApp message. Never background-initiated. A new session is created on the first message ever, OR the first message after `/done`, OR the first message that arrives more than 2 days after the last turn.
- **Continue:** Every subsequent message within the 2-day window resumes the current session.
- **Close via `/done`:** Claude runs one wrap-up turn (save memories, update dashboard, produce summary) and then `cos-turn.sh` clears the session ID.
- **Auto-retire on next use:** If a new message arrives and `.cos-session` mtime is >2 days old, `cos-turn.sh` discards it and starts fresh. The retirement happens at message time, not on a timer — nothing runs in the background.

### `/done` semantics

The point of `/done` is not "forget everything" — it's "close this session properly." This gives the user a single, memorable command that ends a session with the closing work they want done (memory writes, file updates, dashboard cleanup, learnings captured).

Behavior:

1. User sends `/done` (optionally with trailing context, e.g. `/done thanks, that's all for tonight`).
2. `cos-turn.sh` detects the `/done` prefix and **appends a wrap-up directive** to the message before passing to Claude. Example directive: *"The user has signaled the session is complete. Perform closing work as specified in the `/done` section of your system prompt: update `dashboard.md` (mark finished tasks done, move in-progress to paused with status), save important learnings to the second brain if any exist, and return a concise summary of this session."*
3. Claude runs the turn normally — with full conversation context — and does the closing work.
4. On Claude exit 0, `cos-turn.sh` clears `.cos-session`. The next message starts fresh.

The list of things `/done` should do lives in the system prompt (`CLAUDE.md`), not in shell code. Extending the behavior later (e.g., "also commit changes to your workdir," "also post a summary to Supabase as a `digest` entry") is a system-prompt edit, not an engineering task.

## VPS setup

### Claude installation audit

Before anything else, verify the VPS is running the 2026 Claude Code native installer and clean up any legacy symlinks. Expected state:

- Single binary, likely `~/.claude/local/claude` or `/usr/local/bin/claude`.
- No residual `/usr/bin/claude` or `/bin/claude` symlinks pointing at a global npm install.
- `claude --version` returns a modern version.
- OAuth token in `/home/agent/.profile` loads correctly.

If duplicates exist, remove the npm-backed symlinks only after confirming the native installer works with `claude --version` and a successful no-op invocation.

### Directory layout

```
/home/agent/
├── bin/
│   └── cos-turn.sh              # thin wrapper invoked by n8n via SSH
├── chief-of-staff/
│   ├── CLAUDE.md                # system prompt for the agent
│   └── dashboard.md             # Claude's own task tracker
├── .cos-session                 # current session ID (plaintext)
├── .cos-session.lock            # flock target
└── logs/
    └── cos-turn-YYYY-MM-DD.log  # per-day rolling log of every turn
```

### `cos-turn.sh` responsibilities

- Accept the incoming message as a single command-line argument.
- Acquire the flock (blocks if another turn is running).
- Detect `/done` prefix; if present, augment the message with the wrap-up directive.
- Decide whether to resume: check `.cos-session` exists and is <2 days old.
- Run Claude with `--output-format json --dangerously-skip-permissions` in cwd `/home/agent/chief-of-staff/` (so `CLAUDE.md` loads as system prompt).
- Parse Claude's JSON output: extract `.result` (reply text) and `.session_id`.
- Persist session ID to `.cos-session` (and mirror to Supabase).
- If `/done`: clear `.cos-session` on success.
- POST `{reply: "<result>"}` to the outbound webhook.
- On Claude non-zero exit: POST a plain-language error + last 20 lines of stderr.
- Release lock. Append every turn to the daily log.

### `CLAUDE.md` — system prompt outline

Content the file must contain (exact wording decided during implementation):

- **Role.** Chief of Staff — receive tasks from Usman via WhatsApp, execute autonomously, report results concisely.
- **Communication style.** Under ~2000 characters per reply; never dump raw logs or command output; push detail to `dashboard.md`.
- **Sub-agents.** For non-trivial or parallelizable work, use the Task tool. Parallelize where independent.
- **Self-review.** Before replying, check your output against what Usman actually asked. If uncertain, ask him.
- **`dashboard.md` discipline.** On any turn that touches a task, update it. Schema: Active, Paused, Done, Open Questions.
- **`/done` closing work.** When the user says `/done`, update `dashboard.md`, save important learnings to the second brain (if any), return a concise summary.
- **Destructive operations forbidden.** Never `rm -rf`, never modify `/etc`, never force-push to shared repos. VPS weekly snapshot is the safety net, not a license.
- **Failure reporting.** If something fails, return what was attempted, what went wrong (specific error), and what recovery was tried.
- **Authentication.** OAuth token sourced from `.profile`.

## n8n workflows (new, additive)

### `CoS: Inbound`

- **Trigger:** WhatsApp Trigger node, credential `Send WhatsApp Personal` (Usman's personal number).
- **Filter:** Only process messages from Usman's own number (`352621486096`). Drop everything else.
- **Action:** SSH node (`Sophia's VPS SSH` credential). Command template (single line, escaped):
  ```
  nohup /home/agent/bin/cos-turn.sh "$MESSAGE_TEXT" \
    >> /home/agent/logs/cos-turn-$(date +%Y-%m-%d).log 2>&1 &
  ```
  Background dispatch so SSH returns immediately.
- **Response:** Immediate 200. No waiting on Claude.

### `CoS: Outbound`

- **Trigger:** Webhook, POST, path `/webhook/cos-outbound`.
- **Body schema:** `{ "reply": "<text>" }`.
- **Action:** WhatsApp Send (native node) using `Send WhatsApp Personal` credential. Phone number ID `1020770907789933`, recipient `352621486096`, text body `{{ $json.reply }}`.
- **Response:** 200 on success. Log failures; retry once.

### Existing workflows

All Chief of Staff workflows created in the 2026-04-14 design — Task Intake, Planner, Executor, Reviewer, Delivery, Retrospective, Weekly Review, Email Watcher, n8n Backup — remain **untouched and active** during the validation period. They are not deleted until at least one week of reliable usage with the new system confirms we don't need to roll back.

## Operational rules

| Concern | Rule |
|---------|------|
| Concurrency | `flock` on `.cos-session.lock`. Turns processed in arrival order. |
| Session retirement | `/done` (explicit) or 2-day inactivity (evaluated when the next message arrives — never in the background). |
| Long turns | No progress pings in v1. User waits silently during multi-minute turns. |
| Failure surfacing | `cos-turn.sh` posts plain-language error + stderr tail to outbound webhook on any non-zero Claude exit. |
| Destructive ops | Forbidden by system prompt. Weekly VPS snapshot is the safety net. |
| Secrets | OAuth token in `.profile`. No credentials in session file, dashboard, or memory files. |
| Reply size | System prompt caps replies at ~2000 chars. Overflow goes to `dashboard.md`. |
| Non-user messages | WhatsApp trigger filters by sender ID before SSHing. |
| Logging | Per-day rolling log in `/home/agent/logs/`. No rotation policy in v1. |

## Deferred (explicitly not in v1)

| Feature | Defer until |
|---------|-------------|
| Progress pings on long turns | User reports the silence is a problem |
| True background worker processes (multi-day tasks) | User has a concrete multi-day task |
| Extra slash commands beyond `/done` | User asks for one |
| Web UI dashboard | User wants one |
| Deletion of old Chief of Staff n8n workflows | ≥1 week of reliable new-system usage |
| Separate sessions per task-thread | User finds a single long session gets confused across topics |
| Retrospective / weekly self-improvement loop | Post-v1; re-evaluate whether still worth building once new design has run |

## Success criteria

This redesign is successful when:

1. Usman sends a WhatsApp message. Agent's reply arrives. Under the hood the session was resumed, one Claude turn ran, and the agent is ready for the next message.
2. Usman sends a second message 20 minutes later. Claude knows what was discussed and continues naturally — no re-briefing needed.
3. Usman sends a complex task ("research X and Y"). Claude spawns sub-agents, synthesizes, and replies once with the combined result.
4. Usman sends a message while Claude is mid-turn on a previous one. The second message queues and is processed after the first turn completes.
5. Usman sends `/done`. Claude updates `dashboard.md`, writes any important memory, and replies with a session summary. The next message starts a new session.
6. On any failure (Claude error, OAuth expiry, SSH hiccup, rate limit), Usman sees a plain-language error via WhatsApp — never silence.
7. All existing n8n workflows continue to exist untouched throughout the validation week.

## Open decisions parked for implementation

- **Supabase mirror shape.** Store session ID as a `memories` row with category `log` and slug `cos/session/<date>`, or add a dedicated `sessions` column/table. Lean toward reusing `memories` for simplicity.
- **Claude CLI output parsing.** Use `--output-format json`, read `.result` and `.session_id`. Verify field stability during build — if the schema shifts, fall back to text output + `--print-session-id` or equivalent.
- **Lock timeout.** Should `flock` wait indefinitely or time out after N seconds? Default to wait indefinitely in v1; revisit if we see stuck locks.
- **Rate-limit handling.** If Claude returns a rate-limit error, `cos-turn.sh` surfaces it verbatim. Future: detect and auto-retry after the window.
- **Message passing to the shell.** WhatsApp messages can contain quotes, backticks, newlines, and emojis — passing them as a raw shell argument risks quoting issues and command-injection. Decide during build whether to base64-encode the message in the SSH command and decode in `cos-turn.sh`, or pipe it via `stdin`. Do not pass user text as an unquoted shell argument.
