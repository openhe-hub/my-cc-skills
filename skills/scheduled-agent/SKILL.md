---
name: scheduled-agent
description: Set up unattended, scheduled Claude Code jobs using cron or systemd timers with `claude -p` headless mode. Use when the user wants to automate Claude Code to run on a schedule, minimize interruptions, configure allowedTools, write CLAUDE.md rules, set up hooks, or generate complete systemd/cron templates.
argument-hint: "[action] [project-dir]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Scheduled Claude Code Agent Skill

You help users set up **unattended, scheduled Claude Code jobs** using `claude -p` (headless/non-interactive mode) combined with cron or systemd timers.

## Core Concepts

### `claude -p` Headless Mode
The key flags for unattended automation:

| Flag | Purpose |
|------|---------|
| `-p` / `--print` | Non-interactive mode, prints output and exits |
| `--continue` | Resume the most recent session in the current directory |
| `--resume <id>` | Resume a specific session by ID |
| `--allowedTools` | Pre-approve tools so Claude never pauses to ask permission |
| `--model` | Specify model (e.g. `sonnet`, `opus`, `haiku`) |
| `--max-turns N` | Limit agentic turns (default: unlimited — omit if you want it to run to completion) |
| `--output-format` | `text` (default), `json`, or `stream-json` |

**Do NOT set `--max-turns` too small** — hitting the limit causes an error exit. If you want cost control, use `--max-budget-usd` instead.

### Token Cost
Headless mode (`-p`) consumes the **same tokens** as interactive mode — it only removes human confirmation, the underlying agent loop is identical. Control cost with:
- `--model sonnet` or `haiku` instead of `opus`
- `--max-budget-usd 0.5` to cap spending per run
- Specific, numbered prompts to reduce trial-and-error turns

### Why Jobs Interrupt
Interruptions come from:
1. **Missing tool permissions** — fix with `--allowedTools`
2. **Ambiguous prompt** — fix with a clear prompt + `CLAUDE.md`
3. **Auth/network failure** — fix with stable infra + retry wrapper
4. **Explicit limits hit** — don't set `--max-turns` too small
5. **Unexpected errors in the project** — fix with robust hooks

---

## Handling User Requests

If the user says **"$ARGUMENTS"**, interpret it as:

- `setup <dir>` → generate full template for that project directory
- `script` / `bash` → generate `run_claude_job.sh` only
- `systemd` → generate systemd service + timer units only
- `cron` → generate crontab entry only
- `claude.md` → generate a `CLAUDE.md` template only
- `hooks` → generate hooks config only
- `status` → show how to check systemd timer/job status
- Otherwise, interpret the intent and generate the appropriate artifacts

Always ask for the project directory if not provided.

---

## Step-by-Step Setup

### Step 1 — Write the job script

Create `/home/<user>/run_claude_job.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR="/path/to/your/project"
LOG_DIR="/path/to/logs"
mkdir -p "$LOG_DIR"

# ── Environment Setup ──────────────────────────────────────
# Allow running from within another Claude Code session (testing)
unset CLAUDECODE

# Load auth credentials (systemd does NOT inherit terminal env vars)
set -a
source "$HOME/.claude/.env"
set +a

# If claude is installed via fnm/nvm, set up PATH explicitly
export PATH="$HOME/.local/share/fnm:$HOME/.local/bin:$PATH"
eval "$(fnm env)"
# ───────────────────────────────────────────────────────────

cd "$PROJECT_DIR"
timestamp=$(date +"%Y-%m-%d_%H-%M-%S")
logfile="$LOG_DIR/claude_job_$timestamp.log"

# IMPORTANT: Use heredoc via stdin for multi-line prompts.
# Passing multi-line strings as a positional argument breaks
# under systemd and other non-interactive shells.
cat <<'PROMPT' | claude -p \
  --model sonnet \
  --allowedTools "Bash,Read,Edit,Write,Glob,Grep" \
  2>&1 | tee "$logfile"
Your task prompt here. Be specific:
1. What to check or do
2. What to run (tests, lint, etc.)
3. What files to produce or update
4. How to report results
PROMPT
```

```bash
chmod +x /home/<user>/run_claude_job.sh
```

**Key points:**
- **Heredoc via stdin**: Do NOT pass multi-line prompts as a positional argument — it breaks under systemd. Always pipe through `cat <<'PROMPT' | claude -p ...`
- **`unset CLAUDECODE`**: Required if you ever test the script from within an existing Claude Code session, otherwise it refuses to start (nested session detection)
- **`source ~/.claude/.env`**: systemd services run with a minimal environment — auth tokens, API keys, and custom base URLs must be loaded from a file
- **fnm/nvm PATH**: If Claude Code was installed via fnm or nvm, the node binary is not in the default PATH — you must `eval "$(fnm env)"` explicitly
- `--allowedTools` pre-approves tools — include everything your task needs
- `-p` mode **buffers output** until completion — log files will appear empty while running, this is normal
- Log each run to a timestamped file for debugging

---

### Step 2 — Create the auth env file

systemd services run with a minimal environment and **cannot** access your terminal's env vars. You must store credentials in a file:

```bash
touch ~/.claude/.env && chmod 600 ~/.claude/.env
```

Contents of `~/.claude/.env`:

```bash
# For API key auth:
ANTHROPIC_API_KEY=sk-ant-...

# Or for custom/proxy setups:
ANTHROPIC_AUTH_TOKEN=sk-...
ANTHROPIC_BASE_URL=https://your-proxy.example.com
```

**Auth methods**:
- **OAuth login** (`claude login`): Credentials are stored in `~/.claude/` automatically. In this case the env file may not be needed — but systemd still needs a valid `HOME` and access to `~/.claude/`. Test by running the script manually first.
- **API key**: Must be in the env file since systemd won't inherit `ANTHROPIC_API_KEY` from your shell.
- **Custom proxy / third-party key**: Put both `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_BASE_URL` in the env file.

**Security**: `chmod 600` ensures only your user can read this file. Never commit it to git.

---

### Step 3 — Write CLAUDE.md in the project root

`CLAUDE.md` is loaded at the start of every session. Use it for standing rules:

```markdown
# Project Rules for Automated Agent

## Workflow
- Always run `pytest -q` before and after any code change
- Always run `ruff check .` and `ruff format .` before committing
- Only make minimal, focused changes
- Do not touch `docs/legacy/` or `config/production/`
- Append a summary of each run to `AGENT_LOG.md` with timestamp

## On Failure
- If tests fail, summarize the failures before attempting fixes
- Do not retry the same fix more than 2 times
- If blocked, write the blocker to `AGENT_LOG.md` and stop

## On Completion
- Write results to `DAILY_REPORT.md` (append, don't overwrite)
- Include: what was done, what was skipped, and next recommended action
```

---

### Step 4 — Configure hooks (hard constraints)

Hooks run deterministically on Claude Code events, unlike `CLAUDE.md` which is a soft prompt.

Create `.claude/settings.json` in the project:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && ruff format . && ruff check . --fix"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[hook] Bash command approved'"
          }
        ]
      }
    ]
  }
}
```

Use hooks to:
- Auto-run formatter/linter after every file edit
- Block writes to protected directories
- Run tests after edits (PostToolUse on Write/Edit)

---

### Step 5 — Set up systemd timer (recommended over cron)

**`/etc/systemd/system/claude-job.service`**

```ini
[Unit]
Description=Scheduled Claude Code Agent Job
After=network.target

[Service]
Type=oneshot
User=YOUR_USER
WorkingDirectory=/path/to/your/project
EnvironmentFile=/home/YOUR_USER/.claude/.env
ExecStart=/home/YOUR_USER/run_claude_job.sh
StandardOutput=journal
StandardError=journal
```

**`/etc/systemd/system/claude-job.timer`**

```ini
[Unit]
Description=Run Claude Code Agent daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now claude-job.timer
```

Check status and logs:

```bash
systemctl status claude-job.timer
systemctl status claude-job.service
journalctl -u claude-job.service -f
journalctl -u claude-job.service --since "1 hour ago"
```

**`Persistent=true`** means if the machine was off at the scheduled time, it will run once it comes back online.

---

### Alternative: cron

```cron
# Run at 2:00 AM every day
0 2 * * * /home/YOUR_USER/run_claude_job.sh >> /home/YOUR_USER/cron.log 2>&1
```

Edit crontab:
```bash
crontab -e
```

---

### One-shot delayed task: `systemd-run`

For "run once N minutes from now" tasks, use `systemd-run --user` instead of creating persistent timer files:

```bash
systemd-run --user \
  --on-active="5m" \
  --unit=claude-onetime-job \
  --description="One-shot Claude job" \
  /usr/bin/bash /home/YOUR_USER/run_claude_job.sh
```

Check status:
```bash
systemctl --user status claude-onetime-job.timer   # countdown
systemctl --user status claude-onetime-job.service  # execution result
```

The timer and service are **transient** — they auto-clean after completion.

---

## allowedTools Reference

```bash
# Minimal (read + run commands)
--allowedTools "Bash,Read,Glob,Grep"

# Standard (read + edit + run)
--allowedTools "Bash,Read,Edit,Write,Glob,Grep"

# Full (including agent delegation)
--allowedTools "Bash,Read,Edit,Write,Glob,Grep,Agent"
```

Always include `Bash` if your task runs tests, lint, git, or any shell command.

---

## Recommended Prompt Structure

A good headless prompt is specific and self-contained:

```
Check the current repository for issues:
1. Read TODO.md and any FIXME comments in src/
2. Run `pytest -q` and note failures
3. Fix failing tests one at a time (minimal changes only)
4. Re-run tests to confirm fixes
5. Run `ruff check . --fix && ruff format .`
6. Append a summary to DAILY_REPORT.md:
   - Date and time
   - Tests fixed (list)
   - Tests still failing (list)
   - Files changed
   - Next recommended action
Stop if you are unsure about a change — write the uncertainty to DAILY_REPORT.md instead.
```

---

## Full Project Template Layout

```
/your/project/
├── CLAUDE.md                    # standing rules for the agent
├── DAILY_REPORT.md              # append-only run log
├── AGENT_LOG.md                 # blocker / error log
└── .claude/
    └── settings.json            # hooks config

/home/your_user/
├── run_claude_job.sh            # job script
└── logs/
    └── claude_job_TIMESTAMP.log # per-run logs

/etc/systemd/system/
├── claude-job.service
└── claude-job.timer
```

---

## Troubleshooting

These are real errors encountered during testing:

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Input must be provided either through stdin or as a prompt argument` | Multi-line prompt as positional arg breaks under systemd | Use heredoc via stdin: `cat <<'PROMPT' \| claude -p ...` |
| `Not logged in · Please run /login` | systemd doesn't inherit terminal env vars (auth token missing) | Create `~/.claude/.env` with credentials, `source` it in script |
| `Claude Code cannot be launched inside another Claude Code session` | Testing from within Claude Code | Add `unset CLAUDECODE` at top of script |
| Job stops mid-task asking for permission | Tool not in `--allowedTools` | Add missing tool to `--allowedTools` |
| Job exits immediately | `claude` not in PATH | Add fnm/nvm PATH setup: `eval "$(fnm env)"` |
| Log file stays empty while running | `-p` mode buffers output until completion | This is normal — check log after job finishes |
| Job runs but does nothing useful | Prompt too vague | Rewrite prompt with numbered, specific steps |
| systemd timer not firing | Not enabled | `systemctl enable --now claude-job.timer` |
| Claude loops forever | No stopping condition | Add explicit stop condition to prompt and `CLAUDE.md` |

For PATH issues in systemd, add `Environment` in the service unit:

```ini
[Service]
Environment="PATH=/usr/local/bin:/usr/bin:/bin:/home/YOUR_USER/.local/bin:/home/YOUR_USER/.local/share/fnm/node-versions/v24.11.1/installation/bin"
```
