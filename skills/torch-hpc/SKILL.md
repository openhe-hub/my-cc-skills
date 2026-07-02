---
name: torch-hpc
description: NYU Torch HPC cluster access and Slurm job management. Use when the user needs to connect to NYU HPC, submit GPU jobs, check job status, or manage Slurm workloads on the Torch cluster.
argument-hint: "[action] [options]"
---

# NYU Torch HPC - Slurm Job Management

## Connection

Access to NYU Torch HPC is via SSH control socket multiplexing through a jump host.

**Connection chain:**
```
Local machine → nyu-127 (jump host, nyuair@172.28.32.47) → NYU Torch HPC
```

**SSH command pattern:**
```bash
ssh nyu-127 "ssh <user>-torch '<command>'"
```

The jump host is reachable via the local SSH alias `nyu-127` (`nyuair@172.28.32.47`). On the jump host, the per-user aliases `yf23-torch` and `zl6890-torch` have `ControlMaster auto` and `ControlPersist yes`, multiplexing over `login.torch.hpc.nyu.edu:22`.

**Prerequisites:** The user must have an active SSH control socket on the jump host. Verify with:
```bash
ssh nyu-127 "ls -la /home/nyuair/.ssh/sockets/"
```
If a socket does not exist, instruct the user to open a terminal on the jump host and run `ssh <user>-torch` to establish the master connection.

**Important:** Do not kill the root SSH connection to `nyu-127` — the per-user sockets live on the jump host and rely on the user-managed control master being up.

## HPC Accounts

Both accounts share the same Slurm project and login node.

| SSH Config Name (on jump host) | HPC User | Control Socket (on jump host) | Home Directory |
|--------------------------------|----------|-------------------------------|----------------|
| `yf23-torch` | `yf23` | `/home/nyuair/.ssh/sockets/yf23@login.torch.hpc.nyu.edu-22` | `/home/yf23` |
| `zl6890-torch` | `zl6890` | `/home/nyuair/.ssh/sockets/zl6890@login.torch.hpc.nyu.edu-22` | `/home/zl6890` |

**HPC login node:** `login.torch.hpc.nyu.edu` (round-robin; resolves to `torch-login-b-1` / `torch-login-b-2` etc.)

**Default account:** Use `yf23` (`yf23-torch`) unless the user explicitly specifies `zl6890`.

## Storage / Filesystems

All user-visible filesystems are VAST NFS, mounted from `vast-eth.torch.hpc.nyu.edu`.

| Mount | Env var | Backed up | Auto-purged | Quota / user | Purpose |
|-------|---------|-----------|-------------|--------------|---------|
| `/home/<user>` | `$HOME` | YES | NO | 50 GB / 30K files | Code, dotfiles, small configs |
| `/scratch/<user>` | `$SCRATCH` | NO | **YES** | 5 TB / 5M files | Datasets, checkpoints, conda envs, training output |
| `/archive/<user>` | `$ARCHIVE` | YES | NO | 2 TB / 20K files | Long-term retention of final results |
| `/share/apps` | — | — | — | — | Read-only shared software / modules |
| `/projects` | — | — | — | — | Project-shared workspace |

**Quota check:** `myquota` (at `/share/apps/local/bin/myquota`) prints one-line usage for `/home`, `/scratch`, `/archive`. Do **not** use `quota -s` — it errors on NFS with `Operation not permitted`.

**Usage guidance:**
- Install miniconda / large conda envs in `/scratch/<user>/miniconda3/` — `/home` is only 50 GB and easily blown by a single env.
- Datasets and training outputs belong in `/scratch/<user>/`. `/scratch` is **not backed up** and is **auto-purged** (purge policy not yet documented in this skill — verify before relying on long-term `/scratch` storage).
- Move anything you want to keep permanently into `/archive/<user>/` before it ages out of `/scratch`.
- `/share/apps` holds shared software (e.g. `myquota` lives at `/share/apps/local/bin/`).

## Slurm Account

- **Account:** `torch_pr_769_tandon_advanced`
- **Alternate account:** `users` (limited)

Both HPC users (`yf23` and `zl6890`) belong to `torch_pr_769_tandon_advanced`. Use `--account=torch_pr_769_tandon_advanced` for all job submissions.

## GPU Partitions and Limits

### Tandon Dedicated Partitions (Higher Priority)

| Partition | GPU Type | Nodes | Total GPUs | Group GPU Limit | Group CPU Limit | Priority |
|-----------|----------|-------|------------|-----------------|-----------------|----------|
| `a100_tandon` | A100 80GB | ga[001-043] | 172 | 60 | 1184 | 20000 |
| `h100_tandon` | H100 80GB HBM3 | gh[001-015] | 60 | 60 | 1440 | 12000 |
| `h200_tandon` | H200 | gh[101-130] | 232 | 112 | 1792 | 11000 |

### Public Partitions (Lower Priority)

| Partition | GPU Type | Group GPU Limit | Priority |
|-----------|----------|-----------------|----------|
| `h200_public` | H200 | 24 | 10000 |
| `l40s_public` | L40S | 208 | 20000 |

### Preemptible Partitions (Lowest Priority, jobs may be requeued)

| Partition | GPU Type | Notes |
|-----------|----------|-------|
| `a100` | A100 | PreemptMode=REQUEUE |
| `h100` | H100 | PreemptMode=REQUEUE |
| `h200` | H200 | PreemptMode=REQUEUE |
| `l40s` | L40S | PreemptMode=REQUEUE |

### CPU-Only Partitions

| Partition | Nodes | Notes |
|-----------|-------|-------|
| `cs` | cs[601-786] | 184 nodes, 23552 CPUs |
| `cl` | cl[011-017] | 7 nodes, 896 CPUs |
| `cpu_short` | cs + gl nodes | High priority CPU |

**Note:** Group limits (GrpTRES) are shared across all users in the same QOS, not per-user.

## Node Hardware

| Node Prefix | GPU | GPUs/Node | CPUs/Node | Mem/Node |
|-------------|-----|-----------|-----------|----------|
| ga[001-043] | A100 80GB | 4 | ~77 | ~920 GB |
| gh[001-015] | H100 80GB HBM3 | 4 | 96 | ~1545 GB |
| gh[101-130] | H200 | 8 | 128 | ~2061 GB |
| gl[001-068] | L40S | 4 | 128 | ~513 GB |
| gr[101-102] | RTX 6000 | 8 | 128 | ~1545 GB |

## Job Submission Templates

### Simple Single-GPU Job

```bash
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --account=torch_pr_769_tandon_advanced
#SBATCH --partition=a100_tandon
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --output=%x_%j.out
#SBATCH --mail-type=ALL
#SBATCH --mail-user=zh3510@nyu.edu

# Load modules / activate env as needed
# module load cuda/12.x

echo "Job $SLURM_JOB_ID on $(hostname)"
nvidia-smi

# Your command here
python train.py
```

### Multi-GPU Single-Node Job

```bash
#!/bin/bash
#SBATCH --job-name=multi_gpu
#SBATCH --account=torch_pr_769_tandon_advanced
#SBATCH --partition=h100_tandon
#SBATCH --gres=gpu:4
#SBATCH --cpus-per-task=16
#SBATCH --mem=128G
#SBATCH --time=08:00:00
#SBATCH --output=%x_%j.out
#SBATCH --mail-type=ALL
#SBATCH --mail-user=zh3510@nyu.edu

echo "Job $SLURM_JOB_ID on $(hostname) with $SLURM_GPUS_ON_NODE GPUs"
nvidia-smi

torchrun --nproc_per_node=4 train.py
```

### Array Job (Hyperparameter Sweep / Batch Processing)

```bash
#!/bin/bash
#SBATCH --job-name=sweep
#SBATCH --account=torch_pr_769_tandon_advanced
#SBATCH --partition=a100_tandon
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=02:00:00
#SBATCH --array=0-9
#SBATCH --output=%x_%A_%a.out
#SBATCH --mail-type=ALL
#SBATCH --mail-user=zh3510@nyu.edu

echo "Array Job $SLURM_ARRAY_JOB_ID, Task $SLURM_ARRAY_TASK_ID on $(hostname)"
nvidia-smi

python train.py --config configs/config_${SLURM_ARRAY_TASK_ID}.yaml
```

**Array format options:**
- `--array=0-9` — tasks 0 through 9
- `--array=1,3,5,7` — specific task IDs
- `--array=0-100%5` — 0-100 with max 5 running concurrently

### Multi-Node Distributed Training

```bash
#!/bin/bash
#SBATCH --job-name=distributed
#SBATCH --account=torch_pr_769_tandon_advanced
#SBATCH --partition=h200_tandon
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:8
#SBATCH --cpus-per-task=32
#SBATCH --mem=256G
#SBATCH --time=12:00:00
#SBATCH --output=%x_%j.out
#SBATCH --mail-type=ALL
#SBATCH --mail-user=zh3510@nyu.edu

export MASTER_ADDR=$(scontrol show hostname $SLURM_NODELIST | head -n1)
export MASTER_PORT=29500

srun torchrun \
    --nnodes=$SLURM_NNODES \
    --nproc_per_node=8 \
    --rdzv_id=$SLURM_JOB_ID \
    --rdzv_backend=c10d \
    --rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT \
    train.py
```

## Output File Patterns

| Pattern | Meaning |
|---------|---------|
| `%j` | Job ID |
| `%x` | Job name |
| `%A` | Array job master ID |
| `%a` | Array task ID |
| `%N` | Node name |

## Email Notifications

Always include in `#SBATCH` directives:
```bash
#SBATCH --mail-type=ALL
#SBATCH --mail-user=zh3510@nyu.edu
```

`--mail-type` options: `NONE`, `BEGIN`, `END`, `FAIL`, `REQUEUE`, `ALL`

## Common Slurm Commands

```bash
# Submit a job
sbatch script.sh

# Check job queue
squeue -u yf23

# Cancel a job
scancel <job_id>

# Cancel all your jobs
scancel -u yf23

# Job details
scontrol show job <job_id>

# Check account limits
sacctmgr show associations user=yf23 format=Account%40

# Check partition QOS limits
sacctmgr show qos format=Name%30,GrpTRES%50

# View completed job info
sacct -j <job_id> --format=JobID,JobName,Partition,State,Elapsed,MaxRSS,MaxVMSize,NodeList

# Interactive GPU session
srun --account=torch_pr_769_tandon_advanced --partition=a100_tandon --gres=gpu:1 --time=01:00:00 --pty bash -l

# Node status
sinfo -p a100_tandon
```

## Execution Notes

- All commands must be wrapped in the double-SSH pattern: `ssh nyu-127 "ssh yf23-torch '...'"`
- For multi-line scripts, create the file on the HPC first with heredoc, then sbatch it
- Escape special characters properly when nesting SSH commands (use single quotes inside double quotes)
- Job output files are written to the directory from which `sbatch` was run (default: home directory)
- Clean up test/temporary scripts and output files after use
