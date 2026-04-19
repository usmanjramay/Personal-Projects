# Chief of Staff — Minimal n8n Redesign: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the three-agent n8n-orchestrated Chief of Staff with a single long-lived Claude Code CLI session that's poked by thin n8n messaging workflows.

**Architecture:** n8n receives WhatsApp → SSH-dispatches `cos-turn.sh` on the VPS (background, no wait) → shell wrapper runs `claude --resume <id>` → Claude's reply is POSTed to an n8n outbound webhook → WhatsApp delivery. Session continuity via `--resume`. Existing CoS workflows stay untouched for the 1-week validation period.

**Tech Stack:** Bash, Claude Code CLI (2026 native installer), n8n (self-hosted on Hostinger VPS), WhatsApp Business API, Supabase (session ID mirror, optional).

**Spec:** `docs/superpowers/specs/2026-04-18-chief-of-staff-minimal-n8n-design.md`

**Notes on this plan's style:**
- This is a VPS ops + n8n automation project, not a typical code repo. There is no unit-test harness. Each task has a manual verification step instead of `pytest`.
- There is no git repo at the project root (confirmed via env). We skip git commits. Progress is tracked via the checkboxes in this file.
- n8n workflow construction uses the n8n MCP tools live in this session (`mcp__claude_ai_n8n__*` or `mcp__7d622793-*`).
- SSH is done via `ssh agent@46.202.159.171`. The user's SSH key is expected to be already loaded in `ssh-agent`. If SSH fails, pause and ask the user to run `ssh-add` (per memory `feedback_ssh_access.md`).

**What happens when:**
- Tasks 1–3: Audit + prep the VPS.
- Tasks 4–7: Write the Chief of Staff files (system prompt, dashboard, wrapper script).
- Tasks 8–10: Build and wire the Outbound n8n workflow; smoke-test the WhatsApp delivery path.
- Tasks 11–12: Build and wire the Inbound n8n workflow; end-to-end test.
- Tasks 13–16: Behavioural tests (continuity, `/done`, concurrency, failure surfacing).
- Task 17: Update memory with final state.

---

## Task 1: Audit Claude Code installation on the VPS

**Files:** none (read-only)

- [ ] **Step 1: SSH in and find all Claude binaries**

```bash
ssh agent@46.202.159.171 'bash -lc "
echo --- which claude ---
which -a claude 2>&1 || true
echo --- ls binaries ---
for p in /usr/bin/claude /bin/claude /usr/local/bin/claude ~/.claude/local/claude; do
  if [ -e \$p ]; then
    ls -l \$p
  fi
done
echo --- version ---
claude --version 2>&1 || true
echo --- npm global check ---
npm root -g 2>/dev/null && ls \$(npm root -g) 2>/dev/null | grep -i claude || echo no-npm-global-claude
"'
```

Expected observations:
- A path like `~/.claude/local/claude` (modern native install), OR
- Legacy symlinks at `/usr/bin/claude` and/or `/bin/claude` pointing at a global npm module (`@anthropic-ai/claude-code`), OR
- Both (duplicate install — the thing we want to clean up).

- [ ] **Step 2: Record findings in a note**

Write a short summary to `/tmp/cos-install-audit.txt` locally (not on the VPS) capturing what exists. This informs Task 2.

- [ ] **Step 3: Verify the native installer works**

```bash
ssh agent@46.202.159.171 'bash -lc "source ~/.profile && claude --version && claude -p \"say ok\" --output-format json --dangerously-skip-permissions 2>&1 | head -c 400"'
```

Expected: A version number ≥ v2.x, and a JSON blob containing `.result` with some short reply, `.session_id`, etc.

If this fails: stop and diagnose before proceeding. Common issues: OAuth token not loaded (`CLAUDE_CODE_OAUTH_TOKEN` missing), wrong PATH, native install broken.

---

## Task 2: Clean up duplicate Claude installs (only if Task 1 found duplicates)

**Files:** none (VPS symlinks only)

Skip this task entirely if Task 1 showed only the native install. Otherwise:

- [ ] **Step 1: Confirm the native install is the canonical one**

Before deleting anything, make sure the native install at `~/.claude/local/claude` (or wherever Task 1 found it) runs successfully from Task 1 Step 3. Do NOT delete the legacy install until the native install is proven working.

- [ ] **Step 2: Remove legacy symlinks**

```bash
ssh agent@46.202.159.171 'bash -lc "
sudo rm -f /usr/bin/claude /bin/claude
which -a claude
claude --version
"'
```

Expected: `which -a claude` returns only the native-install path; `claude --version` still works.

- [ ] **Step 3: (Optional) Uninstall the global npm package**

```bash
ssh agent@46.202.159.171 'bash -lc "sudo npm uninstall -g @anthropic-ai/claude-code 2>&1 || true"'
```

This is only needed if `npm root -g` in Task 1 showed a global install. Harmless if not present.

- [ ] **Step 4: Final verification**

Re-run Task 1 Step 3 and confirm Claude still works from the single remaining binary.

---

## Task 3: Create the Chief of Staff directory layout on the VPS

**Files (on VPS):**
- Create: `/home/agent/bin/` (dir)
- Create: `/home/agent/chief-of-staff/` (dir)
- Create: `/home/agent/logs/` (dir)

- [ ] **Step 1: Create directories**

```bash
ssh agent@46.202.159.171 'bash -lc "
mkdir -p /home/agent/bin /home/agent/chief-of-staff /home/agent/logs
ls -la /home/agent/
"'
```

Expected: All three directories exist and are owned by `agent:agent`.

- [ ] **Step 2: Verify `jq` and `curl` are installed** (cos-turn.sh needs them)

```bash
ssh agent@46.202.159.171 'bash -lc "which jq curl && jq --version && curl --version | head -1"'
```

If `jq` is missing: `sudo apt-get install -y jq`.

---

## Task 4: Write the Chief of Staff system prompt (CLAUDE.md)

**Files:**
- Create: `/home/agent/chief-of-staff/CLAUDE.md`

- [ ] **Step 1: Write the file via SSH heredoc**

```bash
ssh agent@46.202.159.171 'bash -lc "cat > /home/agent/chief-of-staff/CLAUDE.md" <<'"'"'COSPROMPT'"'"'
# Chief of Staff — System Prompt

You are Usman Ramay'"'"'s Chief of Staff, running on a Hostinger VPS. You receive messages from Usman via WhatsApp (routed through n8n) and respond via an outbound webhook that posts back to WhatsApp. Your job: receive requests, execute them autonomously, report back concisely.

## Communication style

- Keep replies under ~2000 characters. WhatsApp caps messages at 4096 characters, but long walls of text are hard to read on a phone.
- Never dump raw command output, terminal logs, or stack traces. Summarize.
- Write detail to `/home/agent/chief-of-staff/dashboard.md` — that is your scratchpad and task tracker. If Usman asks "what are you working on?" you can consult it.
- If you are unsure about intent, ask Usman instead of guessing. One clarifying question now saves ten wasted turns later.

## Sub-agents

- For any non-trivial or parallelizable task, use your built-in Task tool to spawn sub-agents.
- If two parts of a task can be researched or executed independently, parallelize them.
- Sub-agents live and die inside your turn. You synthesize their outputs into one reply.

## dashboard.md discipline

On every turn that touches a task, update `/home/agent/chief-of-staff/dashboard.md`. The file has four sections:

- `## Active` — tasks currently in progress
- `## Paused` — tasks blocked waiting on Usman or an external event
- `## Done` — recently completed tasks (prune old entries when the list grows long)
- `## Open Questions` — questions for Usman that are still unanswered

Move items between sections as state changes. The dashboard is the source of truth about what is happening across sessions.

## `/done` handling

When Usman sends `/done` (optionally with extra text like `/done thanks`), his session is ending. Before replying, do the following:

1. Update `dashboard.md`: mark truly finished tasks as Done; move in-progress items to Paused with a short status note.
2. If anything important was learned during the session (a user preference, a fact worth remembering, a decision), save it to the second brain via the `add_note` MCP tool. Mention what you saved in your reply.
3. Return a concise summary of the session: what was done, what is paused, anything Usman should know.

## Second brain writes

Never write to the second brain speculatively. When you have something worth saving, include the proposed content in your reply and only write it after Usman confirms — except during `/done`, where you may save clearly-useful learnings, mentioning in your reply what you saved.

## Self-review before replying

Before sending a reply, check:
- Did I actually do what Usman asked?
- Is my reply under ~2000 characters?
- Did I update `dashboard.md` if a task changed state?
- If something failed, am I reporting it clearly (what I tried, what went wrong, what recovery I tried)?

## Destructive operations — forbidden

- Never `rm -rf`, never modify `/etc`, never touch system files outside your own workdir.
- Never force-push to any shared repo.
- Never delete anything from the second brain unless Usman explicitly says so.
- The VPS has a weekly snapshot — that is the safety net, not a license to be reckless.

## Failure reporting

If something fails, report in this shape:
- What you attempted
- What went wrong (specific error, not a summary)
- What recovery you tried, if any

## Authentication

Your OAuth token is loaded from `/home/agent/.profile` at shell start. You do not need to manage it.

## Directory layout

- `/home/agent/chief-of-staff/` — your workdir (cwd on every invocation)
- `/home/agent/chief-of-staff/CLAUDE.md` — this file
- `/home/agent/chief-of-staff/dashboard.md` — your task tracker
- `/home/agent/.cos-session` — your current session ID (managed by cos-turn.sh — do not edit directly)
- `/home/agent/logs/` — per-day turn logs
COSPROMPT'
```

- [ ] **Step 2: Verify file contents**

```bash
ssh agent@46.202.159.171 'bash -lc "wc -l /home/agent/chief-of-staff/CLAUDE.md && head -20 /home/agent/chief-of-staff/CLAUDE.md"'
```

Expected: ~70–90 lines, starts with `# Chief of Staff — System Prompt`.

---

## Task 5: Initialize dashboard.md

**Files:**
- Create: `/home/agent/chief-of-staff/dashboard.md`

- [ ] **Step 1: Write empty dashboard template**

```bash
ssh agent@46.202.159.171 'bash -lc "cat > /home/agent/chief-of-staff/dashboard.md" <<'"'"'DASH'"'"'
# Chief of Staff Dashboard

_Maintained by the Chief of Staff agent. Tracks task state across sessions._

## Active

_(none)_

## Paused

_(none)_

## Done

_(none)_

## Open Questions

_(none)_
DASH'
```

- [ ] **Step 2: Verify**

```bash
ssh agent@46.202.159.171 'cat /home/agent/chief-of-staff/dashboard.md'
```

Expected: Four section headers, all `_(none)_`.

---

## Task 6: Write cos-turn.sh

**Files:**
- Create: `/home/agent/bin/cos-turn.sh`

The script does: acquire flock, decode base64 message, detect `/done`, decide resume vs. new session (mtime < 2 days), run Claude in JSON output mode with `--dangerously-skip-permissions`, parse `.result` + `.session_id`, persist session ID, POST reply to outbound webhook, release lock.

The outbound webhook URL is populated at this stage with a placeholder (`https://REPLACE_ME/cos-outbound`). Task 9 replaces it with the real URL after Task 8 creates the outbound workflow.

- [ ] **Step 1: Write cos-turn.sh**

```bash
ssh agent@46.202.159.171 'bash -lc "cat > /home/agent/bin/cos-turn.sh" <<'"'"'COSSH'"'"'
#!/usr/bin/env bash
# cos-turn.sh — Chief of Staff turn wrapper.
# Invoked by n8n via SSH. Receives one WhatsApp message (base64-encoded on the
# wire to avoid shell-quoting issues), runs a single Claude Code turn against
# the current CoS session, and POSTs the reply to the n8n outbound webhook.
#
# Usage:
#   cos-turn.sh <base64-encoded-message>

set -uo pipefail

# Load OAuth token and PATH from agent profile
# shellcheck disable=SC1091
source /home/agent/.profile

WORKDIR="/home/agent/chief-of-staff"
SESSION_FILE="/home/agent/.cos-session"
LOCK_FILE="/home/agent/.cos-session.lock"
LOG_DIR="/home/agent/logs"
OUTBOUND_URL="https://REPLACE_ME/cos-outbound"
SESSION_MAX_AGE_DAYS=2

mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/cos-turn-$(date +%Y-%m-%d).log"

log() {
  echo "[$(date -Iseconds)] $*" >> "$LOG_FILE"
}

post_reply() {
  local reply="$1"
  local payload
  payload=$(jq -n --arg r "$reply" '"'"'{reply: $r}'"'"')
  curl -s -S -X POST "$OUTBOUND_URL" \
    -H "Content-Type: application/json" \
    --data-binary "$payload" >> "$LOG_FILE" 2>&1 \
    || log "outbound webhook POST failed"
}

# --- Lock ---------------------------------------------------------------
exec 9>"$LOCK_FILE"
flock 9

# --- Decode message ----------------------------------------------------
if [[ $# -lt 1 ]]; then
  log "missing message arg"
  post_reply "Internal error: no message received."
  exit 1
fi

if ! MESSAGE=$(echo "$1" | base64 -d 2>/dev/null); then
  log "invalid base64 input"
  post_reply "Internal error: could not decode message."
  exit 1
fi

log "received: $(echo "$MESSAGE" | head -c 200)"

# --- /done detection ---------------------------------------------------
DONE=0
if [[ "$MESSAGE" == /done* ]]; then
  DONE=1
  MESSAGE="$MESSAGE

(SYSTEM NOTE: Usman has signaled /done — the session is ending. Perform the closing work described in the /done handling section of your system prompt: update dashboard.md, save any important learnings to the second brain (mention what you saved in your reply), and return a concise session summary.)"
fi

# --- Decide resume vs new ---------------------------------------------
RESUME_ARGS=()
if [[ -f "$SESSION_FILE" ]]; then
  AGE_SECONDS=$(( $(date +%s) - $(stat -c %Y "$SESSION_FILE") ))
  MAX_AGE_SECONDS=$(( SESSION_MAX_AGE_DAYS * 86400 ))
  if (( AGE_SECONDS < MAX_AGE_SECONDS )); then
    SESSION_ID=$(cat "$SESSION_FILE")
    if [[ -n "$SESSION_ID" ]]; then
      RESUME_ARGS=(--resume "$SESSION_ID")
      log "resuming session $SESSION_ID (age ${AGE_SECONDS}s)"
    fi
  else
    log "session file older than ${SESSION_MAX_AGE_DAYS}d; starting fresh"
    rm -f "$SESSION_FILE"
  fi
fi

# --- Run Claude -------------------------------------------------------
cd "$WORKDIR"
CLAUDE_STDERR=$(mktemp)
set +e
CLAUDE_OUTPUT=$(claude \
  "${RESUME_ARGS[@]}" \
  -p "$MESSAGE" \
  --output-format json \
  --dangerously-skip-permissions 2>"$CLAUDE_STDERR")
CLAUDE_EXIT=$?
set -e

if (( CLAUDE_EXIT != 0 )); then
  log "claude exited $CLAUDE_EXIT"
  STDERR_TAIL=$(tail -n 15 "$CLAUDE_STDERR" | tr '"'"'\n'"'"' '"'"' '"'"')
  rm -f "$CLAUDE_STDERR"
  post_reply "Claude failed (exit $CLAUDE_EXIT). Last stderr: $STDERR_TAIL"
  exit 0
fi
rm -f "$CLAUDE_STDERR"

# --- Parse output -----------------------------------------------------
REPLY=$(echo "$CLAUDE_OUTPUT" | jq -r '"'"'.result // empty'"'"')
NEW_SESSION_ID=$(echo "$CLAUDE_OUTPUT" | jq -r '"'"'.session_id // empty'"'"')

if [[ -z "$REPLY" ]]; then
  log "empty reply from claude; raw output: $(echo "$CLAUDE_OUTPUT" | head -c 500)"
  post_reply "Claude returned no reply. Check VPS logs at $LOG_FILE."
  exit 0
fi

# --- Persist session ID -----------------------------------------------
if [[ -n "$NEW_SESSION_ID" ]]; then
  echo -n "$NEW_SESSION_ID" > "$SESSION_FILE"
  log "session id: $NEW_SESSION_ID"
fi

# --- Clear on /done ---------------------------------------------------
if (( DONE == 1 )); then
  rm -f "$SESSION_FILE"
  log "/done — session cleared"
fi

# --- Send reply -------------------------------------------------------
post_reply "$REPLY"
log "reply sent ($(echo -n "$REPLY" | wc -c) bytes)"
COSSH
chmod +x /home/agent/bin/cos-turn.sh'
```

- [ ] **Step 2: Verify executable**

```bash
ssh agent@46.202.159.171 'bash -lc "ls -l /home/agent/bin/cos-turn.sh && bash -n /home/agent/bin/cos-turn.sh && echo SYNTAX_OK"'
```

Expected: `-rwxr-xr-x`, `SYNTAX_OK`.

---

## Task 7: Smoke-test cos-turn.sh with a throwaway outbound URL

This verifies Claude runs and the script's internal plumbing works — before we hook it up to n8n.

- [ ] **Step 1: Temporarily swap the outbound URL to webhook.site**

Go to https://webhook.site in a browser, copy the "Your unique URL." Then:

```bash
ssh agent@46.202.159.171 "sed -i 's|https://REPLACE_ME/cos-outbound|<THE_WEBHOOK_SITE_URL>|' /home/agent/bin/cos-turn.sh"
```

Replace `<THE_WEBHOOK_SITE_URL>` with the real URL, e.g. `https://webhook.site/abc-123-def`.

- [ ] **Step 2: Invoke cos-turn.sh with a test message**

```bash
ssh agent@46.202.159.171 'bash -lc "MSG=$(echo -n \"Hello chief of staff. Just reply with the single word OK.\" | base64 -w 0) && /home/agent/bin/cos-turn.sh \"$MSG\""'
```

Expected:
- Command returns with exit 0.
- Within ~10s, a POST request appears at the webhook.site URL containing `{"reply": "OK"}` (or similar short reply).
- `/home/agent/.cos-session` now contains a session ID.
- `/home/agent/logs/cos-turn-<today>.log` has entries for received, session id, reply sent.

- [ ] **Step 3: Verify session continuity with a follow-up**

```bash
ssh agent@46.202.159.171 'bash -lc "MSG=$(echo -n \"What did I just ask you to reply with? Only say the one word.\" | base64 -w 0) && /home/agent/bin/cos-turn.sh \"$MSG\""'
```

Expected: webhook.site receives `{"reply": "OK"}` (Claude remembers from the resumed session).

- [ ] **Step 4: Revert the placeholder URL**

```bash
ssh agent@46.202.159.171 "sed -i 's|<THE_WEBHOOK_SITE_URL>|https://REPLACE_ME/cos-outbound|' /home/agent/bin/cos-turn.sh"
```

(Task 9 replaces it with the real n8n URL.)

---

## Task 8: Build the `CoS: Outbound` n8n workflow

**Files (n8n):**
- Create new workflow: `CoS: Outbound`

- [ ] **Step 1: Look up node schemas**

```
mcp__claude_ai_n8n__search_nodes({ query: "webhook" })
mcp__claude_ai_n8n__search_nodes({ query: "whatsapp send message" })
mcp__claude_ai_n8n__get_node_types({ types: [
  { name: "n8n-nodes-base.webhook", version: 2 },
  { name: "n8n-nodes-base.whatsApp", version: 1 }
]})
```

Note the exact parameter schemas for both nodes.

- [ ] **Step 2: Create the workflow via SDK code**

```
mcp__claude_ai_n8n__create_workflow_from_code({
  name: "CoS: Outbound",
  code: `
import { createWorkflow } from '@n8n/sdk';

const workflow = createWorkflow({ name: 'CoS: Outbound' });

const hook = workflow.addNode({
  type: 'n8n-nodes-base.webhook',
  typeVersion: 2,
  name: 'Webhook',
  parameters: {
    httpMethod: 'POST',
    path: 'cos-outbound',
    responseMode: 'onReceived',
    options: {}
  }
});

const wa = workflow.addNode({
  type: 'n8n-nodes-base.whatsApp',
  typeVersion: 1,
  name: 'Send WhatsApp',
  parameters: {
    resource: 'message',
    operation: 'send',
    phoneNumberId: '1020770907789933',
    recipientPhoneNumber: '352621486096',
    textBody: '={{ $json.reply }}',
    additionalFields: {}
  },
  credentials: { whatsAppApi: { id: 'kOBiI6xLEZFkr28v', name: 'Send WhatsApp Personal' } }
});

workflow.connect(hook, wa);
workflow.setSettings({ availableInMCP: true });

export default workflow;
  `
})
```

- [ ] **Step 3: Validate + publish**

```
mcp__claude_ai_n8n__validate_workflow({ workflowId: <returned id> })
mcp__claude_ai_n8n__publish_workflow({ workflowId: <returned id> })
```

Expected: validation passes; publish succeeds.

- [ ] **Step 4: Retrieve the webhook URL**

```
mcp__claude_ai_n8n__get_workflow_details({ workflowId: <id>, includeWebhookUrls: true })
```

Note the full webhook URL — e.g. `https://n8n.srv1016866.hstgr.cloud/webhook/cos-outbound`. **Save this URL.** You will plug it into `cos-turn.sh` in Task 9.

---

## Task 9: Wire cos-turn.sh to the real outbound webhook URL

**Files:**
- Modify: `/home/agent/bin/cos-turn.sh` (one line: `OUTBOUND_URL`)

- [ ] **Step 1: Replace the placeholder**

```bash
REAL_URL="<outbound webhook URL from Task 8 Step 4>"
ssh agent@46.202.159.171 "sed -i 's|https://REPLACE_ME/cos-outbound|${REAL_URL}|' /home/agent/bin/cos-turn.sh"
```

- [ ] **Step 2: Verify the replacement**

```bash
ssh agent@46.202.159.171 'grep OUTBOUND_URL /home/agent/bin/cos-turn.sh'
```

Expected: line shows the real URL, not `REPLACE_ME`.

---

## Task 10: End-to-end smoke test (cos-turn.sh → WhatsApp)

Before we hook up the inbound side, confirm Claude → outbound → WhatsApp works.

- [ ] **Step 1: Start fresh (clear any prior test session)**

```bash
ssh agent@46.202.159.171 'rm -f /home/agent/.cos-session'
```

- [ ] **Step 2: Invoke cos-turn.sh with a short test message**

```bash
ssh agent@46.202.159.171 'bash -lc "MSG=$(echo -n \"This is a connectivity test. Reply with: I am online.\" | base64 -w 0) && /home/agent/bin/cos-turn.sh \"$MSG\""'
```

- [ ] **Step 3: Verify a WhatsApp message arrives on Usman's phone**

Expected: within ~15s, a WhatsApp message containing "I am online" (or similar) from the Chief of Staff WhatsApp number.

If no message arrives: check `/home/agent/logs/cos-turn-<today>.log` for the POST result; check n8n executions for `CoS: Outbound`.

---

## Task 11: Build the `CoS: Inbound` n8n workflow

**Files (n8n):**
- Create new workflow: `CoS: Inbound`

- [ ] **Step 1: Look up node schemas**

```
mcp__claude_ai_n8n__search_nodes({ query: "whatsapp trigger" })
mcp__claude_ai_n8n__search_nodes({ query: "ssh" })
mcp__claude_ai_n8n__search_nodes({ query: "filter" })
mcp__claude_ai_n8n__get_node_types({ types: [
  { name: "n8n-nodes-base.whatsAppTrigger", version: 1 },
  { name: "n8n-nodes-base.if", version: 2.2 },
  { name: "n8n-nodes-base.ssh", version: 3 }
]})
```

Confirm exact parameter names (especially for the WhatsApp Trigger node and the SSH node).

- [ ] **Step 2: Create the workflow**

Shape: WhatsApp Trigger → IF (filter to Usman's number) → SSH (dispatch cos-turn.sh in background).

```
mcp__claude_ai_n8n__create_workflow_from_code({
  name: "CoS: Inbound",
  code: `
import { createWorkflow } from '@n8n/sdk';

const workflow = createWorkflow({ name: 'CoS: Inbound' });

const trigger = workflow.addNode({
  type: 'n8n-nodes-base.whatsAppTrigger',
  typeVersion: 1,
  name: 'WhatsApp Trigger',
  parameters: {
    updates: ['messages']
  },
  credentials: { whatsAppTriggerApi: { id: 'kOBiI6xLEZFkr28v', name: 'Send WhatsApp Personal' } }
});

const filter = workflow.addNode({
  type: 'n8n-nodes-base.if',
  typeVersion: 2.2,
  name: 'Only from Usman',
  parameters: {
    conditions: {
      options: { caseSensitive: true, leftValue: '', typeValidation: 'loose' },
      conditions: [{
        leftValue: '={{ $json.messages[0].from }}',
        rightValue: '352621486096',
        operator: { type: 'string', operation: 'equals' }
      }],
      combinator: 'and'
    },
    options: {}
  }
});

const ssh = workflow.addNode({
  type: 'n8n-nodes-base.ssh',
  typeVersion: 3,
  name: 'Dispatch cos-turn.sh',
  parameters: {
    resource: 'command',
    operation: 'execute',
    command: '=MSG_B64=$(printf "%s" "{{ $json.messages[0].text.body.replaceAll(\\'"\\', \\'\\\\"\\') }}" | base64 -w 0); nohup /home/agent/bin/cos-turn.sh "$MSG_B64" >> /home/agent/logs/cos-turn-$(date +%Y-%m-%d).log 2>&1 &',
    cwd: '/home/agent'
  },
  credentials: { sshPassword: { id: '<SSH_CREDENTIAL_ID>', name: "Sophia's VPS SSH" } }
});

workflow.connect(trigger, filter);
workflow.connect(filter, ssh, { outputIndex: 0 }); // only "true" branch
workflow.setSettings({ availableInMCP: true });

export default workflow;
  `
})
```

**Important:**
- The SSH credential ID (`<SSH_CREDENTIAL_ID>`) is the ID of `Sophia's VPS SSH`. Look it up via `mcp__claude_ai_n8n__search_workflows` on an existing CoS workflow if not remembered — it should be the same one used in `Task Intake` (`3wJttx3ZflsYmPGz`).
- The `replaceAll` call escapes quotes in the WhatsApp message body before base64 encoding. This is the injection-safe path from the spec's open decisions.
- Connect the filter's **true branch only** (`outputIndex: 0`) to the SSH node. Messages from other numbers hit the false branch and die silently.

- [ ] **Step 3: Validate + publish**

```
mcp__claude_ai_n8n__validate_workflow({ workflowId: <returned id> })
mcp__claude_ai_n8n__publish_workflow({ workflowId: <returned id> })
```

- [ ] **Step 4: Confirm the WhatsApp Trigger is now the active receiver**

Meta's WhatsApp Business webhook points at n8n's aggregated webhook endpoint. When the new trigger is active with the `Send WhatsApp Personal` credential, it should start receiving inbound messages.

Verify by sending a test message (next task).

---

## Task 12: End-to-end test — real WhatsApp message round-trip

This is the full flow: you send a WhatsApp message from your phone, and the reply comes back.

- [ ] **Step 1: Clear any prior test session**

```bash
ssh agent@46.202.159.171 'rm -f /home/agent/.cos-session'
```

- [ ] **Step 2: Send a WhatsApp message**

From Usman's phone, send to the Chief of Staff WhatsApp number: `Hello, are you online?`

- [ ] **Step 3: Verify inbound execution**

Within ~5s, `CoS: Inbound` should show a new execution. Verify via:

```
mcp__claude_ai_n8n__search_workflows({ query: "CoS: Inbound" }) // get id
// Then check recent executions in n8n UI, or via get_execution with the most recent id
```

- [ ] **Step 4: Verify reply arrives via WhatsApp**

Within ~30s total, a WhatsApp reply should arrive (something like "Yes, I'm online and ready.").

If no reply:
- Check `/home/agent/logs/cos-turn-<today>.log` for errors.
- Check `CoS: Inbound` execution for SSH errors.
- Check `CoS: Outbound` execution to confirm the callback was received.

---

## Task 13: Test session continuity

- [ ] **Step 1: Send a first message that establishes context**

WhatsApp: `Remember this codeword for a moment: PURPLE ELEPHANT. Reply with just 'noted'.`

Wait for reply.

- [ ] **Step 2: Send a second message that relies on memory**

WhatsApp: `What was the codeword?`

Expected: reply includes `PURPLE ELEPHANT`. Proves `--resume` is working and the session persists.

- [ ] **Step 3: Verify only one session file exists**

```bash
ssh agent@46.202.159.171 'cat /home/agent/.cos-session && echo && ls -la /home/agent/.cos-session'
```

Expected: a single session ID, modified recently.

---

## Task 14: Test `/done` behaviour

- [ ] **Step 1: Send `/done` over WhatsApp**

Message: `/done thanks, wrap it up`

Expected: reply is a session summary. Claude mentions what it saved (if anything) and summarises the session.

- [ ] **Step 2: Verify session was cleared**

```bash
ssh agent@46.202.159.171 'ls -la /home/agent/.cos-session 2>&1'
```

Expected: `No such file or directory`.

- [ ] **Step 3: Send a follow-up message**

WhatsApp: `What was the codeword?`

Expected: Claude does NOT know the codeword — this is a fresh session. The reply should indicate it has no memory of a previous codeword.

---

## Task 15: Test concurrency (lock)

- [ ] **Step 1: Kick off a deliberately slow turn, then fire a quick follow-up**

WhatsApp message 1: `Spend 30 seconds doing nothing, then reply with the word FIRST.`

Immediately (within a few seconds) send WhatsApp message 2: `Now reply with the word SECOND.`

Expected:
- First reply: `FIRST` (after ~30s).
- Second reply: `SECOND` (after the first finishes — queued, not parallel).
- Order is preserved.

- [ ] **Step 2: Inspect the log**

```bash
ssh agent@46.202.159.171 'tail -40 /home/agent/logs/cos-turn-$(date +%Y-%m-%d).log'
```

Expected: you can see the two turns serialized, not overlapping.

---

## Task 16: Test failure surfacing

- [ ] **Step 1: Temporarily break the session ID to force an error**

```bash
ssh agent@46.202.159.171 'echo -n "nonexistent-session-id" > /home/agent/.cos-session'
```

- [ ] **Step 2: Send a WhatsApp message**

WhatsApp: `Are you there?`

Expected: within ~30s, a WhatsApp reply with a plain-language error mentioning "Claude failed" and a stderr tail.

- [ ] **Step 3: Recover**

```bash
ssh agent@46.202.159.171 'rm -f /home/agent/.cos-session'
```

Send another WhatsApp message. A new session should start cleanly and reply normally.

---

## Task 17: Update auto-memory with final state

**Files:**
- Modify: `/Users/usmanramay/.claude/projects/-Users-usmanramay-Documents-Projects-System-Automation/memory/project_chief_of_staff_agent.md`
- Modify: `/Users/usmanramay/.claude/projects/-Users-usmanramay-Documents-Projects-System-Automation/memory/MEMORY.md` (only if a new memory file is added)

- [ ] **Step 1: Append a dated update to the existing Chief of Staff memory**

Add a section to `project_chief_of_staff_agent.md`:

```markdown
## 2026-04-18: Minimal n8n redesign shipped

New architecture replaces the three-agent orchestration with a single long-lived Claude Code CLI session:
- Two new n8n workflows: `CoS: Inbound` (id: <fill in>) and `CoS: Outbound` (id: <fill in>)
- VPS wrapper: `/home/agent/bin/cos-turn.sh`
- System prompt: `/home/agent/chief-of-staff/CLAUDE.md`
- Dashboard: `/home/agent/chief-of-staff/dashboard.md`
- Session state: `/home/agent/.cos-session` (cleared on /done or >2d inactivity)
- Outbound webhook: `https://n8n.srv1016866.hstgr.cloud/webhook/cos-outbound`

Old workflows (Task Intake, Planner, Executor, Reviewer, Delivery, Retrospective, Weekly Review, Email Watcher, n8n Backup) kept untouched for the 1-week validation window. Delete after confirmed reliable.

Slash commands: `/done` is the only one intercepted by cos-turn.sh. Others pass through verbatim.
```

- [ ] **Step 2: Commit no-op if repo isn't git-tracked**

No git repo here, so no commit. Memory files are read on every future Claude session start.

---

## Self-review checklist

Confirm before handoff:

- [ ] Every spec section has at least one task implementing it (Architecture → Tasks 3–11; Operational rules → built into cos-turn.sh in Task 6; Success criteria → Tasks 12–16).
- [ ] No "TBD" or "fill in details" placeholders (the one `<SSH_CREDENTIAL_ID>` in Task 11 Step 2 is explicitly noted as a lookup the engineer does).
- [ ] Every task includes exact commands and expected outputs.
- [ ] Shell-injection risk from spec open decisions is addressed (base64 encoding in the SSH node's command template, base64 decode in cos-turn.sh).
- [ ] The 2-day session retirement is reflected in `cos-turn.sh` (`SESSION_MAX_AGE_DAYS=2`).
- [ ] `/done` behaviour matches spec: intercepted in shell, augmented with wrap-up directive, session cleared on success, the actual wrap-up *work* is described in `CLAUDE.md` and not hardcoded in shell.
- [ ] No existing CoS workflows are deleted.

## Risks / things to watch during implementation

- **Claude CLI JSON schema drift.** `.result` and `.session_id` are assumed stable. If a newer version shifts these, Task 7 smoke test will catch it. Fall back: `--output-format stream-json` or plain text with session capture via a different flag.
- **WhatsApp trigger payload shape.** The `$json.messages[0].text.body` and `$json.messages[0].from` paths assume the standard inbound webhook shape from Meta. Verify in Task 11 or via a first test message.
- **Meta webhook routing.** If inbound messages do not arrive at the new trigger after publish, verify (a) the trigger is active, (b) the credential is correct, (c) n8n's WhatsApp trigger infrastructure is dispatching to active workflows.
- **Concurrency and the outbound webhook.** If two turns finish back-to-back, the outbound workflow handles them sequentially; verify both WhatsApp messages arrive in order in Task 15.
- **OAuth token expiry.** Token is ~1 year valid. Failure surfacing (Task 16) will catch this when it happens.
