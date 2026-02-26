---
name: air-dgx
description: Manage the AIR DGX Spark Cluster — check node/GPU status, submit Slurm jobs, write sbatch scripts, and query job queues. Use when the user asks about the DGX cluster, Slurm, GPU availability, or wants to run jobs on the HPC.
argument-hint: "[action] [args...]"
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# AIR DGX Spark Cluster Skill

You are managing the AIR DGX Spark Cluster via the login node `dgx-login`.
All remote commands should be run via `ssh dgx-login "<command>"`.

## Cluster Overview

- **Cluster Name:** spark-cluster
- **Login Node:** `dgx-login` (10.224.16.131, user: cvpr)
- **Compute Nodes:** 30 x NVIDIA DGX GB10 (ARM64 / aarch64)
- **Scheduler:** Slurm 23.11.4
- **Partition:** `spark` (default, only partition)

### Per Node Specs

| Resource | Value |
|----------|-------|
| CPU | 20 cores (aarch64) |
| Memory | 110 GB |
| GPU | 1 x NVIDIA GB10 (Grace Blackwell) |
| CUDA | 13.0 |
| Driver | 580.95.05 |

### Cluster Totals

- 30 nodes, 600 CPUs, ~3.3 TB RAM, 30 GPUs

## Shared Storage

Both mounts are available on the login node AND all compute nodes:

| Mount | Size | Source |
|-------|------|--------|
| `/CVPR` | 2 TB | `//rcsfileshare.abudhabi.nyu.edu/CVPR` |
| `/AML` | 2 TB | `//rcsfileshare.abudhabi.nyu.edu/AML` |

Always store code, data, logs, and Python environments on `/CVPR` or `/AML`.

## Common Actions

When the user asks you to do something on the cluster, use `ssh dgx-login "<cmd>"`.

### Check Status

```bash
# Node & GPU availability
ssh dgx-login "sinfo"
ssh dgx-login "sinfo -N -l"

# Job queue
ssh dgx-login "squeue"
ssh dgx-login "squeue -u cvpr"

# Disk usage
ssh dgx-login "df -h /CVPR /AML"
```

### Submit Jobs

```bash
ssh dgx-login "sbatch /CVPR/<path>/job.sh"
```

### Cancel Jobs

```bash
ssh dgx-login "scancel <job_id>"
ssh dgx-login "scancel -u cvpr"          # cancel all
ssh dgx-login "scancel --array=0-3 <id>" # cancel array tasks
```

### Check Job Output

```bash
ssh dgx-login "cat /CVPR/<path>/logs/<name>_<jobid>.out"
ssh dgx-login "tail -f /CVPR/<path>/logs/<name>_<jobid>.out"
```

## Sbatch Script Templates

When writing sbatch scripts, follow these conventions:

### Single GPU Job

```bash
#!/bin/bash
#SBATCH --job-name=<name>
#SBATCH --output=/CVPR/<dir>/logs/%x_%j.out
#SBATCH --error=/CVPR/<dir>/logs/%x_%j.err
#SBATCH --partition=spark
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --gres=gpu:1
#SBATCH --mem=100G
#SBATCH --time=24:00:00

export PYTHONUNBUFFERED=1
source /CVPR/python_envs/<env>/bin/activate
cd /CVPR/<dir>/<project>

echo "Job $SLURM_JOB_ID on $(hostname) started at $(date)"
python train.py
echo "Done at $(date)"
```

### Array Job (Parallel Multi-GPU)

Use `--array` to launch independent tasks in parallel on separate nodes/GPUs.

```bash
#!/bin/bash
#SBATCH --job-name=<name>
#SBATCH --output=/CVPR/<dir>/logs/%x_%A_%a.out
#SBATCH --error=/CVPR/<dir>/logs/%x_%A_%a.err
#SBATCH --partition=spark
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --gres=gpu:1
#SBATCH --mem=100G
#SBATCH --time=24:00:00
#SBATCH --array=0-3

export PYTHONUNBUFFERED=1
source /CVPR/python_envs/<env>/bin/activate
cd /CVPR/<dir>/<project>

echo "Array Task $SLURM_ARRAY_TASK_ID on $(hostname)"
python train.py --fold $SLURM_ARRAY_TASK_ID
```

Array syntax: `0-3` (4 tasks), `0-29` (all 30 nodes), `0-19%5` (20 tasks, max 5 concurrent).

### Multi-Node Distributed Job

```bash
#!/bin/bash
#SBATCH --job-name=<name>
#SBATCH --output=/CVPR/<dir>/logs/%x_%j.out
#SBATCH --error=/CVPR/<dir>/logs/%x_%j.err
#SBATCH --partition=spark
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=16
#SBATCH --gres=gpu:1
#SBATCH --mem=100G
#SBATCH --time=24:00:00

export PYTHONUNBUFFERED=1
source /CVPR/python_envs/<env>/bin/activate
cd /CVPR/<dir>/<project>

srun python -m torch.distributed.launch \
    --nproc_per_node=1 \
    --nnodes=$SLURM_NNODES \
    --node_rank=$SLURM_NODEID \
    --master_addr=$(scontrol show hostnames $SLURM_JOB_NODELIST | head -n 1) \
    --master_port=29500 \
    train.py
```

## Key Slurm Environment Variables

| Variable | Description |
|----------|-------------|
| `$SLURM_JOB_ID` | Job ID |
| `$SLURM_NODELIST` | Allocated node(s) |
| `$SLURM_NNODES` | Number of nodes |
| `$SLURM_NODEID` | Node rank (multi-node) |
| `$SLURM_ARRAY_JOB_ID` | Parent array job ID |
| `$SLURM_ARRAY_TASK_ID` | Array task index |

## Important Notes

- **Architecture is ARM64 (aarch64)** — all pip packages and compiled binaries must be ARM-compatible.
- **Max per node:** 20 CPUs, 110 GB RAM, 1 GPU. Never request more.
- **Python environments** should live on `/CVPR/python_envs/` for shared access.
- **Logs** must go to `/CVPR` or `/AML` (not `/home` or `/tmp`) to be readable from the login node.
- **Output patterns:** `%j` = job ID, `%x` = job name, `%A` = array job ID, `%a` = array task ID.

## Handling User Requests

- If the user says **"$ARGUMENTS"**, interpret it as a cluster action:
  - `status` / `info` → run `sinfo` and `squeue`
  - `jobs` / `queue` → run `squeue -u cvpr`
  - `nodes` → run `sinfo -N -l`
  - `disk` → run `df -h /CVPR /AML`
  - `submit <path>` → run `sbatch <path>`
  - `cancel <id>` → run `scancel <id>`
  - `log <path>` → read the log file
  - Otherwise, interpret the intent and act accordingly
