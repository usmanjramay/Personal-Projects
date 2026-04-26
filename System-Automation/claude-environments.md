# Claude Environments & Configuration

**Status:** Testing
**Last updated:** 2026-04-26

> Reference for how Claude Code environments are configured and what still needs to be done to run them reliably across projects.

---

## Permission Settings

All Claude Code permissions are managed centrally in `~/.claude/settings.json`. This is the single source of truth for:

- Tool allow/deny rules (Bash, Read, Edit, MCP servers, etc.)
- Additional directories Claude can browse
- Permission default mode per project

**All local `settings.local.json` files across every project have been wiped to `{}`** — Personal-Projects, System-Automation, and EoL-Context. Any future permission changes should be made to `~/.claude/settings.json` and tracked in this file's change log below.

The one exception is project-level `settings.json` files (not local), which set `defaultMode: auto` — this is intentional and stays per-project since it applies to web sessions too.

---

## How It Works

Claude Code on the web reads configuration from the GitHub repository of the project being worked on. It does not use local machine settings.

### What the Web Session Reads

| Source | File | Purpose |
|--------|------|---------|
| GitHub repo | `CLAUDE.md` | Project instructions and rules |
| GitHub repo | `.claude/settings.json` | Permissions and session behavior |
| GitHub repo | `.mcp.json` | MCP server connections |
| claude.ai Environment settings | Environment variables | Secrets and credentials for MCP servers |

### MCP Configuration

MCP servers (e.g., Second Brain, n8n) must be declared in `.mcp.json` in the GitHub repo. The file specifies which servers to connect and how, but sensitive values (API keys, URLs) should be stored as environment variables in claude.ai — not hardcoded in the file.

### Environments

Web sessions run inside named environments configured in claude.ai. Currently one environment is set up: **default**. This environment is shared across all projects running web sessions — any environment variables added here are available to all of them.

For further reference: [Claude Code on the Web documentation](https://code.claude.com/docs/en/claude-code-on-the-web)

---

## Permissions

Permissions are configured in `.claude/settings.json` in each repo. The current default everywhere is **auto mode**, which uses machine-learning classifiers to decide what to approve automatically vs. what to pause and ask.

**Unattended sessions require an extra step.** Because the same `settings.json` is used both locally (where auto mode is fine) and in web sessions (where there is no one to answer prompts), unattended web sessions must be launched with the `--dangerously-skip-permissions` flag. This tells Claude to skip all permission prompts and proceed autonomously. This flag must be set manually at session start — it is not in `settings.json` because it should not apply to local sessions.

---

## Auto-Merge for Scheduled Agents

When a web session runs on a schedule (or any time Claude is not being watched), it works by creating a branch and opening a PR — it does not push directly to main. GitHub then requires a human to review and merge. To allow fully unattended operation, auto-merge needs to be enabled.

### How auto-merge works

1. A PR is created by the agent.
2. GitHub Actions runs any configured checks (CI/CD).
3. When all checks pass (green), GitHub auto-merges the PR to main.

### What is needed to enable it

Two settings in GitHub, nothing else:

1. **Enable branch protection on `main`** — GitHub requires this as a prerequisite before auto-merge can be turned on. No checks need to be required; just having branch protection active is enough.
2. **Enable auto-merge on the repo** — once branch protection is on, this can be turned on in repo Settings → General.

No GitHub Actions or CI workflows are needed. Since there are no required checks, the auto-merge condition is satisfied the moment the PR is created. The full flow becomes:

1. Claude creates a branch and pushes changes
2. Claude opens a PR
3. GitHub auto-merges it to main within seconds
4. Done — no human action needed

**Current decision:** Not setting this up yet — there is no current need for scheduled agents that commit to these repos. When the need arises, the two steps above are all that is required.

---

## Known Limitations

### Desktop App: Skills Not Showing in Slash Command Menu

The Desktop App (v2.1.110+) has a known regression where plugin/local skills don't appear in the `/` menu — only a handful of outdated ones show up. Root cause: Desktop App uses a registry-based discovery system, not a direct filesystem scan like the VS Code extension. Skills still *load* at runtime if you type the exact name, but they won't appear in the dropdown.

**Workaround:** Use the VS Code/Cursor extension for Claude Code work — skills work correctly there.

Tracked in: GitHub Issue #48963

---

## Open Items

- [ ] Add `.mcp.json` to the `EoL-Context` repo so web sessions there have the same MCP access as Personal-Projects.
- [ ] When unattended scheduled sessions are needed: enable branch protection on `main` (no checks required) + enable auto-merge on the repo — two settings, no GitHub Actions needed.

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-26 | File created as web-sessions.md | Document web session configuration after initial setup |
| 2026-04-26 | Renamed to claude-environments.md; added Permission Settings section | Broadened scope to cover all Claude config; centralized all permissions to ~/.claude/settings.json |
