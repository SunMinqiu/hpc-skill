---
name: darwin
description: Manage jobs on UD DARWIN HPC cluster via SSH (submit, status, jupyter, cancel)
allowed-tools: Bash, Read
argument-hint: "e.g., 'check my job status', 'run python train.py on gpu-v100 for 4 hours with 2 GPUs'"
---

# DARWIN Cluster Management

You are helping the user manage jobs on the University of Delaware DARWIN HPC cluster remotely via SSH.

## Connection Info

- **Host**: `darwin.hpc.udel.edu`
- **User**: `your_username`
- **Workgroup**: `your_workgroup`
- **Home**: `/home/your_username`
- **Lustre Work**: `/lustre/your_workgroup/users/your_username`

## User Input

`$ARGUMENTS`

## Pre-flight Check (MUST run first)

Before executing ANY action, verify SSH connectivity to the cluster:

```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes darwin.hpc.udel.edu "echo ok" 2>&1
```

- If output contains `ok` → proceed to Action Router
- If it fails → stop immediately and report the issue to the user with diagnosis:
  - `Connection refused` / `Connection timed out` → network issue or host unreachable
  - `Permission denied` → SSH key not configured or not authorized
  - `Could not resolve hostname` → DNS issue or no network
  - `No such file or directory` (ssh) → SSH client not installed
  - Other errors → show the raw error message

  Suggest the user check:
  1. SSH key is set up: `ssh-keygen` then add public key to the cluster
  2. SSH config exists in `~/.ssh/config` for the host
  3. Network connectivity: `ping darwin.hpc.udel.edu`
  4. VPN if required by the institution

  **Do NOT proceed with any cluster operation if this check fails.**

---

## Action Router

Parse `$ARGUMENTS` and route to the appropriate action.

### If $ARGUMENTS is empty or "help", show this menu:

```
DARWIN Cluster — What You Can Say
=================================

Job Submission:
  "run python train.py on gpu-v100 for 4 hours with 2 GPUs"
  "run python preprocess.py on standard for 2 hours with 64G memory"
  "submit my_experiment.qs"

Jupyter Notebook:
  "launch jupyter on gpu-v100 for 8 hours"
  "start jupyter on gpu-t4 for 4 hours"
  "get the tunnel command for job 12345"

Job Management:
  "check my job status"
  "show log for job 12345"
  "cancel job 12345"
  "show details for job 12345"

Resources:
  "show available partitions"
  "list my slurm scripts"
  "check disk usage"

Available Partitions:
  CPU:
  - standard     : 64 cores, 512 GiB, max 7d
  - large-mem    : 64 cores, 1024 GiB, max 7d
  - xlarge-mem   : 64 cores, 2048 GiB, max 7d
  - extended-mem : 64 cores, 1024 GiB + 2.73 TiB NVMe, max 7d

  GPU:
  - gpu-t4       : 64 cores, 1x T4 (16 GB), max 7d
  - gpu-v100     : 48 cores, 4x V100 (32 GB), max 7d (default for GPU)
  - gpu-mi50     : 64 cores, 1x MI50, max 7d
  - gpu-mi100    : 64 cores, 1x MI100, max 7d

  Special:
  - idle         : All node types, preemptible, no charge
```

---

## Partition Configuration

| Name | SLURM Partition | Cores | Memory | GPUs | Max Time |
|------|-----------------|-------|--------|------|----------|
| standard | standard | 64 | 499.7 GiB | - | 7-00:00:00 |
| large-mem | large-mem | 64 | 999.4 GiB | - | 7-00:00:00 |
| xlarge-mem | xlarge-mem | 64 | 2031.6 GiB | - | 7-00:00:00 |
| extended-mem | extended-mem | 64 | 999.4 GiB | - | 7-00:00:00 |
| gpu-t4 | gpu-t4 | 64 | 491.5 GiB | 1x T4 | 7-00:00:00 |
| gpu-v100 | gpu-v100 | 48 | 737.3 GiB | 4x V100 | 7-00:00:00 |
| gpu-mi50 | gpu-mi50 | 64 | 491.5 GiB | 1x MI50 | 7-00:00:00 |
| gpu-mi100 | gpu-mi100 | 64 | 491.5 GiB | 1x MI100 | 7-00:00:00 |
| idle | idle | varies | varies | varies | 7-00:00:00 |

**Default GPU partition**: `gpu-v100`
**Default CPU partition**: `standard`
**Default time**: 30 minutes (if not specified)
**Default memory**: 1G per core (if not specified)

---

## Important Notes

- **Workgroup**: The `workgroup` command spawns a new subshell, so `workgroup -g your_workgroup && cmd` does NOT work (`cmd` runs after the workgroup shell exits). Instead, **pipe commands into workgroup**: `echo "cmd" | workgroup -g your_workgroup`. For job submission, use a two-step approach: (1) write the script to a file, (2) pipe `sbatch` into workgroup.
- **GPU directive**: Use `--gpus=<count>` only. Do NOT use `--gres` on DARWIN.
- **VALET**: For scripts that need VALET but lack `#!/bin/bash -l`, add `source /etc/profile.d/valet.sh` before `vpkg_require` commands.
- **Templates**: System templates are at `/opt/shared/templates/slurm/generic/`
- **Lustre**: Use `$WORKDIR` or `/lustre/your_workgroup/users/your_username` for job data, NOT home directory.

---

## Actions

### RUN - Template Mode (Default)

Trigger: user wants to run/execute a command on the cluster (e.g., "run python train.py on gpu-v100 for 4 hours with 2 GPUs")

Extract from natural language:
- **partition**: look for "on <partition>" (default: gpu-v100 for GPU jobs, standard for CPU jobs)
- **time**: look for "for <duration>" — convert to HH:MM:SS or D-HH:MM:SS (default: 1:00:00)
- **gpus**: look for "with <N> GPUs" (default: 1, only for gpu-* partitions)
- **nodes**: look for "<N> nodes" (default: 1)
- **cpus**: look for "<N> CPUs/cores" (default: 4 for GPU jobs, 1 for CPU jobs)
- **memory**: look for "with <N>G memory" (default: 8G for GPU jobs, 4G for CPU jobs)
- **job name**: look for "named <name>" (default: claude-job)
- **command**: the program/script the user wants to run

Auto-detect partition type: if user provides a gpu-* partition or uses `-g`, treat as GPU job.

Submit using two-step approach (write script file, then pipe sbatch into workgroup):

Step 1 — Write the script to a file:
```bash
ssh darwin.hpc.udel.edu "cat > /lustre/your_workgroup/users/your_username/<job_name>.qs << 'SLURM_SCRIPT'
#!/bin/bash -l
#SBATCH --job-name=<job_name>
#SBATCH --partition=<partition>
#SBATCH --nodes=<nodes>
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=<cpus>
#SBATCH --mem=<memory>
#SBATCH --time=<time>
#SBATCH --output=slurm-%j.out
#SBATCH --error=slurm-%j.err
<if GPU job: #SBATCH --gpus=<gpus>>

cd \$SLURM_SUBMIT_DIR

<user_command>
SLURM_SCRIPT"
```

Step 2 — Submit via workgroup:
```bash
ssh darwin.hpc.udel.edu 'echo "cd /lustre/your_workgroup/users/your_username && sbatch /lustre/your_workgroup/users/your_username/<job_name>.qs" | workgroup -g your_workgroup'
```

---

### SUBMIT - Script Mode

Trigger: user wants to submit an existing SLURM script (e.g., "submit my_experiment.qs")

```bash
ssh darwin.hpc.udel.edu 'echo "cd /lustre/your_workgroup/users/your_username && sbatch <script>" | workgroup -g your_workgroup'
```

---

### JUPYTER - Launch Jupyter Notebook

Trigger: user wants to launch/start jupyter (e.g., "launch jupyter on gpu-v100 for 8 hours")

Default partition: `gpu-v100`

Partition mapping:
- `gpu-t4` / `t4` → gpu-t4, port 8891, default 4:00:00, gpus 1
- `gpu-v100` / `v100` → gpu-v100, port 8892, default 8:00:00, gpus 1
- `gpu-mi100` / `mi100` → gpu-mi100, port 8893, default 4:00:00, gpus 1
- `standard` / `cpu` → standard, port 8894, default 4:00:00, no GPU
- `idle` → idle, port 8895, default 4:00:00, no GPU (preemptible, no charge)

Submit Jupyter using two-step approach:

Step 1 — Write the Jupyter script to a file:
```bash
ssh darwin.hpc.udel.edu "cat > /lustre/your_workgroup/users/your_username/jupyter-<partition>.qs << 'SLURM_SCRIPT'
#!/bin/bash -l
#SBATCH --job-name=jupyter-<partition>
#SBATCH --partition=<slurm_partition>
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=<time>
#SBATCH --output=slurm-%j.out
<if GPU: #SBATCH --gpus=<gpus>>

vpkg_require anaconda

PORT=<port>
HOST=$(hostname)

echo ================================================
echo Jupyter running on node: $HOST
echo Port: $PORT
echo ================================================

jupyter lab --no-browser --ip=0.0.0.0 --port=$PORT
SLURM_SCRIPT"
```

Step 2 — Submit via workgroup:
```bash
ssh darwin.hpc.udel.edu 'echo "cd /lustre/your_workgroup/users/your_username && sbatch /lustre/your_workgroup/users/your_username/jupyter-<partition>.qs" | workgroup -g your_workgroup'
```

After submission, check node:
```bash
ssh darwin.hpc.udel.edu 'echo "squeue -j <job_id> --format=%.R -h" | workgroup -g your_workgroup'
```

Output format:
```
Jupyter Job Submitted!
======================
Job ID:     <job_id>
Partition:  <partition>
Time:       <time>
Port:       <port>

When job starts on node <node_name>:

1. SSH tunnel (run on LOCAL machine):
   ssh -L <port>:<node_name>:<port> darwin.hpc.udel.edu

2. Browser:
   http://localhost:<port>

3. Get token:
   darwin log <job_id>
```

---

### TUNNEL - Get SSH Tunnel Command

Trigger: user wants the SSH tunnel command (e.g., "get tunnel command for job 12345", "how do I connect to job 12345")

```bash
ssh darwin.hpc.udel.edu 'echo "squeue -j <job_id> --format=\"%R %j %P\" -h" | workgroup -g your_workgroup'
```

Determine port from job name/partition, output tunnel command.

---

### STATUS - View Job Queue

Trigger: user asks about job status (e.g., "check my job status", "what's running", "show my jobs")

```bash
ssh darwin.hpc.udel.edu 'echo "squeue -u your_username --format=\"%.10i %.20j %.12P %.8T %.12M %.4D %.20R\"" | workgroup -g your_workgroup'
```

If empty, show today's history:
```bash
ssh darwin.hpc.udel.edu 'echo "sacct -u your_username --starttime=today --format=JobID,JobName%20,Partition,State,Elapsed,ExitCode -n" | workgroup -g your_workgroup'
```

---

### LOG - View Job Output

Trigger: user wants to see job output/log (e.g., "show log for job 12345", "what's the output of job 12345")

Check both Lustre work directory and home directory (output location depends on submission directory):
```bash
ssh darwin.hpc.udel.edu "tail -100 /lustre/your_workgroup/users/your_username/slurm-<job_id>.out 2>/dev/null || tail -100 /home/your_username/slurm-<job_id>.out 2>/dev/null || echo 'Log not found'"
```

If no job_id, show most recent from either location:
```bash
ssh darwin.hpc.udel.edu "ls -t /lustre/your_workgroup/users/your_username/slurm-*.out /home/your_username/slurm-*.out 2>/dev/null | head -1 | xargs tail -50"
```

---

### CANCEL - Cancel Job

Trigger: user wants to cancel/kill/stop a job (e.g., "cancel job 12345", "stop job 12345")

```bash
ssh darwin.hpc.udel.edu 'echo "scancel <job_id>" | workgroup -g your_workgroup'
```

---

### INFO - Detailed Job Info

Trigger: user wants detailed info about a job (e.g., "show details for job 12345", "job info 12345")

```bash
ssh darwin.hpc.udel.edu 'echo "scontrol show job <job_id>" | workgroup -g your_workgroup'
```

---

### PARTITIONS - Show Resources

Trigger: user asks about available partitions/resources (e.g., "show available partitions", "what resources are available")

```bash
ssh darwin.hpc.udel.edu 'echo "sinfo -o \"%.16P %.6a %.12l %.6D %.6t %.N\"" | workgroup -g your_workgroup'
```

---

### SCRIPTS - List SLURM Scripts

Trigger: user wants to list their SLURM scripts (e.g., "list my slurm scripts", "show my scripts")

```bash
ssh darwin.hpc.udel.edu "ls -la /lustre/your_workgroup/users/your_username/*.{qs,slurm} 2>/dev/null"
```

---

### QUOTA - Check Disk Usage

Trigger: user asks about disk usage/quota (e.g., "check disk usage", "how much storage am I using")

```bash
ssh darwin.hpc.udel.edu "df -h /home/your_username && echo '---' && du -sh /lustre/your_workgroup/users/your_username 2>/dev/null"
```

---

## Notes

- Cluster timezone: US Eastern (ET)
- Output files: `slurm-<jobid>.out` in Lustre work directory
- Template mode generates scripts on-the-fly
- Default runtime is 30 minutes if `--time` is not specified — always specify time explicitly
- The `idle` partition is preemptible: jobs may be killed at any time, use checkpointing
- Max 400 jobs per user per partition (320 for idle)
