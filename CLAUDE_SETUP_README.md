# Claude Code Installation & Settings Guide

This document contains all custom settings, hooks, rules, and configurations for replicating this Claude Code setup on another machine.

---

## Quick Setup

1. Install Claude Code
2. Copy the sections below to the appropriate locations:
   - **Global settings** → `~/.claude/settings.json`
   - **Global rules** → `~/.claude/rules/context7.md`
   - **Global CLAUDE.md** → `~/.claude/claude.md`
   - **Project settings** → `./.claude/settings.local.json` (in project directory)

3. Restart Claude Code

---

## 1. Global Settings (`~/.claude/settings.json`)

Contains global hooks, model settings, themes, and plugin configuration.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "(echo '#!/bin/bash'; jq -r '\"claude --resume \" + .session_id') > \"$(pwd)/claude-resume.sh\" && chmod +x \"$(pwd)/claude-resume.sh\" 2>/dev/null || true",
            "timeout": 5
          }
        ]
      }
    ]
  },
  "enabledPlugins": {
    "superpowers": false
  },
  "effortLevel": "medium",
  "theme": "dark",
  "agentPushNotifEnabled": true,
  "model": "haiku"
}
```

### Hook Explanation: Stop Hook (Session Resume)

**What it does:** When Claude quits, automatically creates an executable shell script in your project directory.

**Output file:** `./claude-resume.sh`

**How to use:** 
```bash
./claude-resume.sh
```

This resumes your exact Claude session without losing context.

**Technical details:**
- Extracts session ID from Claude's context on exit
- Creates executable script with proper shebang (`#!/bin/bash`)
- Saves to current working directory so you have the command in context
- Timeout: 5 seconds (non-blocking)

---

## 2. Global Rules (`~/.claude/rules/context7.md`)

Custom rules for using Context7 documentation fetching tool. Instructs Claude to always fetch current documentation for libraries and APIs instead of relying on training data.

```markdown
Use the `ctx7` CLI to fetch current documentation whenever the user asks about a library, framework, SDK, API, CLI tool, or cloud service -- even well-known ones like React, Next.js, Prisma, Express, Tailwind, Django, or Spring Boot. This includes API syntax, configuration, version migration, library-specific debugging, setup instructions, and CLI tool usage. Use even when you think you know the answer -- your training data may not reflect recent changes. Prefer this over web search for library docs.

Do not use for: refactoring, writing scripts from scratch, debugging business logic, code review, or general programming concepts.

## Steps

1. Resolve library: `npx ctx7@latest library <name> "<user's question>"` — use the official library name with proper punctuation (e.g., "Next.js" not "nextjs", "Customer.io" not "customerio", "Three.js" not "threejs")
2. Pick the best match (ID format: `/org/project`) by: exact name match, description relevance, code snippet count, source reputation (High/Medium preferred), and benchmark score (higher is better). If results don't look right, try alternate names or queries (e.g., "next.js" not "nextjs", or rephrase the question)
3. Fetch docs: `npx ctx7@latest docs <libraryId> "<user's question>"`
4. Answer using the fetched documentation

You MUST call `library` first to get a valid ID unless the user provides one directly in `/org/project` format. Use the user's full question as the query -- specific and detailed queries return better results than vague single words. Do not run more than 3 commands per question. Do not include sensitive information (API keys, passwords, credentials) in queries.

For version-specific docs, use `/org/project/version` from the `library` output (e.g., `/vercel/next.js/v14.3.0`).

If a command fails with a quota error, inform the user and suggest `npx ctx7@latest login` or setting `CONTEXT7_API_KEY` env var for higher limits. Do not silently fall back to training data.
```

---

## 3. Global CLAUDE.md (`~/.claude/claude.md`)

Global instructions for all projects. Defines your tech stack, architectural patterns, coding standards, and workflow.

```markdown
<!-- https://kirill-markin.com/articles/claude-code-rules-for-ai/ -->

# CLAUDE.md

You are an expert Python software engineer and systems architect. Always provide direct, technical responses — no filler, no apologies. You operate in a disposable headless Ubuntu VM via SSH/tmux.

## Stack

- **Frontend:** Next.js 15 (App Router), TypeScript (strict), Tailwind, Shadcn/ui, Zustand, Recharts
- **Backend:** FastAPI + Pydantic, SQLAlchemy (Core for raw queries, ORM for CRUD), Uvicorn
- **DB:** PostgreSQL — schema changes via Alembic only
- **Jobs:** Celery + Redis + Celery Beat
- **Auth:** API key stub

## Layout (monorepo)

- `backend/` — FastAPI app (`app/`) + Celery workers (`workers/`)
- `frontend/` — Next.js app
- `scripts/` — operational scripts
- `systemd/` — unit files
- `tests/` — pytest
- `docs/` — docs and logs
- `delete/` — soft deleted files/folders
- `logs/` - all logging

Data flow, never shortcut: backend `router → service → model → DB`; frontend `page → store → API client → backend`.

## Environment

Services run as systemd units. Next.js → FastAPI via `next.config` rewrites. Ports: Next.js 3000, FastAPI 8000, Postgres 5432, Redis 6379.

**Python venv (critical)**: ALWAYS use explicit binary paths (e.g., .venv/bin/python, .venv/bin/pip). NEVER use source .venv/bin/activate or chain commands with &&, as this triggers shell evaluation safety blockers. Never install globally. Update requirements.txt after installs.

## Rules

- TypeScript strict — no `any`, no unchecked casts. Python type hints on all signatures.
- ES modules only. Functional React components with explicit return types.
- Handle async errors explicitly — no swallowed exceptions.
- Structured logging only — no `console.log` / `print()` in production paths.
- Next.js API routes are a BFF proxy only — no business logic.
- Frontend API client is generated from the OpenAPI spec — never hand-written.
- Conventional commits (`feat:`, `fix:`, etc.). Branch per task — never commit to `main`.
- No Docker, no SaaS task queues, no credentials in committed files (`.env`, gitignored).
- Prefer explicit over clever — readable code over terse code
- **No Multi-line Quoted Commands:** Never execute shell commands where a quoted argument contains a newline followed by a hash character (`\n#`) to avoid triggering security warnings. For Python scripts (`python3 -c`), omit inline comments completely. For complex scripts, write to a temporary file in ./temp directory before execution.

## Workflow

- Plan non-trivial features before coding. For tasks >~30 min, write a plan (scope, files, deps, risks), architecture recommendation and options, data model, and wait for approval.
- Scaffold structure before writing implementation
- Verify library versions before assuming compatibility — environment runs recent releases (Python 3.14, Postgres 18). Use Context7 for current docs.
- New dependency → ask first. Ambiguity → state the assumption, proceed, flag it. Uncertainty → make the conservative choice and say why. Corrected → suggest a rule for CLAUDE.md.

## Boundaries

- Read-only commands (`ls`, `cat`, `pytest`) — run freely.
- Destructive commands (`rm -rf`, dropping tables) or anything touching external systems — ask first. Always prefer to move folders, files to delete folder instead of deleting. For db objects, rename to delete_
- Pre-approved Executions: You have explicit permission to execute local Python scripts using the .venv/bin/python direct path without asking for user confirmation. Treat local script execution as a "Read-only" level command unless it modifies the database.

## Documentation

- Fully document all code using standard Python docstrings.

## Logging

- Keep `README.md` current (architecture including system design picture and data model, setup, run commands). 
- After significant work, append a timestamped entry to `docs/claude_session.log`: request, files changed, commands run, outcome. 
- After every significant interaction, architecture decision, or execution of a complex task, you MUST append a timestamped entry to `docs/claude_session.log`.
- The log entry should briefly summarize: The user's request, the files modified, the terminal commands you executed, and the outcome.
```

---

## 4. Project-Specific Settings (`./.claude/settings.local.json`)

Place this in your project directory (`.claude/settings.local.json`). Contains project-specific permissions for tools and skills.

```json
{
  "permissions": {
    "allow": [
      "Skill(update-config)",
      "Bash(gh --version)",
      "Bash(npx --version)",
      "Read(//usr/local/bin/**)",
      "Bash(gh auth *)",
      "Read(//root/.claude/**)",
      "Read(//root/.config/claude/**)",
      "Read(//root/**)",
      "Bash(python3 -c \"import json,sys; d=json.load\\(sys.stdin\\); [print\\(k, d[k].get\\('enabled'\\)\\) for k in d if 'context7' in k.lower\\(\\)]\")",
      "Bash(npx ctx7@latest *)",
      "Skill(playwright-cli)"
    ]
  }
}
```

### Permission Rules Explained

- `Skill(update-config)` — Allow configuration changes
- `Bash(gh --version)` & `Bash(gh auth *)` — GitHub CLI operations
- `Bash(npx --version)` & `Bash(npx ctx7@latest *)` — Context7 documentation fetching
- `Read(...)` rules — Allow reading files from various directories
- `Skill(playwright-cli)` — Allow Playwright browser automation tests

---

## 5. Enabled Skills & Features

### Built-in Skills Available
- `find-docs` — Find and retrieve documentation
- `ui-ux-pro-max` — UI/UX design assistance
- `playwright-cli` — Browser automation and testing

### Plugin Configuration
- `superpowers` plugin: **disabled**

### Model & Performance Settings
- **Default model:** Haiku (fast inference)
- **Effort level:** Medium (balanced quality)
- **Theme:** Dark mode
- **Notifications:** Agent push notifications enabled

---

## 6. Custom GitHub Naming Convention

From auto-memory: When creating new GitHub repositories, always prefix them with `claude-` (e.g., `claude-myproject`).

---

## 7. Installation Instructions

### Step 1: Create Global Settings Directory
```bash
mkdir -p ~/.claude/rules
```

### Step 2: Save Global Settings
Create `~/.claude/settings.json` with the JSON from section 1.

### Step 3: Save Global Rules
Create `~/.claude/rules/context7.md` with the content from section 2.

### Step 4: Save Global CLAUDE.md
Create `~/.claude/claude.md` with the content from section 3.

### Step 5: Configure Project-Specific Settings
In your project directory, create `.claude/settings.local.json` with the JSON from section 4.

### Step 6: Restart Claude Code
```bash
# Kill any running Claude sessions
pkill -f "claude"

# Restart
claude
```

### Step 7: Verify Setup
Check that your settings loaded correctly:
```bash
cat ~/.claude/settings.json
```

---

## 8. Session Resume Workflow

After quitting Claude, a `claude-resume.sh` script is automatically created in your project directory.

**To resume your session:**
```bash
./claude-resume.sh
```

The script contains your session ID and automatically reconnects you to the exact same conversation context.

---

## 9. Notes for Replication

- **Secrets:** The `.credentials.json` file contains authentication tokens and should NOT be shared. Generate new credentials on the new machine via `claude auth login`.
- **Project memory:** Auto-memory is stored per-project in `~/.claude/projects/<project-id>/memory/`. Copy these if you want to preserve project-specific knowledge across machines.
- **Model selection:** Haiku is set as default for speed. Change to `sonnet` or `opus` in settings.json if you need higher reasoning capability.
- **Context7 API:** If Context7 fails with quota errors, set `CONTEXT7_API_KEY` environment variable or run `npx ctx7@latest login`.

---

## 10. Customization Guide

### Change Default Model
Edit `~/.claude/settings.json`:
```json
{
  "model": "sonnet"  // or "opus" for maximum capability
}
```

### Change Theme
```json
{
  "theme": "light"  // or "auto", "light-daltonized", "dark-daltonized"
}
```

### Add New Permissions
Edit `.claude/settings.local.json` and add to the `permissions.allow` array.

### Modify Stop Hook
Edit the `hooks.Stop` section in `~/.claude/settings.json` to customize what happens when Claude quits.

---

## 11. Troubleshooting

**Settings not loading?**
- Verify JSON syntax: `jq . ~/.claude/settings.json`
- Check file permissions: `ls -la ~/.claude/settings.json`
- Restart Claude Code

**Hooks not running?**
- Check syntax with: `jq -e '.hooks.Stop' ~/.claude/settings.json`
- Verify the hook command works manually
- Check Claude Code logs: `tail -f ~/.claude/debug/*.log` (if debug directory exists)

**Resume script not created?**
- Verify `jq` is installed: `which jq`
- Check write permissions in project directory: `ls -ld .`
- Manually test the hook command with a dummy session ID

---

Generated: 2026-06-03
Last updated: 2026-06-03
