# Claude Session Analytics Routine

**Status:** Planning
**Last updated:** 2026-04-26

> A weekly scheduled routine that parses Claude Code's local JSONL session files to produce a performance report — no tokens spent, no AI reading conversation content.

---

## Critical Assumptions

_Last reviewed: 2026-04-26_

| Assumption | Value | If this changes... |
|------------|-------|--------------------|
| JSONL file location | `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl` | Update all path references in scripts |
| Token fields in JSONL | Unreliable raw — must deduplicate by `requestId` | Replace raw summing with dedup logic |
| Skill calls in JSONL | `type: tool_use`, `name: "Skill"`, `input.skill: "<skill-name>"` | Update skill-detection logic |
| ccusage tool available | `npx ccusage@latest` via npm | Replace with direct Python dedup if unavailable |
| Claude Code version | claude-sonnet-4-6 (April 2026) | Re-verify JSONL schema fields |

---

## User Story

**Pain Points**

1. Usman has no visibility into how efficiently Claude Code sessions are running — he cannot tell after the fact whether a session wasted tokens, looped unnecessarily, or required too much back-and-forth.
2. There is no way to track which skills are being used most, or whether they are actually reducing session length or token spend.
3. Token counts shown in the Claude Code UI are not saved anywhere persistent, so weekly or monthly trends are invisible.
4. Back-and-forth with the user (user clarifying, correcting, redirecting) is the biggest time cost — but there is no current measure of how often this happens.
5. No way to detect when Claude went in a loop (same tool called repeatedly on the same file) without manually reviewing a session.

**Wish List**

1. Every week, a report is automatically generated covering all sessions from the past 7 days — no manual action required.
2. The report shows per-session: duration, total tokens (input/output/cache), estimated cost, tool call breakdown, and skill invocations.
3. The report flags sessions where excessive back-and-forth occurred (user turns > 40% of total turns).
4. The report flags sessions where a loop was likely (same tool called 4+ times on the same file path).
5. Over time, the report builds a correlation between which skills were invoked and session efficiency (token spend, duration, back-and-forth count).
6. The report shows cache hit rate per session so underutilised caching is visible.
7. The entire routine runs without spending any Claude tokens — purely local script execution.

**Potential Risks**

1. JSONL token counts are placeholder-heavy (75% of entries have zeros) — raw summing gives numbers 100–174× too low. Deduplication by `requestId` is mandatory.
2. Reading a JSONL file while a session is actively running is safe (no crashes) but will only reflect what has been written so far — run the report after sessions complete, not during.
3. If a skill is invoked inside a scheduled Claude agent to run the scripts, there is a risk it will use the Read tool on JSONL files (which would spend tokens). The skill must explicitly instruct Claude to use Bash only, and explicitly prohibit Read on JSONL files.
4. Tokens per individual tool call are not stored — only per assistant turn (which may contain multiple tool calls). Token-per-tool attribution is approximate only.

---

## How It Works

### What is extractable with zero tokens

All of the following comes from parsing JSONL structure only — no conversation text is read, no Claude involvement needed:

| Metric | Source field |
|--------|-------------|
| Session duration | First vs. last `timestamp` |
| Total input / output tokens | `message.usage.input_tokens` / `output_tokens` (after dedup by `requestId`) |
| Cache tokens | `message.usage.cache_creation_input_tokens` / `cache_read_input_tokens` |
| Estimated cost | `costUSD` field |
| Tool call names and counts | `type: tool_use`, `name` field inside assistant messages |
| Skill names invoked | `name: "Skill"`, `input.skill` value |
| User turns vs. assistant turns | Count `type: user` vs `type: assistant` entries |
| Total session turns | All timestamped entries |
| Loop detection | Same `name` + same file path argument, 4+ consecutive calls |
| Read:Edit ratio | Count `Read` vs `Edit` tool_use entries |
| Model used | `model` field |
| Project / working directory | `cwd` field |
| Git branch | `gitBranch` field |

### What requires reading conversation text (costs tokens)

- Quality of Claude's responses
- Whether a solution was correct or required correction
- Reasoning depth or clarity
- Why a loop happened (only that it happened is detectable)

### Architecture

```
Weekly cron trigger (n8n or Claude scheduled agent)
  └─> Bash: run ccusage for token totals (handles dedup internally)
  └─> Bash: run Python script for structural signals
        ├─ Loop detection (same tool × same path, 4+ times)
        ├─ Back-and-forth ratio (user turns / total turns)
        ├─ Read:Edit ratio per session
        ├─ Skill invocation list with counts
        └─ Cache hit rate (cache_read / total input tokens)
  └─> Combine into markdown report
  └─> Save to Second Brain or designated report file
```

### File paths

Claude Code JSONL files are stored at:
```
~/.claude/projects/<encoded-cwd>/<session-id>.jsonl
```

The `<encoded-cwd>` is the absolute working directory with every non-alphanumeric character replaced by `-`. For example:

| Working directory | Encoded path |
|-------------------|-------------|
| `/Users/usmanramay/Documents/Projects/Personal-Projects` | `-Users-usmanramay-Documents-Projects-Personal-Projects` |
| `/Users/usmanramay/Documents/Projects/Experiment-Of-Life/EoL-Context` | `-Users-usmanramay-Documents-Projects-Experiment-Of-Life-EoL-Context` |

### Core Python script

```python
import json
import glob
import os
from collections import Counter, defaultdict
from datetime import datetime, timedelta, timezone

PROJECTS_DIR = os.path.expanduser("~/.claude/projects")
LOOKBACK_DAYS = 7

def parse_session(filepath):
    timestamps = []
    tools = Counter()
    skills = Counter()
    user_turns = 0
    assistant_turns = 0
    tool_sequences = []  # list of (tool_name, file_arg) tuples
    seen_request_ids = set()
    input_tokens = 0
    output_tokens = 0
    cache_read_tokens = 0
    cache_write_tokens = 0
    reads = 0
    edits = 0

    for line in open(filepath):
        try:
            obj = json.loads(line)
        except Exception:
            continue

        ts = obj.get("timestamp")
        if ts:
            try:
                timestamps.append(datetime.fromisoformat(ts.replace("Z", "+00:00")))
            except Exception:
                pass

        t = obj.get("type")
        if t == "user":
            user_turns += 1
        elif t == "assistant":
            assistant_turns += 1
            msg = obj.get("message", {})

            # Deduplicate token counts by requestId
            req_id = msg.get("id") or obj.get("requestId")
            if req_id and req_id not in seen_request_ids:
                seen_request_ids.add(req_id)
                usage = msg.get("usage", {})
                input_tokens += usage.get("input_tokens", 0)
                output_tokens += usage.get("output_tokens", 0)
                cache_read_tokens += usage.get("cache_read_input_tokens", 0)
                cache_write_tokens += usage.get("cache_creation_input_tokens", 0)

            for block in msg.get("content", []):
                if not isinstance(block, dict):
                    continue
                if block.get("type") == "tool_use":
                    name = block.get("name", "")
                    inp = block.get("input", {})
                    tools[name] += 1
                    if name == "Skill":
                        skills[inp.get("skill", "unknown")] += 1
                    if name == "Read":
                        reads += 1
                    if name == "Edit":
                        edits += 1
                    # capture file path for loop detection
                    file_arg = inp.get("file_path") or inp.get("command", "")[:60]
                    tool_sequences.append((name, file_arg))

    # Loop detection: same tool + same file 4+ consecutive times
    loops_detected = 0
    i = 0
    while i < len(tool_sequences) - 3:
        if all(tool_sequences[i+j] == tool_sequences[i] for j in range(1, 4)):
            loops_detected += 1
            i += 4
        else:
            i += 1

    duration = (max(timestamps) - min(timestamps)) if len(timestamps) >= 2 else timedelta(0)
    total_turns = user_turns + assistant_turns
    back_and_forth_ratio = (user_turns / total_turns) if total_turns > 0 else 0
    cache_hit_rate = (cache_read_tokens / (cache_read_tokens + input_tokens)) if (cache_read_tokens + input_tokens) > 0 else 0

    return {
        "file": os.path.basename(filepath),
        "start": min(timestamps) if timestamps else None,
        "duration_min": round(duration.total_seconds() / 60, 1),
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "cache_read_tokens": cache_read_tokens,
        "cache_write_tokens": cache_write_tokens,
        "cache_hit_rate_pct": round(cache_hit_rate * 100, 1),
        "total_turns": total_turns,
        "back_and_forth_pct": round(back_and_forth_ratio * 100, 1),
        "tools": dict(tools.most_common()),
        "skills": dict(skills),
        "read_edit_ratio": round(reads / edits, 1) if edits > 0 else None,
        "loops_detected": loops_detected,
        "flags": []
    }

def add_flags(session):
    if session["back_and_forth_pct"] > 40:
        session["flags"].append("HIGH_BACK_AND_FORTH")
    if session["loops_detected"] > 0:
        session["flags"].append(f"LOOP_DETECTED x{session['loops_detected']}")
    if session["cache_hit_rate_pct"] < 20 and session["input_tokens"] > 5000:
        session["flags"].append("LOW_CACHE_HIT_RATE")
    if session["read_edit_ratio"] and session["read_edit_ratio"] < 3:
        session["flags"].append("LOW_READ_EDIT_RATIO")

cutoff = datetime.now(timezone.utc) - timedelta(days=LOOKBACK_DAYS)
sessions = []

for jsonl_file in glob.glob(f"{PROJECTS_DIR}/**/*.jsonl", recursive=True):
    try:
        mtime = datetime.fromtimestamp(os.path.getmtime(jsonl_file), tz=timezone.utc)
        if mtime < cutoff:
            continue
        s = parse_session(jsonl_file)
        if s["total_turns"] < 2:
            continue
        add_flags(s)
        sessions.append(s)
    except Exception as e:
        pass

sessions.sort(key=lambda x: x["start"] or datetime.min.replace(tzinfo=timezone.utc), reverse=True)

print(f"# Claude Session Report — Last {LOOKBACK_DAYS} Days\n")
print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}\n")
print(f"Sessions analysed: {len(sessions)}\n")
print("---\n")

for s in sessions:
    date_str = s["start"].strftime("%Y-%m-%d %H:%M") if s["start"] else "unknown"
    flags_str = " ⚠️ " + ", ".join(s["flags"]) if s["flags"] else ""
    print(f"## {date_str}{flags_str}")
    print(f"- **Duration:** {s['duration_min']} min")
    print(f"- **Tokens:** {s['input_tokens']:,} in / {s['output_tokens']:,} out")
    print(f"- **Cache hit rate:** {s['cache_hit_rate_pct']}%")
    print(f"- **Turns:** {s['total_turns']} total, {s['back_and_forth_pct']}% user")
    if s["read_edit_ratio"]:
        print(f"- **Read:Edit ratio:** {s['read_edit_ratio']}")
    if s["tools"]:
        tools_str = ", ".join(f"{k} ({v})" for k, v in s["tools"].items())
        print(f"- **Tools:** {tools_str}")
    if s["skills"]:
        skills_str = ", ".join(f"{k} ({v})" for k, v in s["skills"].items())
        print(f"- **Skills:** {skills_str}")
    print()
```

Save this script as `~/.claude/scripts/session-report.py` and run it via:

```bash
python3 ~/.claude/scripts/session-report.py
```

### How to run this via a scheduled agent (token-safe)

When triggering this from inside Claude Code (e.g., via a skill or scheduled agent), the skill must use the Bash tool only. Example skill instruction:

```markdown
**IMPORTANT: Never use the Read tool on JSONL files. Always run the script below using the Bash tool.**

Run the following and report the output:

```bash
python3 ~/.claude/scripts/session-report.py
```

Do not open any JSONL files directly. Your entire input is the script output above.
```

### Existing community tools (for token totals validation)

| Tool | Purpose | Install |
|------|---------|---------|
| ccusage | Token totals, cost, cache — already handles dedup | `npx ccusage@latest` |
| claude-session-analyzer | Read:Edit ratio, behavioral signals | GitHub: lucemia/claude-session-analyzer |
| claude-token-analyzer | Anomaly scoring across sessions | GitHub: li195111/claude-token-analyzer |

---

## Current Status

Planning only. Script drafted above but not saved to disk or tested. No scheduled routine created yet.

---

## Open Tasks and Ideas

- [ ] Save the Python script to `~/.claude/scripts/session-report.py`
- [ ] Test the script manually against one week of real JSONL files
- [ ] Validate token totals against `npx ccusage@latest` output
- [ ] Create scheduled routine (n8n or Claude scheduled agent) to run weekly
- [ ] Decide report destination: Second Brain, local markdown file, or both
- [ ] Idea: track skill-to-session-efficiency correlation over 4+ weeks once data accumulates
- [ ] Idea: add per-project breakdown (group sessions by `cwd`)

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-26 | Document created | Conversation establishing approach and validating JSONL structure |
