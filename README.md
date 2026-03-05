# My Claude Code Skills

A personal collection of Claude Code Skills.

## Skills

| Skill | Description |
|-------|-------------|
| [air-dgx](skills/air-dgx/) | Manage AIR DGX Spark Cluster — check node/GPU status, submit Slurm jobs, generate sbatch scripts |
| [scheduled-agent](skills/scheduled-agent/) | Set up unattended scheduled Claude Code jobs — generate cron/systemd templates, CLAUDE.md rules, hooks, and job scripts using `claude -p` headless mode |
| [labssh](skills/labssh/) | Manage SSH host book — list, search, connect to lab servers, run remote commands by short name without memorizing IPs |

## Usage

Copy `skills/<skill-name>/SKILL.md` to your project's `.claude/skills/<skill-name>/SKILL.md`.

## Directory Structure

```
skills/<skill-name>/
├── SKILL.md      # skill prompt file
└── README.md     # usage documentation
```
