---
name: air-dgx
description: Manage the AIR DGX Spark Cluster — check node/GPU status, submit Slurm jobs, write sbatch scripts, and query job queues. Use when the user asks about the DGX cluster, Slurm, GPU availability, or wants to run jobs on the HPC.
argument-hint: "[action] [args...]"
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# AIR DGX Spark Cluster Skill

You are managing the AIR DGX Spark Cluster via the login node `dgx-login`.
All remote commands should be run via `ssh dgx-login "<command>"`.

## SSH Routing

- Primary login alias: `dgx-login`.
- `nyu-127` and `nyu-118` are on the same internal network as `dgx-login`.
- If direct local access to `dgx-login` is slow or unavailable, first SSH to `nyu-127` or `nyu-118`, then run `ssh dgx-login` from there for an interactive DGX login.
- For one-shot commands through a relay host, use a nested SSH command such as:

```bash
ssh nyu-127 'ssh dgx-login "sinfo"'
ssh nyu-118 'ssh dgx-login "squeue -u cvpr"'
```

- Treat `nyu-127` and `nyu-118` as relay hosts only; Slurm commands and DGX filesystem work should still run on `dgx-login`.

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
| `/media/cvpr` | Large shared CVPR mount | `//rcsfileshare.abudhabi.nyu.edu/mmvc-large` |
| `/media/aml` | Large shared AML mount | `//rcsfileshare.abudhabi.nyu.edu/mmvc-large-2` |

Always store code, data, logs, and final Python environments on shared storage,
not under `/home` or persistent local node paths. It is OK, and often necessary,
to use compute-node `/tmp` as temporary build/cache space while creating an
environment, then copy the finished environment to shared storage.

### Conda and Python Environments

The login node and compute nodes differ in important ways. In the current validated state, `dgx-login` is `x86_64`, while Slurm compute nodes are `aarch64` DGX GB10 machines. Do not create or install conda/Python environments directly on `dgx-login`; packages and binaries built there can be unusable on compute nodes.

Default priority path for user environments:

```bash
/media/cvpr/zhewen/envs/<name>_env
```

When setting up or repairing a conda environment:

1. SSH to `dgx-login`.
2. Allocate a compute node with Slurm.
3. Create or update the environment on that compute node, preferably in a
   temporary path under `/tmp`.
4. Copy the finished environment directory to
   `/media/cvpr/zhewen/envs/<name>_env` or the user-specified shared-storage
   prefix.
5. Validate imports and CUDA with `srun` on a compute node before using the
   environment in jobs.

Example interactive setup flow:

```bash
ssh dgx-login
mkdir -p /media/cvpr/zhewen/envs
srun -p spark -N1 -n1 --cpus-per-task=4 --mem=16G --gres=gpu:1 --time=02:00:00 --pty bash -l

# Now running on a compute node, not dgx-login:
export ENV_PREFIX=/media/cvpr/zhewen/envs/<name>_env
export TMP_ENV=/tmp/<name>_env-$USER
export CONDA_PKGS_DIRS=/tmp/<name>-conda-pkgs-$USER
export PIP_CACHE_DIR=/tmp/<name>-pip-cache-$USER

rm -rf "$TMP_ENV"
python -m conda create -y -p "$TMP_ENV" python=3.10 pip
"$TMP_ENV/bin/python" -m pip install <packages>

rm -rf "$ENV_PREFIX"
mkdir -p "$ENV_PREFIX"
(
  cd "$TMP_ENV"
  tar -chf - .
) | (
  cd "$ENV_PREFIX"
  tar --no-same-owner --no-same-permissions --touch -xf -
)

"$ENV_PREFIX/bin/python" -c "import platform; print(platform.machine())"
```

If an environment is built or downloaded as a directory named `<name>_env/`, place that directory at `/media/cvpr/zhewen/envs/<name>_env/` and validate it via Slurm. Use the explicit interpreter path in repeated checks and jobs, for example:

```bash
/media/cvpr/zhewen/envs/<name>_env/bin/python -c "import torch; print(torch.__version__)"
```

#### Robust Environment Bootstrap Pattern

Use this pattern when conda or pip behaves strangely on `/media/cvpr`,
`/media/aml`, `/CVPR`, or `/AML`.

- The final env should live on shared storage, but do the actual conda/pip
  install in compute-node `/tmp`.
- Keep `CONDA_PKGS_DIRS`, `PIP_CACHE_DIR`, and other build caches on `/tmp`.
- Use explicit Python paths such as `"$ENV_PREFIX/bin/python"` rather than
  relying on shell activation.
- If the base `conda` script has a broken shebang, call conda through Python:

```bash
/path/to/miniforge/bin/python -m conda create -y -p "$TMP_ENV" python=3.10 pip
```

- Copy the finished env with `tar -h` so conda symlinks are dereferenced:

```bash
(
  cd "$TMP_ENV"
  tar -chf - .
) | (
  cd "$ENV_PREFIX"
  tar --no-same-owner --no-same-permissions --touch -xf -
)
```

- CIFS metadata errors such as the following can be non-fatal if `bin/python`
  exists and a compute-node import/CUDA validation passes:

```text
tar: .: Cannot change mode to ...: Operation not permitted
```

- Do not accept the environment as ready just because copying finished. Always
  run a fresh `srun` validation on a compute node.

#### Shared-Storage Pitfalls

The shared mounts are CIFS-backed and can reject operations that work on local
Linux filesystems.

- Direct `conda create -p /media/...` may fail on symlinks.
- Direct `pip install` into `/media/...` may fail when pip creates temporary
  files in `site-packages`.
- `rsync` may fail because it creates hidden dot-temp files.
- Metadata operations such as `chmod`, `utime`, and ownership restoration may
  fail. Prefer `tar --no-same-owner --no-same-permissions --touch`.
- The mount can temporarily return `Host is down`. If that happens, stop
  writing to it, verify the mount with `df -h` and `ls -ld`, then resume after
  it recovers.

When reducing env size before copying, pruning package tests and caches is OK,
but be conservative. Do not remove runtime-imported internals just because a
directory name looks like tests; for example, some PyTorch versions import
`torch.testing._internal` during normal runtime paths.

#### Package Compatibility Checks

For GPU/compiled Python packages on DGX:

- Confirm wheels are `linux_aarch64`/ARM-compatible before installing.
- Prefer CUDA 13-compatible wheels for GB10 nodes when available.
- Validate compiled packages with real imports on a compute node, not on
  `dgx-login`.
- If an import fails after install, check ABI mismatches such as NumPy 1.x vs
  2.x before reinstalling the whole environment.
- For ONNX export stacks, check exporter-time dependencies separately from
  simple imports; a package may import successfully but fail during model
  export because a helper package is missing.

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
ssh dgx-login "df -h /media/cvpr /media/aml"
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
source /media/cvpr/zhewen/envs/<env>_env/bin/activate
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
source /media/cvpr/zhewen/envs/<env>_env/bin/activate
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
source /media/cvpr/zhewen/envs/<env>_env/bin/activate
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
- **Python environments** should be created on compute nodes and should live under `/media/cvpr/zhewen/envs/<name>_env/` by default.
- **Do not configure conda on `dgx-login`** for DGX jobs; use Slurm to enter a compute node first, then create/update the env.
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
