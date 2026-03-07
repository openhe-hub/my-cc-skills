# My Claude Code Skills

A personal collection of Claude Code Skills.

## Skills

| Skill | Description |
|-------|-------------|
| [air-dgx](skills/air-dgx/) | Manage AIR DGX Spark Cluster — check node/GPU status, submit Slurm jobs, generate sbatch scripts |
| [scheduled-agent](skills/scheduled-agent/) | Set up unattended scheduled Claude Code jobs — generate cron/systemd templates, CLAUDE.md rules, hooks, and job scripts using `claude -p` headless mode |
| [labssh](skills/labssh/) | Manage SSH host book — list, search, connect to lab servers, run remote commands by short name without memorizing IPs |
| [torch-hpc](skills/torch-hpc/) | NYU Torch HPC cluster access and Slurm job management — connect via SSH jump host, submit GPU jobs, check status on A100/H100/H200 partitions |

## Thirdparty Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [converter](https://skills.sh/boshu2/agentops/converter) | Cross-platform skill converter — parse skills into universal bundle format, then convert to Codex, Cursor, etc. | `npx skills add boshu2/agentops@converter -g -y` |

## Usage

Copy `skills/<skill-name>/SKILL.md` to your project's `.claude/skills/<skill-name>/SKILL.md`.

## Directory Structure

```
skills/<skill-name>/
├── SKILL.md      # skill prompt file
└── README.md     # usage documentation
```
