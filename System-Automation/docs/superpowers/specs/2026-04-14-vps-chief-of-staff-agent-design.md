# VPS Chief of Staff Agent — Design Spec

**Date:** 2026-04-14
**Status:** Draft
**Author:** Usman Ramay + Claude

## Problem

AI conversations are transactional — you ask, you get an answer, you do the work yourself. For tasks that require research, browsing, comparison, and iteration (finding flights, researching purchases, setting up services), the human is still doing most of the legwork. The goal is a system that can receive a task, execute it autonomously with high quality, and return a finished result — taking minimal time from the user.

## Solution Overview

A "Chief of Staff" AI agent running on the existing Hostinger VPS, orchestrated by n8n, powered by Claude Code CLI with browser automation via Patchright. The system uses a three-agent architecture (Planner → Executor → Reviewer) to ensure output quality without requiring user involvement in iteration loops.

**Core principle:** The system should be self-extending. Given Claude Code with sudo access, a browser, and an email identity, it can set up whatever else it needs — new services, new tools, hosted web UIs — without the user having to configure anything manually.

## Architecture

### Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **n8n** | VPS (existing) | I/O layer: WhatsApp, webhooks, email, task logging (Supabase `memories` table with category `log`), orchestration |
| **Claude Code CLI** | VPS (`/home/agent/`) | AI brain: reasoning, planning, tool use, sub-agents |
| **Patchright** | VPS (`/home/agent/`) | Browser automation: headless Chromium with anti-detection patches |
| **Email** | Hostinger email hosting | `sophia@ptriconsulting.com` — agent's identity for sign-ups, verification |
| **Chrome profile** | VPS (`/home/agent/browser-data/`) | Persistent browser state: cookies, saved logins |
| **Second brain** | Supabase (existing) | Long-term memory: user preferences, knowledge, past task learnings |
| **GitHub repos** | GitHub (existing) | Project files, code, markdown docs the agent may need for context |

### VPS Resource Budget (4 GB RAM, 1 CPU)

| Process | RAM estimate |
|---------|-------------|
| OS + overhead | ~500 MB |
| n8n | ~500 MB |
| Claude Code invocation | ~300 MB |
| Patchright (one browser tab) | ~300 MB |
| **Headroom** | ~2.4 GB |

Tasks run sequentially — one at a time. This is sufficient for personal use.

### User Identity (Linux)

- **Dedicated user:** `agent` (not root)
- **Sudo access:** Yes — needed for installing packages, configuring services, setting up hosted applications
- **Home directory:** `/home/agent/`
- **Rationale:** Separation is for hygiene and auditability, not restriction. VPS snapshots provide the safety net.

## Task Flow

### Entry Points

| Channel | How it reaches n8n |
|---------|-------------------|
| **WhatsApp** | Existing WhatsApp integration in n8n |
| **Webhook** | n8n webhook endpoint (supports Siri Shortcuts, HTTP calls, etc.) |

### End-to-End Flow

```
User sends task (WhatsApp or webhook)
         │
         ▼
┌─────────────────┐
│   n8n: INTAKE    │
│                  │
│ - Log task       │
│ - Pull context   │
│   from Supabase  │
│ - Assign task ID │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ PHASE 1: PLAN   │
│ (Claude Code #1) │
│                  │
│ - Analyze task   │
│ - Define success │
│   criteria       │
│ - Return criteria│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ n8n: ALIGNMENT  │
│                  │
│ - Send criteria  │
│   to user via    │
│   WhatsApp       │
│ - Wait for       │
│   confirmation   │
│ - Save confirmed │
│   criteria       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ PHASE 2: EXECUTE │
│ (Claude Code #2) │
│                  │
│ - Receives: task │
│   + criteria     │
│ - Breaks into    │
│   sub-tasks      │
│ - Spawns sub-    │
│   agents         │
│ - Uses browser   │
│   (Patchright)   │
│ - Self-reviews   │
│   against        │
│   criteria       │
│ - Iterates       │
│   internally     │
│ - Returns result │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ PHASE 3: REVIEW  │
│ (Claude Code #3) │
│                  │
│ - Receives ONLY: │
│   task + criteria│
│   + output       │
│ - Does NOT see   │
│   execution      │
│   reasoning      │
│ - Reviews each   │
│   criterion:     │
│   pass/fail with │
│   evidence       │
└────────┬────────┘
         │
    ┌────┴────┐
    │  Pass?  │
    └────┬────┘
     yes │    no (max 2 retries)
         │         │
         │         ▼
         │    ┌──────────┐
         │    │ n8n sends │
         │    │ feedback  │
         │    │ to new    │
         │    │ Executor  │
         │    │ invocation│
         │    └──────────┘
         │         │
         │         └──► (back to PHASE 2)
         ▼
┌─────────────────┐
│ n8n: DELIVERY   │
│                  │
│ - Store learnings│
│   to Supabase    │
│ - Send result    │
│   via WhatsApp   │
│ - Update task    │
│   log: "done"    │
└─────────────────┘
```

### Mid-Task Questions

If the Executor needs user input during execution:

1. Executor returns a structured response with `status: "question"`, the question text, and a `progress` field summarizing work completed so far
2. n8n saves the progress summary and sends the question to the user via WhatsApp
3. n8n waits for the user's reply (webhook listener)
4. n8n resumes by calling a new Executor invocation with: original task + criteria + saved progress summary + user's answer

The progress summary is plain text (not code or intermediate state) — it tells the new invocation what was already done so it doesn't repeat work.

### Failure Handling

Every failure — at any stage — produces a structured error with:
- **What** was being attempted
- **What** went wrong (specific error, not a summary)
- **What** recovery was tried and why it also failed

n8n sends failure notifications via WhatsApp:
```
Task failed: [task description]
Step: [which step failed]
Error: [specific error message]
Attempted: [recovery actions taken]
Task ID: [reference]
```

Claude Code's structured output format:
```json
{
  "status": "failed",
  "summary": "Could not complete Amazon product research",
  "completed_steps": ["Opened Amazon", "Searched for product"],
  "failed_step": "Loading product page B0xyz",
  "error": "HTTP 503 - blocked by Amazon anti-bot after 3 retries",
  "recovery_attempted": "Cleared cookies, tried alternative URL format",
  "learnings": "Amazon blocks this VPS IP for product pages"
}
```

## Claude Code Configuration

### Invocation Pattern

```bash
claude -p "<prompt>" \
  --output-format json \
  --dangerously-skip-permissions \
  --max-budget-usd 1.00 \
  --system-prompt-file /home/agent/prompts/executor.md \
  --model sonnet
```

- `--dangerously-skip-permissions`: Full autonomy — no permission prompts
- `--max-budget-usd 1.00`: Cost cap per invocation (adjustable based on experience)
- `--output-format json`: Structured output for n8n to parse
- `--system-prompt-file /home/agent/prompts/<role>.md`: Role-specific prompt per agent
- `--json-schema '<schema>'`: Validates structured output format

### Authentication

- OAuth token generated via `claude setup-token` on Usman's Mac
- Stored as environment variable on VPS: `CLAUDE_CODE_OAUTH_TOKEN`
- Valid for ~1 year; renewal requires re-running `setup-token` on a machine with a browser

### System Prompt Structure

Each of the three agents (Planner, Executor, Reviewer) gets a different system prompt. Common elements:

- Agent identity and role
- Available tools and how to use them
- Error reporting requirements (what/what/what format)
- Output format (structured JSON)

**Planner prompt** includes:
- Instructions to define clear, measurable success criteria
- Format for criteria output

**Executor prompt** includes:
- Task description and confirmed success criteria
- Instructions to break task into sub-tasks and use sub-agents
- Instructions to self-review against criteria before returning
- Browser tool documentation (Patchright scripts)
- How to request user input (return `status: "question"`)

**Reviewer prompt** includes:
- Only: original task, success criteria, executor's output
- Instructions to evaluate each criterion independently with evidence
- Pass/fail determination with specific feedback if fail
- No access to execution process or reasoning

## Browser Automation (Patchright)

### Setup

- Patchright installed via npm: `npm install patchright`
- Headless Chromium bundled (no separate browser install)
- Persistent profile at `/home/agent/browser-data/`

### Pre-built Scripts

```
/home/agent/scripts/
  ├── browse.js        # Open URL, return page content as markdown
  ├── interact.js      # Navigate, click, fill forms, take screenshots
  ├── screenshot.js    # Capture page as image
  └── session.js       # Manage login sessions (save/restore cookies)
```

These are convenience scripts for common patterns. Claude Code can also write custom Patchright scripts on the fly for unanticipated browser interactions.

### Email Verification Flow

```
Patchright submits sign-up form on a website
         │
         ▼
Patchright script starts fs.watch() on /home/agent/inbox/
         │                         (timeout: 3 minutes)
         │
    Meanwhile...
         │
Site sends verification email to sophia@ptriconsulting.com
         │
         ▼
n8n IMAP trigger detects new email
         │
         ▼
n8n extracts verification code from email body
         │
         ▼
n8n writes code to /home/agent/inbox/<timestamp>.txt
         │
         ▼
fs.watch() fires in Patchright script
         │
         ▼
Script reads code, enters it on the website, continues
```

If no code arrives within 3 minutes, the script returns a failure with the specific error: "Verification email not received within timeout for [site]."

### Anti-Bot Limitations

Patchright reduces headless detection signals but does not guarantee evasion on all sites. Sites with aggressive anti-bot (Amazon, social media, Cloudflare high-security) may block the VPS IP.

**Current approach:** If a site blocks the agent, it reports the failure clearly. No residential proxy is set up initially.

**Future upgrades (when needed):**
- Residential proxy service (DataImpulse at ~$1/GB or IPRoyal at ~$5/IP)
- Only route blocked sites through proxy, not all traffic

## Memory Architecture

### Three memory layers

| Layer | Storage | Scope | Examples |
|-------|---------|-------|---------|
| **User knowledge** | Supabase (second brain) | Persistent, shared across all AI clients | Preferences ("prefers Lufthansa"), research findings, decisions |
| **Project files** | GitHub | Persistent, versioned | Markdown docs, code, task specs, plans |
| **Agent operational state** | VPS filesystem (`/home/agent/memory/`) | Persistent, agent-specific | Website credentials, site-specific quirks, tool configs |

### How Context is Loaded

When n8n prepares a task for Claude Code:
1. Queries Supabase for relevant preferences and past task learnings (via search/get_note)
2. Includes this context in the Claude Code prompt
3. Claude Code can also query Supabase mid-task via MCP tools if it needs more context

For project-related tasks:
- Claude Code has GitHub access (SSH key or personal access token for `sophia@ptriconsulting.com`)
- Pulls relevant repos on demand (`git clone` / `git pull`)
- Works on files locally, pushes changes back if needed

### Learning Loop

After task completion, n8n stores new learnings:
- User preference discoveries → Supabase second brain (via `add_note`)
- Operational learnings (site quirks, tool configs) → `/home/agent/memory/`
- Task results and history → Supabase task log

## Email Identity

### Setup

- Address: `sophia@ptriconsulting.com`
- Hosted by: Hostinger email (included with hosting plan)
- Accessed by: n8n via IMAP trigger (for incoming verification codes and messages)
- Used for: website sign-ups, receiving verification codes, potential future email communication

The agent uses email+password sign-ups for websites (not Google OAuth), as this is more reliable for automated flows. Browser session persistence is handled by Patchright's persistent profile directory (`/home/agent/browser-data/`), which stores cookies and login state across sessions without needing a Google Account.

## Safety and Recovery

### Guardrails

| Guardrail | What it prevents |
|-----------|-----------------|
| `--max-budget-usd 1.00` | Runaway loops — Claude Code stops after 50 tool-use cycles |
| Three-agent separation | Executor self-preference bias — independent reviewer catches blind spots |
| Structured error reporting | Silent failures — every failure is explicit with what/what/what |
| WhatsApp notifications | Lack of visibility — user is informed of progress, questions, and failures |
| Sequential task execution | Resource exhaustion — one task at a time on 4 GB RAM |

### Backup Strategy

| What | How | Frequency |
|------|-----|-----------|
| **n8n workflows** | Automated n8n workflow exports all workflows to JSON, pushes to GitHub | Daily |
| **VPS snapshot** | Hostinger built-in VPS snapshot | Weekly |

### Recovery Scenarios

| Scenario | Recovery |
|----------|---------|
| Claude Code crashes mid-task | n8n detects timeout/error, notifies user, task can be retried |
| Agent corrupts its own files | VPS snapshot restore (agent directory only if possible, full VPS if needed) |
| n8n workflows corrupted | Restore from daily GitHub backup |
| VPS completely broken | Restore from Hostinger weekly snapshot, re-run setup |
| OAuth token expires | Re-run `claude setup-token` on Mac, copy new token to VPS |

**Estimated recovery time for worst case (full VPS restore):** Under half a day.

## File Structure on VPS

```
/home/agent/
  ├── scripts/              # Patchright browser scripts
  │   ├── browse.js
  │   ├── interact.js
  │   ├── screenshot.js
  │   └── session.js
  ├── workdir/              # Claude Code working directory per task
  ├── browser-data/         # Persistent Chrome profile
  ├── inbox/                # Verification codes from n8n
  ├── memory/               # Agent operational state
  ├── logs/                 # Task execution logs
  └── repos/                # Cloned GitHub repos
```

## Future Upgrades (Not in Initial Build)

These are deferred until the need arises:

| Upgrade | Trigger to add it |
|---------|------------------|
| Residential proxy | Specific sites consistently blocking VPS IP |
| VPS RAM upgrade | Hitting memory limits on complex tasks |
| Web UI dashboard | Wanting to review task history visually instead of WhatsApp |
| Phone call automation | Specific task requires it (via Twilio + Vapi/Retell) |
| Apify integration | Need for heavy batch scraping beyond what Patchright handles |

## Self-Improvement System

### Core Principle

The system tracks its own performance and uses that data to get better over time. The two goals, in priority order:

1. **Autonomy** — reduce how often it needs to ask the user for input
2. **Efficiency** — reduce wasted tokens (repetitive loops, dead-end approaches)

### What Gets Tracked

Every task execution logs a structured performance record to Supabase:

```json
{
  "task_id": "abc123",
  "task_summary": "Find best noise-cancelling headphones under 200 EUR",
  "timestamp": "2026-04-15T10:30:00Z",
  "outcome": "success | partial | failed",
  "turns_used": 34,
  "review_iterations": 1,
  "user_questions_asked": 0,
  "user_interventions": 0,
  "errors": [],
  "blocked_sites": [],
  "learnings": ["rtings.com is the best source for headphone comparisons"],
  "improvement_flags": []
}
```

### Improvement Flags

n8n automatically flags patterns that indicate the system should improve:

| Flag | Trigger | What it means |
|------|---------|--------------|
| `asked_user` | Executor returned `status: "question"` | System couldn't figure something out on its own |
| `review_failed` | Reviewer rejected executor output | Executor's self-review didn't catch a quality issue |
| `multiple_review_loops` | More than 1 review iteration | Executor struggled to meet criteria even with feedback |
| `high_cost` | Cost > $0.80 (80% of budget) | Approaching runaway territory — possibly going in circles |
| `repeated_error` | Same error type seen in 3+ recent tasks | Systemic issue not being addressed |
| `site_blocked` | Anti-bot detection blocked the agent | Specific site needs a different approach |
| `timeout` | Claude Code hit max-turns limit | Task was too complex or agent got stuck in a loop |

### How the System Learns

**Automatic (built into every task):**

After each task, n8n runs a lightweight "retrospective" — a short Claude Code invocation that receives:
- The performance record above
- The last 10 task performance records
- The current contents of `/home/agent/memory/system-learnings.md`

It answers three questions:
1. Did anything go wrong that could be prevented next time? If so, what specific instruction or approach change would help?
2. Is there a recurring pattern across recent tasks? (e.g., "keep getting blocked on Amazon" or "keep asking user about budget")
3. Should any system prompt, script, or default behavior be updated?

The output is appended to `/home/agent/memory/system-learnings.md` — a growing document of operational lessons. This file is included in the context for every future Executor invocation, so the agent literally learns from its past.

**Example entries in system-learnings.md:**
```markdown
## 2026-04-15: Amazon product pages
- Amazon blocks this VPS IP for product detail pages
- Workaround: use Amazon's mobile site (amazon.com?k=...) which has lighter anti-bot
- If that fails too, flag for residential proxy upgrade

## 2026-04-16: User budget preferences
- User was asked about budget twice in flight searches
- Learning: default budget assumption is "mid-range" unless specified
- If task is price-sensitive and no budget given, check second brain for past preferences before asking

## 2026-04-18: Token efficiency
- Headphone research task used 47 turns (near limit)
- Root cause: visited 8 review sites when 3 would have sufficed
- Learning: for product research, cap at 3-4 high-quality sources, don't exhaustively crawl
```

**Periodic (weekly):**

A scheduled n8n workflow runs a deeper analysis:
- Aggregates all task performance records from the past week
- Calculates: average turns per task, question rate, failure rate, review rejection rate
- Compares to previous weeks — are the numbers improving or degrading?
- Sends a brief weekly summary to the user via WhatsApp:
  ```
  Weekly agent report:
  - Tasks completed: 12
  - Success rate: 92% (up from 83%)
  - Avg turns per task: 28 (down from 35)
  - User questions asked: 1 (down from 4)
  - Top learning: Stopped over-crawling product review sites
  ```

### Loop Detection

To prevent the agent from burning tokens going in circles, the Executor prompt includes:

- **Repetition detection:** "If you find yourself attempting the same action more than twice with the same approach, stop. Either try a fundamentally different approach or report the blocker."
- **Escalation over silence:** "It is better to return a partial result with a clear explanation of what's missing than to exhaust all budget trying to achieve perfection."
- **Progress checkpoints:** "After every major step, briefly note what you accomplished. If you are not making measurable progress, stop and return what you have."

### System Prompt Evolution

The system prompts for Planner, Executor, and Reviewer are stored as files on the VPS (`/home/agent/prompts/`). The retrospective process can recommend changes to these prompts, but **does not auto-modify them**. Instead, recommended prompt changes are:
1. Logged to `/home/agent/memory/prompt-change-proposals.md`
2. Included in the weekly summary to the user
3. Applied only after user approval (or after the system has enough confidence from repeated evidence)

This prevents the system from degrading its own prompts through a bad self-modification loop while still allowing it to evolve.

## n8n Workflows to Build

| Workflow | Purpose |
|----------|---------|
| **Task Intake** | Receives WhatsApp/webhook → logs task → triggers planning phase |
| **Planner** | Calls Claude Code #1 for success criteria → sends to user for confirmation |
| **Executor** | Calls Claude Code #2 with task + confirmed criteria → handles mid-task questions |
| **Reviewer** | Calls Claude Code #3 with task + criteria + output → pass/fail decision |
| **Delivery** | Sends result via WhatsApp → stores learnings → updates task log |
| **n8n Backup** | Exports all workflows to JSON → pushes to GitHub (daily) |
| **Email Watcher** | IMAP trigger on `sophia@ptriconsulting.com` → extracts codes → writes to `/home/agent/inbox/` |
| **Task Retrospective** | After each task: logs performance record, runs lightweight learning analysis, updates system-learnings.md |
| **Weekly Review** | Scheduled weekly: aggregates performance data, calculates trends, sends summary to user via WhatsApp |

These may be combined into fewer workflows during implementation — the separation above is logical, not necessarily physical.

## Success Criteria for the System Itself

The system is working when:
1. User can send a task via WhatsApp, receive success criteria for confirmation, and get a quality-reviewed result back — without touching a computer
2. The agent can browse websites, sign up for services, and extract information autonomously
3. Failures are reported clearly with specific errors, not silent
4. The system can self-extend: given a task that requires new tooling, it installs and configures what it needs
5. VPS and n8n can be restored from backups if anything goes wrong
6. The system tracks its own performance and the user can see improvement trends over time (fewer questions asked, fewer review rejections, fewer turns per task)
7. The agent does not go in circles — it detects when it's stuck and escalates or returns partial results rather than burning tokens
