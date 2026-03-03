# scheduled-agent

Claude Code Skill for setting up unattended, scheduled Claude Code jobs.

## Features

- Generate complete job scripts (`run_claude_job.sh`)
- Set up systemd service + timer units (recommended)
- Generate cron entries
- Write `CLAUDE.md` standing rules for the agent
- Configure hooks for hard constraints (auto-format, auto-test, etc.)
- Troubleshoot interrupted or stalled jobs

## Usage

Copy `SKILL.md` to your project's `.claude/skills/` directory:

```bash
cp SKILL.md /path/to/your/project/.claude/skills/scheduled-agent.md
```

Then trigger it in Claude Code via `/scheduled-agent`, e.g.:

```
/scheduled-agent setup /home/me/myrepo
/scheduled-agent script
/scheduled-agent systemd
/scheduled-agent claude.md
/scheduled-agent hooks
/scheduled-agent status
```

## What Gets Generated

| Artifact | Purpose |
|----------|---------|
| `run_claude_job.sh` | Bash script using `claude -p --continue --allowedTools` |
| `claude-job.service` | systemd oneshot service unit |
| `claude-job.timer` | systemd timer (daily schedule, persistent) |
| `CLAUDE.md` | Standing rules loaded every session |
| `.claude/settings.json` | Hooks for hard constraints (auto-lint, auto-test) |

## Key Principles

- Use `claude -p` (headless) — no terminal required
- Use `--continue` to preserve session context across runs
- Use `--allowedTools` to pre-approve tools and avoid mid-run pauses
- Use `CLAUDE.md` for soft rules, hooks for hard constraints
- Do **not** set `--max-turns` too small — omit it to run to completion
- systemd timers are preferred over cron (better logging, `Persistent=true`)
