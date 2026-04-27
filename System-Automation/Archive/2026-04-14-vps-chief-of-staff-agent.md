# VPS Chief of Staff Agent — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up an autonomous AI agent on the Hostinger VPS that receives tasks via WhatsApp/webhook, executes them with browser automation, and returns quality-reviewed results.

**Architecture:** n8n orchestrates a three-phase flow (Plan → Execute → Review) using Claude Code CLI invocations on the VPS. Patchright provides headless browser automation. The system tracks its own performance and improves over time.

**Tech Stack:** Claude Code CLI, Patchright (Node.js), n8n, Supabase, Hostinger VPS (Ubuntu), WhatsApp Business API

**Spec:** `docs/superpowers/specs/2026-04-14-vps-chief-of-staff-agent-design.md`

---

## File Structure Overview

### VPS Files (created during setup)

```
/home/agent/
  ├── prompts/
  │   ├── planner.md              # Planner agent system prompt
  │   ├── executor.md             # Executor agent system prompt
  │   ├── reviewer.md             # Reviewer agent system prompt
  │   └── retrospective.md        # Retrospective agent system prompt
  ├── scripts/
  │   ├── browse.js               # Open URL, return page content as markdown
  │   ├── interact.js             # Navigate, click, fill forms, screenshots
  │   ├── screenshot.js           # Capture page as image
  │   ├── wait_for_email.js        # Watch inbox for verification codes (fs.watch, 3min timeout)
  │   └── session.js              # Manage login sessions (save/restore cookies)
  ├── schemas/
  │   ├── planner-output.json     # JSON schema for planner response
  │   ├── executor-output.json    # JSON schema for executor response
  │   └── reviewer-output.json    # JSON schema for reviewer response
  ├── workdir/                    # Claude Code working directory
  ├── browser-data/               # Persistent Chrome profile
  ├── inbox/                      # Verification codes from n8n
  ├── memory/
  │   ├── system-learnings.md     # Growing document of operational learnings
  │   └── prompt-change-proposals.md  # Proposed prompt improvements
  ├── logs/                       # Task execution logs
  └── repos/                      # Cloned GitHub repos
```

### n8n Workflows (created via n8n MCP tools)

```
Workflows:
  ├── Chief of Staff: Task Intake      # WhatsApp/webhook → log → trigger planner
  ├── Chief of Staff: Planner          # Claude Code #1 → criteria → WhatsApp confirmation
  ├── Chief of Staff: Executor         # Claude Code #2 → work → handle questions
  ├── Chief of Staff: Reviewer         # Claude Code #3 → pass/fail → iterate or deliver
  ├── Chief of Staff: Delivery         # WhatsApp result → store learnings → update log
  ├── Chief of Staff: Retrospective    # Post-task analysis → update system-learnings.md
  ├── Chief of Staff: Email Watcher    # IMAP trigger → extract codes → write to inbox/
  ├── Chief of Staff: Weekly Review    # Scheduled: aggregate metrics → WhatsApp summary
  └── Chief of Staff: n8n Backup       # Daily: export workflows → push to GitHub
```

---

## Phase 1: VPS Foundation

### Task 1: Create the `agent` User and Directory Structure

**Context:** The VPS is at 46.202.159.171, SSH as root with Ed25519 key. We need a dedicated user with sudo access and the full directory tree.

**Prerequisites:** SSH access to VPS working (`ssh-add` if needed)

- [ ] **Step 1: SSH into VPS and create the agent user**

```bash
ssh root@46.202.159.171

# Create user with home directory and bash shell
useradd -m -s /bin/bash agent

# Set a password (will be needed for sudo)
passwd agent
# Enter a strong password when prompted

# Add to sudo group
usermod -aG sudo agent
```

- [ ] **Step 2: Create the full directory structure**

```bash
# As root, create all directories
su - agent
mkdir -p ~/prompts ~/scripts ~/schemas ~/workdir ~/browser-data ~/inbox ~/memory ~/logs ~/repos
```

- [ ] **Step 3: Verify the structure**

```bash
find /home/agent -type d | sort
```

Expected output:
```
/home/agent
/home/agent/browser-data
/home/agent/inbox
/home/agent/logs
/home/agent/memory
/home/agent/prompts
/home/agent/repos
/home/agent/schemas
/home/agent/scripts
/home/agent/workdir
```

- [ ] **Step 4: Initialize memory files**

```bash
cat > /home/agent/memory/system-learnings.md << 'EOF'
# System Learnings

This file is automatically updated after each task. It contains operational lessons
learned by the Chief of Staff agent. Each entry includes the date, what happened,
and what to do differently next time.

This file is included in every Executor invocation as context.

---

EOF

cat > /home/agent/memory/prompt-change-proposals.md << 'EOF'
# Prompt Change Proposals

Proposed changes to system prompts, identified during retrospective analysis.
These are reviewed by the user before being applied.

---

EOF
```

- [ ] **Step 5: Set up SSH key for the agent user (for GitHub access)**

```bash
# As agent user
su - agent
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "sophia@ptriconsulting.com"
cat ~/.ssh/id_ed25519.pub
```

Copy the public key output — it will be added to GitHub in a later step.

---

### Task 2: Install Node.js and Claude Code CLI

**Context:** Claude Code requires Node.js 18+. Patchright is an npm package.

- [ ] **Step 1: Install Node.js 20 LTS**

```bash
# As root
ssh root@46.202.159.171

# Install Node.js 20 via NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

# Verify
node --version  # Should show v20.x.x
npm --version   # Should show 10.x.x
```

- [ ] **Step 2: Install Claude Code CLI**

```bash
# As agent user
su - agent
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
```

- [ ] **Step 3: Generate OAuth token on Usman's Mac**

On the local Mac (not VPS):
```bash
claude setup-token
```

Follow the interactive flow — it will open a browser, authenticate with your Anthropic account, and output a token.

- [ ] **Step 4: Configure the OAuth token on VPS**

```bash
# As agent user on VPS
su - agent

# Add token to .bashrc for persistence
echo 'export CLAUDE_CODE_OAUTH_TOKEN="<paste-token-here>"' >> ~/.bashrc
source ~/.bashrc

# Verify Claude Code works
claude -p "Say hello" --output-format json
```

Expected: JSON response with `"subtype": "success"` and `"result"` containing a greeting.

- [ ] **Step 5: Test with dangerously-skip-permissions**

```bash
claude -p "List files in /home/agent/" --output-format json --dangerously-skip-permissions
```

Expected: JSON response listing the directory contents.

---

### Task 3: Install Patchright and Browser Dependencies

**Context:** Patchright is a patched fork of Playwright with anti-detection. It needs system dependencies for headless Chromium on Linux.

- [ ] **Step 1: Install Patchright**

```bash
# As agent user
su - agent
cd ~

# Initialize npm project
npm init -y

# Install patchright
npm install patchright

# Install browser binaries + system dependencies
npx patchright install chromium
sudo npx patchright install-deps chromium
```

- [ ] **Step 2: Test that headless Chromium launches**

```bash
cat > /tmp/test-browser.js << 'SCRIPT'
const { chromium } = require('patchright');

(async () => {
  const browser = await chromium.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto('https://example.com');
  const title = await page.title();
  console.log('Page title:', title);
  await browser.close();
  console.log('Browser test passed');
})().catch(err => {
  console.error('Browser test failed:', err.message);
  process.exit(1);
});
SCRIPT

cd /home/agent && node /tmp/test-browser.js
```

Expected output:
```
Page title: Example Domain
Browser test passed
```

- [ ] **Step 3: Test with persistent profile directory**

```bash
cat > /tmp/test-profile.js << 'SCRIPT'
const { chromium } = require('patchright');

(async () => {
  const context = await chromium.launchPersistentContext('/home/agent/browser-data', {
    headless: true
  });
  const page = context.pages()[0] || await context.newPage();
  await page.goto('https://example.com');
  console.log('Title:', await page.title());
  await context.close();
  console.log('Persistent profile test passed');
})().catch(err => {
  console.error('Test failed:', err.message);
  process.exit(1);
});
SCRIPT

cd /home/agent && node /tmp/test-profile.js
```

Expected: Same output, plus files created in `/home/agent/browser-data/`.

- [ ] **Step 4: Clean up test files**

```bash
rm /tmp/test-browser.js /tmp/test-profile.js
```

---

## Phase 2: Browser Scripts

### Task 4: Create browse.js — URL to Markdown

**Context:** The simplest and most-used browser script. Takes a URL, loads it, and returns the page content as clean markdown suitable for LLM consumption.

- [ ] **Step 1: Create browse.js**

```bash
cat > /home/agent/scripts/browse.js << 'SCRIPT'
const { chromium } = require('patchright');

const url = process.argv[2];
if (!url) {
  console.error(JSON.stringify({ error: 'Usage: node browse.js <url>' }));
  process.exit(1);
}

const timeout = parseInt(process.argv[3] || '30000', 10);

(async () => {
  const context = await chromium.launchPersistentContext('/home/agent/browser-data', {
    headless: true,
    args: ['--disable-blink-features=AutomationControlled']
  });

  try {
    const page = context.pages()[0] || await context.newPage();
    const response = await page.goto(url, { waitUntil: 'domcontentloaded', timeout });

    if (!response) {
      console.error(JSON.stringify({
        error: `Failed to load ${url}: no response received`,
        url
      }));
      process.exit(1);
    }

    const status = response.status();
    if (status >= 400) {
      console.error(JSON.stringify({
        error: `HTTP ${status} loading ${url}`,
        url,
        status
      }));
      process.exit(1);
    }

    // Extract text content as clean markdown-ish format
    const content = await page.evaluate(() => {
      // Remove script, style, nav, footer, ads
      const removeSelectors = ['script', 'style', 'nav', 'footer', 'iframe',
        '[role="banner"]', '[role="navigation"]', '.ad', '.ads', '.advertisement',
        '.cookie-banner', '.popup', '#cookie-consent'];
      removeSelectors.forEach(sel => {
        document.querySelectorAll(sel).forEach(el => el.remove());
      });

      function extractText(element) {
        const lines = [];
        for (const node of element.childNodes) {
          if (node.nodeType === 3) { // text node
            const text = node.textContent.trim();
            if (text) lines.push(text);
          } else if (node.nodeType === 1) { // element node
            const tag = node.tagName.toLowerCase();
            if (['h1','h2','h3','h4','h5','h6'].includes(tag)) {
              const level = parseInt(tag[1]);
              const prefix = '#'.repeat(level);
              lines.push(`\n${prefix} ${node.textContent.trim()}\n`);
            } else if (tag === 'p') {
              lines.push(`\n${node.textContent.trim()}\n`);
            } else if (tag === 'li') {
              lines.push(`- ${node.textContent.trim()}`);
            } else if (tag === 'a') {
              const href = node.getAttribute('href');
              const text = node.textContent.trim();
              if (href && text) {
                lines.push(`[${text}](${href})`);
              } else if (text) {
                lines.push(text);
              }
            } else if (['div', 'section', 'article', 'main'].includes(tag)) {
              lines.push(...extractText(node));
            } else {
              const text = node.textContent.trim();
              if (text) lines.push(text);
            }
          }
        }
        return lines;
      }

      const main = document.querySelector('main, article, [role="main"]') || document.body;
      return extractText(main).join('\n').replace(/\n{3,}/g, '\n\n').trim();
    });

    const result = {
      url,
      title: await page.title(),
      status,
      content: content.substring(0, 50000) // Cap at 50k chars to avoid overwhelming LLM context
    };

    console.log(JSON.stringify(result));
  } finally {
    await context.close();
  }
})().catch(err => {
  console.error(JSON.stringify({
    error: `Browser error: ${err.message}`,
    url
  }));
  process.exit(1);
});
SCRIPT
```

- [ ] **Step 2: Test browse.js**

```bash
cd /home/agent && node scripts/browse.js "https://example.com" | python3 -m json.tool
```

Expected: JSON with `url`, `title` ("Example Domain"), `status` (200), and `content` containing the page text.

- [ ] **Step 3: Test with a real content page**

```bash
cd /home/agent && node scripts/browse.js "https://en.wikipedia.org/wiki/Luxembourg" | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Title: {d[\"title\"]}\nContent length: {len(d[\"content\"])} chars\nFirst 200 chars: {d[\"content\"][:200]}')"
```

Expected: Title should be "Luxembourg - Wikipedia", content should be substantial.

---

### Task 5: Create interact.js — Form Filling and Navigation

**Context:** For interactive browsing — filling forms, clicking buttons, taking screenshots. Actions are passed as a JSON array.

- [ ] **Step 1: Create interact.js**

```bash
cat > /home/agent/scripts/interact.js << 'SCRIPT'
const { chromium } = require('patchright');
const fs = require('fs');

const url = process.argv[2];
const actionsJson = process.argv[3];

if (!url || !actionsJson) {
  console.error(JSON.stringify({
    error: 'Usage: node interact.js <url> \'[{"action":"fill","selector":"#email","value":"test@example.com"},{"action":"click","selector":"#submit"}]\''
  }));
  process.exit(1);
}

let actions;
try {
  actions = JSON.parse(actionsJson);
} catch (e) {
  console.error(JSON.stringify({ error: `Invalid JSON actions: ${e.message}` }));
  process.exit(1);
}

(async () => {
  const context = await chromium.launchPersistentContext('/home/agent/browser-data', {
    headless: true,
    args: ['--disable-blink-features=AutomationControlled']
  });

  try {
    const page = context.pages()[0] || await context.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30000 });

    const results = [];

    for (const action of actions) {
      try {
        switch (action.action) {
          case 'fill':
            await page.fill(action.selector, action.value);
            results.push({ action: 'fill', selector: action.selector, status: 'ok' });
            break;

          case 'click':
            await page.click(action.selector, { timeout: 10000 });
            results.push({ action: 'click', selector: action.selector, status: 'ok' });
            break;

          case 'select':
            await page.selectOption(action.selector, action.value);
            results.push({ action: 'select', selector: action.selector, status: 'ok' });
            break;

          case 'wait':
            await page.waitForSelector(action.selector, { timeout: parseInt(action.timeout || '10000', 10) });
            results.push({ action: 'wait', selector: action.selector, status: 'ok' });
            break;

          case 'screenshot':
            const screenshotPath = action.path || `/home/agent/logs/screenshot-${Date.now()}.png`;
            await page.screenshot({ path: screenshotPath, fullPage: action.fullPage || false });
            results.push({ action: 'screenshot', path: screenshotPath, status: 'ok' });
            break;

          case 'type':
            await page.type(action.selector, action.value, { delay: parseInt(action.delay || '50', 10) });
            results.push({ action: 'type', selector: action.selector, status: 'ok' });
            break;

          case 'press':
            await page.press(action.selector || 'body', action.key);
            results.push({ action: 'press', key: action.key, status: 'ok' });
            break;

          case 'wait_for_navigation':
            await page.waitForNavigation({ timeout: parseInt(action.timeout || '15000', 10) });
            results.push({ action: 'wait_for_navigation', status: 'ok' });
            break;

          case 'get_text':
            const text = await page.textContent(action.selector);
            results.push({ action: 'get_text', selector: action.selector, status: 'ok', value: text });
            break;

          case 'delay':
            await new Promise(r => setTimeout(r, parseInt(action.ms || '1000', 10)));
            results.push({ action: 'delay', ms: action.ms, status: 'ok' });
            break;

          default:
            results.push({ action: action.action, status: 'error', error: `Unknown action: ${action.action}` });
        }
      } catch (actionErr) {
        results.push({
          action: action.action,
          selector: action.selector,
          status: 'error',
          error: actionErr.message
        });
        // Continue with remaining actions unless it's critical
        if (action.required !== false) break;
      }
    }

    // Get final page state
    const finalState = {
      url: page.url(),
      title: await page.title(),
      actions: results
    };

    console.log(JSON.stringify(finalState));
  } finally {
    await context.close();
  }
})().catch(err => {
  console.error(JSON.stringify({
    error: `Browser error: ${err.message}`,
    url
  }));
  process.exit(1);
});
SCRIPT
```

- [ ] **Step 2: Test interact.js with a simple action**

```bash
cd /home/agent && node scripts/interact.js "https://example.com" '[{"action":"screenshot","path":"/home/agent/logs/test-screenshot.png"}]' | python3 -m json.tool
```

Expected: JSON with final URL, title, and `actions` array showing screenshot status "ok". Screenshot file should exist at the path.

```bash
ls -la /home/agent/logs/test-screenshot.png
```

- [ ] **Step 3: Clean up test screenshot**

```bash
rm -f /home/agent/logs/test-screenshot.png
```

---

### Task 6: Create screenshot.js and session.js

**Context:** Simpler utility scripts — screenshot.js is a shortcut for capturing a page, session.js manages login state.

- [ ] **Step 1: Create screenshot.js**

```bash
cat > /home/agent/scripts/screenshot.js << 'SCRIPT'
const { chromium } = require('patchright');

const url = process.argv[2];
const outputPath = process.argv[3] || `/home/agent/logs/screenshot-${Date.now()}.png`;
const fullPage = process.argv[4] === 'full';

if (!url) {
  console.error(JSON.stringify({ error: 'Usage: node screenshot.js <url> [output-path] [full]' }));
  process.exit(1);
}

(async () => {
  const context = await chromium.launchPersistentContext('/home/agent/browser-data', {
    headless: true,
    args: ['--disable-blink-features=AutomationControlled']
  });

  try {
    const page = context.pages()[0] || await context.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30000 });
    await page.screenshot({ path: outputPath, fullPage });
    console.log(JSON.stringify({ url, path: outputPath, status: 'ok' }));
  } finally {
    await context.close();
  }
})().catch(err => {
  console.error(JSON.stringify({ error: err.message, url }));
  process.exit(1);
});
SCRIPT
```

- [ ] **Step 2: Create session.js**

```bash
cat > /home/agent/scripts/session.js << 'SCRIPT'
const { chromium } = require('patchright');
const fs = require('fs');

const command = process.argv[2]; // 'save' or 'restore' or 'list'
const sessionName = process.argv[3];
const sessionsDir = '/home/agent/browser-data/sessions';

if (!fs.existsSync(sessionsDir)) {
  fs.mkdirSync(sessionsDir, { recursive: true });
}

(async () => {
  switch (command) {
    case 'save': {
      if (!sessionName) {
        console.error(JSON.stringify({ error: 'Usage: node session.js save <name>' }));
        process.exit(1);
      }
      const context = await chromium.launchPersistentContext('/home/agent/browser-data', {
        headless: true
      });
      const state = await context.storageState();
      const path = `${sessionsDir}/${sessionName}.json`;
      fs.writeFileSync(path, JSON.stringify(state, null, 2));
      await context.close();
      console.log(JSON.stringify({ command: 'save', name: sessionName, path, status: 'ok' }));
      break;
    }

    case 'restore': {
      if (!sessionName) {
        console.error(JSON.stringify({ error: 'Usage: node session.js restore <name>' }));
        process.exit(1);
      }
      const path = `${sessionsDir}/${sessionName}.json`;
      if (!fs.existsSync(path)) {
        console.error(JSON.stringify({ error: `Session not found: ${sessionName}`, path }));
        process.exit(1);
      }
      const context = await chromium.launchPersistentContext('/home/agent/browser-data', {
        headless: true,
        storageState: path
      });
      await context.close();
      console.log(JSON.stringify({ command: 'restore', name: sessionName, status: 'ok' }));
      break;
    }

    case 'list': {
      const sessions = fs.readdirSync(sessionsDir)
        .filter(f => f.endsWith('.json'))
        .map(f => f.replace('.json', ''));
      console.log(JSON.stringify({ command: 'list', sessions }));
      break;
    }

    default:
      console.error(JSON.stringify({ error: 'Usage: node session.js <save|restore|list> [name]' }));
      process.exit(1);
  }
})().catch(err => {
  console.error(JSON.stringify({ error: err.message }));
  process.exit(1);
});
SCRIPT
```

- [ ] **Step 3: Test screenshot.js**

```bash
cd /home/agent && node scripts/screenshot.js "https://example.com" "/home/agent/logs/test.png"
ls -la /home/agent/logs/test.png
rm /home/agent/logs/test.png
```

- [ ] **Step 4: Test session.js list (should be empty)**

```bash
cd /home/agent && node scripts/session.js list
```

Expected: `{"command":"list","sessions":[]}`

---

### Task 7: Create wait_for_email.js — Verification Code Watcher

**Context:** When the executor signs up on a website and triggers a verification email, it needs to wait for n8n to extract the code and write it to `/home/agent/inbox/`. This script uses `fs.watch()` to detect new files, with a 3-minute timeout.

- [ ] **Step 1: Create wait_for_email.js**

```bash
cat > /home/agent/scripts/wait_for_email.js << 'SCRIPT'
const fs = require('fs');
const path = require('path');

const inboxDir = '/home/agent/inbox';
const timeoutMs = parseInt(process.argv[2] || '180000', 10); // Default 3 minutes
const site = process.argv[3] || 'unknown site';

// Get existing files before we start watching
const existingFiles = new Set(fs.readdirSync(inboxDir));

const startTime = Date.now();

const watcher = fs.watch(inboxDir, (eventType, filename) => {
  if (eventType === 'rename' && filename && !existingFiles.has(filename)) {
    const filePath = path.join(inboxDir, filename);
    // Small delay to ensure file is fully written
    setTimeout(() => {
      try {
        const content = fs.readFileSync(filePath, 'utf-8').trim();
        watcher.close();
        console.log(JSON.stringify({
          status: 'ok',
          code: content,
          file: filename,
          waited_ms: Date.now() - startTime
        }));
        process.exit(0);
      } catch (err) {
        // File might not be ready yet, ignore
      }
    }, 200);
  }
});

// Timeout handler
setTimeout(() => {
  watcher.close();
  console.error(JSON.stringify({
    error: `Verification email not received within ${timeoutMs / 1000}s timeout for ${site}`,
    waited_ms: timeoutMs,
    site
  }));
  process.exit(1);
}, timeoutMs);
SCRIPT
```

- [ ] **Step 2: Test wait_for_email.js with a simulated code delivery**

In one terminal:
```bash
cd /home/agent && node scripts/wait_for_email.js 10000 "test-site.com" &
```

In another terminal (within 10 seconds):
```bash
echo "123456" > /home/agent/inbox/test-$(date +%s).txt
```

Expected: The first terminal should output JSON with `"status": "ok"`, `"code": "123456"`.

- [ ] **Step 3: Test timeout behavior**

```bash
cd /home/agent && node scripts/wait_for_email.js 3000 "test-site.com"
```

Wait 3 seconds. Expected: JSON error with "Verification email not received within 3s timeout."

- [ ] **Step 4: Clean up test files**

```bash
rm -f /home/agent/inbox/test-*
```

---

## Phase 3: System Prompts and Output Schemas

### Task 8: Create JSON Output Schemas

**Context:** These schemas are passed to `--json-schema` to ensure Claude Code returns structured, parseable output for each agent role.

- [ ] **Step 1: Create planner output schema**

```bash
cat > /home/agent/schemas/planner-output.json << 'SCHEMA'
{
  "type": "object",
  "properties": {
    "task_understanding": {
      "type": "string",
      "description": "One paragraph restating the task to confirm understanding"
    },
    "success_criteria": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "criterion": { "type": "string" },
          "measurable": { "type": "string", "description": "How to objectively verify this criterion is met" }
        },
        "required": ["criterion", "measurable"]
      },
      "minItems": 2
    },
    "approach_summary": {
      "type": "string",
      "description": "Brief description of how the task will be approached"
    },
    "estimated_complexity": {
      "type": "string",
      "enum": ["simple", "moderate", "complex"]
    },
    "questions_for_user": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Any questions that must be answered before execution can begin. Empty array if none."
    }
  },
  "required": ["task_understanding", "success_criteria", "approach_summary", "estimated_complexity", "questions_for_user"]
}
SCHEMA
```

- [ ] **Step 2: Create executor output schema**

```bash
cat > /home/agent/schemas/executor-output.json << 'SCHEMA'
{
  "type": "object",
  "properties": {
    "status": {
      "type": "string",
      "enum": ["complete", "question", "partial", "failed"]
    },
    "summary": {
      "type": "string",
      "description": "Human-readable summary of what was accomplished"
    },
    "result": {
      "type": "string",
      "description": "The actual deliverable — research findings, comparison table, file paths, URLs, etc."
    },
    "completed_steps": {
      "type": "array",
      "items": { "type": "string" }
    },
    "failed_step": {
      "type": "string",
      "description": "Which step failed, if any"
    },
    "error": {
      "type": "string",
      "description": "Specific error message if status is failed"
    },
    "recovery_attempted": {
      "type": "string",
      "description": "What recovery was attempted if something failed"
    },
    "question": {
      "type": "string",
      "description": "Question for user if status is question"
    },
    "progress": {
      "type": "string",
      "description": "Summary of work done so far, used to resume after user answers a question"
    },
    "self_review": {
      "type": "object",
      "properties": {
        "criteria_checked": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "criterion": { "type": "string" },
              "met": { "type": "boolean" },
              "evidence": { "type": "string" }
            },
            "required": ["criterion", "met", "evidence"]
          }
        },
        "overall_pass": { "type": "boolean" }
      },
      "required": ["criteria_checked", "overall_pass"]
    },
    "learnings": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Operational learnings from this task execution"
    }
  },
  "required": ["status", "summary"]
}
SCHEMA
```

- [ ] **Step 3: Create reviewer output schema**

```bash
cat > /home/agent/schemas/reviewer-output.json << 'SCHEMA'
{
  "type": "object",
  "properties": {
    "verdict": {
      "type": "string",
      "enum": ["pass", "fail"]
    },
    "criteria_review": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "criterion": { "type": "string" },
          "met": { "type": "boolean" },
          "evidence": { "type": "string", "description": "Specific evidence from the output supporting the judgment" },
          "feedback": { "type": "string", "description": "What specifically needs to change if not met" }
        },
        "required": ["criterion", "met", "evidence"]
      }
    },
    "overall_feedback": {
      "type": "string",
      "description": "Summary feedback for the executor if verdict is fail"
    },
    "quality_score": {
      "type": "integer",
      "minimum": 1,
      "maximum": 10,
      "description": "Overall quality rating of the output"
    }
  },
  "required": ["verdict", "criteria_review", "quality_score"]
}
SCHEMA
```

---

### Task 9: Create System Prompts

**Context:** Each agent role gets a dedicated prompt file stored on the VPS. These are referenced via `--system-prompt-file`.

- [ ] **Step 1: Create planner prompt**

```bash
cat > /home/agent/prompts/planner.md << 'PROMPT'
# Role: Task Planner

You are the planning agent for Usman's Chief of Staff system. Your job is to analyze incoming tasks and define clear, measurable success criteria before any work begins.

## Your Responsibilities

1. Restate the task in your own words to confirm understanding
2. Define 3-7 success criteria that are specific and objectively measurable
3. Identify the approach at a high level
4. Flag any questions that MUST be answered before execution can begin

## Guidelines

- Success criteria should be verifiable by someone who has never seen the task before
- Bad criterion: "Good quality research" — not measurable
- Good criterion: "At least 5 options compared with price, rating, and key specs for each"
- If the task is clear enough to proceed without questions, leave questions_for_user empty
- Consider what the user actually needs, not just what they literally asked for. If someone asks "find flights to Paris" they probably also want to know about layovers, baggage, and total travel time.
- Keep the approach_summary brief — the executor will figure out the details

## Output

Return structured JSON matching the planner-output schema. Nothing else.
PROMPT
```

- [ ] **Step 2: Create executor prompt**

```bash
cat > /home/agent/prompts/executor.md << 'PROMPT'
# Role: Task Executor

You are the execution agent for Usman's Chief of Staff system. You receive a task with confirmed success criteria and must deliver a result that meets all criteria.

## Your Capabilities

- **Browser automation**: Run scripts in /home/agent/scripts/ via Bash
  - `node /home/agent/scripts/browse.js "<url>"` — fetch a URL and get page content as markdown JSON
  - `node /home/agent/scripts/interact.js "<url>" '<actions-json>'` — fill forms, click buttons, take screenshots
  - `node /home/agent/scripts/screenshot.js "<url>" "<output-path>"` — capture a page screenshot
  - `node /home/agent/scripts/session.js <save|restore|list> [name]` — manage browser sessions
  - You can also write custom Patchright scripts for complex browser interactions
- **File system**: Full read/write access to /home/agent/
- **Shell**: Full bash access with sudo
- **Git**: Access to GitHub repos via SSH

## How to Work

1. Break the task into concrete steps
2. Execute each step, checking progress as you go
3. If you need to ask the user something, return status "question" with your question and progress so far
4. When done, self-review against every success criterion with specific evidence
5. If your self-review finds gaps, iterate — do not return incomplete work
6. Return structured JSON matching the executor-output schema

## Loop Detection Rules

- If you attempt the same action with the same approach more than twice and it fails, STOP. Try a fundamentally different approach or return status "partial" with what you have.
- If you are not making measurable progress, return what you have rather than continuing to burn budget.
- It is always better to return a partial result with a clear explanation than to exhaust your budget achieving nothing.

## Error Reporting

When something fails, be specific:
- BAD: "Could not access the website"
- GOOD: "HTTP 403 from amazon.com/dp/B0xyz — likely anti-bot detection. Tried clearing cookies and using mobile user agent, both returned 403."

## Context

Check /home/agent/memory/system-learnings.md for past learnings before starting. It may contain relevant information about sites, approaches, or preferences discovered in previous tasks.

## Output

Return structured JSON matching the executor-output schema. Nothing else.
PROMPT
```

- [ ] **Step 3: Create reviewer prompt**

```bash
cat > /home/agent/prompts/reviewer.md << 'PROMPT'
# Role: Independent Quality Reviewer

You are the quality review agent for Usman's Chief of Staff system. You receive a task, its success criteria, and the executor's output. You must independently evaluate whether the output meets the criteria.

## Critical Rule

You have NOT seen how the work was done. You only see the output. Judge the output on its own merits, not on effort or process.

## How to Review

For each success criterion:
1. State the criterion
2. Determine if it is met (true/false)
3. Provide specific evidence FROM THE OUTPUT that supports your judgment
4. If not met, provide specific, actionable feedback on what needs to change

## Scoring

- 1-3: Major gaps. Multiple criteria unmet. Not usable.
- 4-5: Partial. Some criteria met but significant gaps remain.
- 6-7: Acceptable. Most criteria met, minor gaps.
- 8-9: Good. All criteria met with solid evidence.
- 10: Exceptional. All criteria exceeded.

## Verdict Rules

- **Pass**: All criteria are met (every criterion has met=true) AND quality_score >= 7
- **Fail**: Any criterion is not met OR quality_score < 7

## Be Rigorous

- Do not assume something is correct just because it looks plausible
- If a criterion says "at least 5 options compared" — count them. If there are only 4, it fails.
- If a criterion says "include prices" — check every option has a price. If one is missing, note it.
- Vague or unsupported claims in the output should be flagged

## Output

Return structured JSON matching the reviewer-output schema. Nothing else.
PROMPT
```

- [ ] **Step 4: Create retrospective prompt**

```bash
cat > /home/agent/prompts/retrospective.md << 'PROMPT'
# Role: Task Retrospective Analyst

You analyze completed task performance data to identify patterns and improvement opportunities for the Chief of Staff system.

## Input

You receive:
1. The performance record for the just-completed task
2. The last 10 task performance records
3. The current contents of /home/agent/memory/system-learnings.md

## Your Job

Answer three questions:

1. **Preventable issues**: Did anything go wrong in this task that could be prevented next time? If so, what specific instruction, approach change, or tool improvement would help?

2. **Recurring patterns**: Looking at the last 10 tasks, are there recurring issues? (e.g., same sites blocking, same types of questions being asked, same error patterns)

3. **Prompt improvements**: Should any system prompt be updated? If so, write the specific change and why. Add it to the prompt-change-proposals section of your output.

## Output Format

Return a markdown block to be appended to system-learnings.md:

```markdown
## YYYY-MM-DD: [task summary]
- [Key learning 1]
- [Key learning 2]
- Pattern detected: [if any]
- Prompt change proposed: [if any, otherwise "none"]
```

Be concise. Only write learnings that will be useful for future tasks. Do not repeat learnings already in system-learnings.md.
PROMPT
```

- [ ] **Step 5: Test a planner invocation end-to-end**

```bash
cd /home/agent/workdir && claude -p "Find the best noise-cancelling headphones under 200 EUR. Consider sound quality, comfort, battery life, and ANC performance." \
  --output-format json \
  --dangerously-skip-permissions \
  --system-prompt-file /home/agent/prompts/planner.md \
  --json-schema "$(cat /home/agent/schemas/planner-output.json)" \
  --max-budget-usd 0.50 \
  --model sonnet 2>&1 | head -100
```

Expected: JSON response matching the planner schema with task_understanding, success_criteria array, approach_summary, and estimated_complexity.

---

## Phase 4: Email Identity

### Task 10: Set Up Email and Google Account

**Context:** This task requires Usman's involvement for Hostinger admin panel access and Google Account creation. These are manual steps.

- [ ] **Step 1: Create email address in Hostinger**

Usman: Log into Hostinger admin panel → Email → Create new email:
- Address: `sophia@ptriconsulting.com`
- Set a strong password (save it — the agent will need IMAP credentials)

Note the IMAP settings (typically):
- Server: `imap.hostinger.com`
- Port: 993 (SSL)
- Username: `sophia@ptriconsulting.com`
- Password: (the one you just set)

- [ ] **Step 2: Create Google Account with custom email**

Go to: https://accounts.google.com/signup
- Choose "I prefer to use my current email address"
- Enter: `sophia@ptriconsulting.com`
- Complete verification (check inbox via Hostinger webmail)
- Set a strong password (save it separately)

- [ ] **Step 3: Add agent's GitHub SSH key**

Go to: https://github.com/settings/keys → New SSH key
- Title: "Chief of Staff Agent (VPS)"
- Paste the public key from Task 1, Step 5
- Save

- [ ] **Step 4: Test GitHub SSH from VPS**

```bash
su - agent
ssh -T git@github.com
```

Expected: "Hi username! You've successfully authenticated..."

---

## Phase 5: n8n Workflows

### Task 11: Create the Email Watcher Workflow

**Context:** This is the simplest workflow and needed by other workflows. It watches the agent's email inbox via IMAP and writes verification codes to the inbox directory. Build this first using the n8n MCP tools.

- [ ] **Step 1: Read the n8n SDK reference**

Use the n8n MCP `get_sdk_reference` tool to understand workflow creation patterns.

- [ ] **Step 2: Search for required nodes**

Use `search_nodes` to find: IMAP Email trigger node, Code node, Execute Command node.

- [ ] **Step 3: Get node type definitions**

Use `get_node_types` for each node identified in step 2.

- [ ] **Step 4: Write the Email Watcher workflow**

The workflow should:
1. **IMAP Email trigger**: Watch `sophia@ptriconsulting.com` for new emails
2. **Code node**: Extract verification code from email body using regex patterns for common formats (6-digit codes, URLs with tokens)
3. **Execute Command node**: Write the extracted code to `/home/agent/inbox/<timestamp>.txt`

Credential needed: Create an IMAP credential in n8n for `sophia@ptriconsulting.com` with the Hostinger IMAP settings from Task 9.

- [ ] **Step 5: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true` in settings.

---

### Task 12: Create the Task Intake Workflow

**Context:** The entry point — receives tasks from WhatsApp or webhooks, logs them, and triggers the planner.

- [ ] **Step 1: Search for required nodes**

Use `search_nodes` for: WhatsApp trigger (or use existing WhatsApp setup), Webhook, HTTP Request (for Supabase logging), Execute Command (for Claude Code).

- [ ] **Step 2: Get node type definitions**

Use `get_node_types` for all nodes from step 1.

- [ ] **Step 3: Write the Task Intake workflow**

The workflow should:
1. **Two entry points** (run in parallel, merge into same flow):
   - WhatsApp trigger: receive message from Usman's WhatsApp (352621486096)
   - Webhook: receive HTTP POST with `{ "task": "..." }` body
2. **Code node**: Generate task ID (UUID), normalize the task text from either source
3. **HTTP Request (Supabase)**: Log task to `memories` table with:
   - `slug`: `task-log-<task-id>`
   - `category`: `log`
   - `source`: `n8n`
   - `project`: `system-automation`
   - `topic`: `chief-of-staff`
   - `content`: Task description, status: "started", timestamp
   - Use `predefinedCredentialType: supabaseApi` and `neverError: true`
4. **HTTP Request (Supabase)**: Query Supabase for relevant context (search for related preferences/learnings)
5. **Execute Command**: Call Claude Code planner:
   ```bash
   cd /home/agent/workdir && claude -p "<task + context>" \
     --output-format json \
     --dangerously-skip-permissions \
     --system-prompt-file /home/agent/prompts/planner.md \
     --json-schema "$(cat /home/agent/schemas/planner-output.json)" \
     --max-budget-usd 0.50 \
     --model sonnet
   ```
6. **Code node**: Parse planner JSON output
7. **WhatsApp (Send)**: Send success criteria to Usman for confirmation:
   ```
   New task: [task summary]

   Success criteria:
   1. [criterion 1]
   2. [criterion 2]
   ...

   Reply "go" to confirm or send adjustments.
   ```
8. Store criteria and task ID in workflow static data (for the executor workflow to pick up)

- [ ] **Step 4: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

### Task 13: Create the Executor Workflow

**Context:** Receives confirmed criteria, calls Claude Code executor, handles mid-task questions.

- [ ] **Step 1: Search for and get node types**

Nodes needed: Webhook (for criteria confirmation), Execute Command, Code, WhatsApp Send, HTTP Request (Supabase), IF.

- [ ] **Step 2: Write the Executor workflow**

The workflow should:
1. **Webhook trigger**: Receives confirmation from task intake (task ID, confirmed criteria, task text, context)
2. **Code node**: Build the executor prompt combining:
   - Task description
   - Confirmed success criteria
   - Context from Supabase
   - Contents of `/home/agent/memory/system-learnings.md` (read via Code node's file access or preceding Execute Command)
3. **Execute Command**: Call Claude Code executor:
   ```bash
   cd /home/agent/workdir && claude -p "<prompt>" \
     --output-format json \
     --dangerously-skip-permissions \
     --system-prompt-file /home/agent/prompts/executor.md \
     --json-schema "$(cat /home/agent/schemas/executor-output.json)" \
     --max-budget-usd 1.00 \
     --model sonnet
   ```
4. **Code node**: Parse executor JSON output
5. **IF node** (`typeValidation: "loose"`): Branch on `status`:
   - `"complete"` or `"partial"` → trigger reviewer workflow via webhook
   - `"question"` → send question via WhatsApp, wait for reply, call executor again with progress
   - `"failed"` → send failure notification via WhatsApp, log to Supabase

For the question loop:
6. **WhatsApp Send**: Send the question to Usman
7. **Webhook wait**: Wait for Usman's reply
8. **Loop back**: Call executor again with original task + criteria + progress + user's answer

- [ ] **Step 3: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

### Task 14: Create the Reviewer Workflow

**Context:** Receives executor output, runs independent review, iterates or delivers.

- [ ] **Step 1: Write the Reviewer workflow**

The workflow should:
1. **Webhook trigger**: Receives task ID, task text, criteria, and executor output
2. **Code node**: Build reviewer prompt with ONLY task + criteria + executor's result (not the executor's reasoning or steps)
3. **Execute Command**: Call Claude Code reviewer:
   ```bash
   cd /home/agent/workdir && claude -p "<prompt>" \
     --output-format json \
     --dangerously-skip-permissions \
     --system-prompt-file /home/agent/prompts/reviewer.md \
     --json-schema "$(cat /home/agent/schemas/reviewer-output.json)" \
     --max-budget-usd 0.50 \
     --model sonnet
   ```
4. **Code node**: Parse reviewer JSON, track iteration count
5. **IF node**: Branch on verdict:
   - `"pass"` → trigger delivery workflow
   - `"fail"` AND iterations < 2 → trigger executor workflow again with reviewer's feedback
   - `"fail"` AND iterations >= 2 → deliver best result so far with reviewer's notes

- [ ] **Step 2: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

### Task 15: Create the Delivery Workflow

**Context:** Sends final result to user, stores learnings, updates task log.

- [ ] **Step 1: Write the Delivery workflow**

The workflow should:
1. **Webhook trigger**: Receives task ID, task text, final result, reviewer score, learnings, and the full Claude Code JSON outputs from executor and reviewer invocations
2. **Code node**: Extract performance metrics from Claude Code JSON outputs:
   - `total_cost_usd` from executor output (this is in the Claude Code JSON envelope, not the structured output)
   - `num_turns` from executor output
   - Count of review iterations (passed from reviewer workflow)
   - Count of user questions asked (passed from executor workflow)
   - List of errors and blocked sites from executor's structured output
   - Reviewer's `quality_score`
3. **WhatsApp Send**: Send result to Usman:
   ```
   Task complete: [task summary]
   Quality score: [X/10]

   [result summary — truncated to WhatsApp message limits]

   Full details saved to second brain.
   ```
4. **HTTP Request (Supabase)**: Update task log entry with full performance record:
   - status: "done"
   - Result summary and quality score
   - Performance metrics: cost, turns, review iterations, questions asked
5. **HTTP Request (Supabase)**: Store learnings as a new note if any (`add_note` via Supabase RPC)
6. **Code node**: Write operational learnings to `/home/agent/memory/` files via Execute Command
7. **HTTP Request**: Trigger retrospective workflow webhook with the complete performance record:
   ```json
   {
     "task_id": "<id>",
     "task_summary": "<summary>",
     "outcome": "success|partial|failed",
     "total_cost_usd": 0.45,
     "num_turns": 28,
     "review_iterations": 1,
     "user_questions_asked": 0,
     "errors": [],
     "blocked_sites": [],
     "quality_score": 8,
     "learnings": ["..."]
   }
   ```

- [ ] **Step 2: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

### Task 16: Create the Retrospective Workflow

**Context:** Runs after each task to analyze performance and update system learnings.

- [ ] **Step 1: Write the Retrospective workflow**

The workflow should:
1. **Webhook trigger**: Receives performance record (task_id, outcome, turns, cost, errors, learnings, review_iterations, questions_asked)
2. **Code node**: Read last 10 performance records from Supabase
3. **Execute Command**: Read `/home/agent/memory/system-learnings.md`
4. **Execute Command**: Call Claude Code retrospective:
   ```bash
   cd /home/agent/workdir && claude -p "<performance data + last 10 records + current learnings>" \
     --output-format json \
     --dangerously-skip-permissions \
     --system-prompt-file /home/agent/prompts/retrospective.md \
     --max-budget-usd 0.25 \
     --model haiku
   ```
   Note: Uses haiku model — this is a lightweight analysis task.
5. **Code node**: Parse output, append new learnings to `/home/agent/memory/system-learnings.md`
6. **IF node**: If prompt changes are proposed, append to `/home/agent/memory/prompt-change-proposals.md`

- [ ] **Step 2: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

### Task 17: Create the Weekly Review Workflow

**Context:** Scheduled weekly, aggregates metrics and sends summary.

- [ ] **Step 1: Write the Weekly Review workflow**

The workflow should:
1. **Schedule trigger**: Every Monday at 9:00 AM
2. **HTTP Request (Supabase)**: Query all task log entries from the past 7 days
3. **Code node**: Calculate metrics:
   - Tasks completed
   - Success/partial/failure counts and rates
   - Average cost per task
   - Total user questions asked
   - Total review rejections
   - Compare to previous week's metrics (query those too)
4. **Code node**: Read `/home/agent/memory/prompt-change-proposals.md` for any pending proposals
5. **WhatsApp Send**: Weekly summary:
   ```
   Weekly agent report (Apr 14-21):
   - Tasks completed: X
   - Success rate: X% (vs X% last week)
   - Avg cost/task: $X.XX
   - User questions: X (vs X last week)
   - Review rejections: X
   - Pending prompt changes: X
   [Top learning from the week]
   ```

- [ ] **Step 2: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

### Task 18: Create the n8n Backup Workflow

**Context:** Daily backup of all n8n workflows to GitHub for disaster recovery.

- [ ] **Step 1: Write the Backup workflow**

The workflow should:
1. **Schedule trigger**: Every day at 2:00 AM
2. **HTTP Request (n8n API)**: `GET /api/v1/workflows` using n8n API credential (irS5M5SchH4BxMdB) to fetch all workflows
3. **Code node**: Format workflows as JSON
4. **Execute Command**: Write to file and push to GitHub:
   ```bash
   su - agent -c '
     cd /home/agent/repos/n8n-backups || (mkdir -p /home/agent/repos && cd /home/agent/repos && git clone git@github.com:<repo>.git n8n-backups && cd n8n-backups)
     echo "<workflows-json>" > workflows-backup-$(date +%Y-%m-%d).json
     cp workflows-backup-$(date +%Y-%m-%d).json workflows-latest.json
     git add -A
     git commit -m "Daily backup $(date +%Y-%m-%d)" || true
     git push origin main
   '
   ```

Note: Usman needs to create a GitHub repo for n8n backups first (e.g., `n8n-backups`). The agent's SSH key from Task 1 needs access to this repo.

- [ ] **Step 2: Validate and create the workflow**

Use `validate_workflow` then `create_workflow_from_code`. Set `availableInMCP: true`.

---

## Phase 6: End-to-End Testing

### Task 19: Test the Complete Flow

**Context:** Run a real task through the entire pipeline to verify everything works together.

- [ ] **Step 1: Send a simple test task via webhook**

```bash
curl -X POST https://n8n.srv1016866.hstgr.cloud/webhook/<task-intake-webhook-path> \
  -H "Content-Type: application/json" \
  -d '{"task": "Find the 3 best-rated coffee shops in Luxembourg City on Google Maps. For each, tell me the name, rating, number of reviews, and address."}'
```

- [ ] **Step 2: Verify planner output arrives via WhatsApp**

Check WhatsApp for the success criteria message. Should include clear, measurable criteria like "3 coffee shops identified", "rating included for each", etc.

- [ ] **Step 3: Reply "go" on WhatsApp to confirm criteria**

- [ ] **Step 4: Wait for executor to complete and reviewer to pass**

Monitor n8n execution logs. Check for:
- Executor calling browse.js to fetch Google Maps results
- Executor self-reviewing against criteria
- Reviewer receiving and evaluating the output
- Delivery sending the final result

- [ ] **Step 5: Verify result arrives via WhatsApp**

Check that the final message includes the 3 coffee shops with all requested details and a quality score.

- [ ] **Step 6: Verify retrospective ran**

```bash
ssh agent@46.202.159.171 "cat /home/agent/memory/system-learnings.md"
```

Should contain a new entry about this task.

- [ ] **Step 7: Verify task log in Supabase**

Check the `memories` table for a log entry with the task ID, status "done", and the result summary.

---

### Task 20: Test Error Handling

**Context:** Verify the system handles failures gracefully.

- [ ] **Step 1: Send a task that will partially fail**

```bash
curl -X POST https://n8n.srv1016866.hstgr.cloud/webhook/<task-intake-webhook-path> \
  -H "Content-Type: application/json" \
  -d '{"task": "Find the cheapest PlayStation 5 currently available on Amazon.lu and compare with Amazon.de prices."}'
```

This task is likely to hit anti-bot issues on Amazon, testing the error reporting path.

- [ ] **Step 2: Verify error reporting quality**

The failure notification on WhatsApp should clearly state:
- What was being attempted (browsing Amazon)
- What went wrong (specific HTTP status or block message)
- What recovery was tried

- [ ] **Step 3: Verify the retrospective captures the learning**

```bash
ssh agent@46.202.159.171 "cat /home/agent/memory/system-learnings.md"
```

Should contain a new entry about the Amazon blocking issue.

---

## Phase 7: VPS Backup Configuration

### Task 21: Enable VPS Snapshots

**Context:** Final safety net — Hostinger VPS snapshots for disaster recovery.

- [ ] **Step 1: Enable weekly snapshots in Hostinger**

Usman: Log into Hostinger → VPS → Snapshots → Enable automatic weekly snapshots.

- [ ] **Step 2: Create an initial snapshot manually**

Hostinger → VPS → Snapshots → Create snapshot now. Label it "Chief of Staff agent - initial setup complete".

- [ ] **Step 3: Test the n8n backup workflow**

Manually trigger the n8n backup workflow in n8n. Verify:
- Workflows JSON appears in the GitHub repo
- Git commit was created with today's date

---

## Summary of Manual Steps (Require Usman)

These steps cannot be automated and require Usman's direct action:

1. **Task 2, Step 3**: Run `claude setup-token` on Mac, copy token to VPS
2. **Task 10, Step 1**: Create email in Hostinger admin panel
3. **Task 10, Step 2**: Create Google Account (browser needed)
4. **Task 10, Step 3**: Add SSH key to GitHub
5. **Task 18**: Create GitHub repo for n8n backups
6. **Task 21**: Enable VPS snapshots in Hostinger
7. **Task 11**: Create IMAP credential in n8n for the new email

Everything else can be executed by Claude Code.
