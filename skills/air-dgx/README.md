# air-dgx

Claude Code Skill for managing the AIR DGX Spark Cluster.

## Features

- Check cluster node/GPU status
- Submit, cancel Slurm jobs
- Generate sbatch scripts (single GPU, array, multi-node distributed)
- Query job queues and logs

## Cluster Info

- 30 x NVIDIA DGX GB10 (ARM64), 1 GPU per node
- Slurm scheduler, operated via `ssh dgx-login`
- Shared storage: `/CVPR`, `/AML` (2TB each)

## Usage

Copy `skill.md` to your project's `.claude/skills/` directory:

```bash
cp skill.md /path/to/your/project/.claude/skills/air-dgx.md
```

Then trigger it in Claude Code via `/air-dgx`, e.g.:

```
/air-dgx status
/air-dgx jobs
/air-dgx submit /CVPR/myproject/train.sh
```

## Source

Collected from `nyuair@172.28.32.47:~/zhewen/dgx-management`.
