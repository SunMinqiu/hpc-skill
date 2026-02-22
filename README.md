# DARWIN HPC Cluster AI Agent Skill

An AI agent skill that lets you manage jobs on the [University of Delaware DARWIN HPC cluster](https://docs.hpc.udel.edu/abstract/darwin/darwin) directly from your terminal using natural language.

## What It Does

This skill turns your AI agent into a remote HPC job manager. Instead of manually SSH-ing into the cluster, writing SLURM scripts, and running `sbatch`, you can just tell your agent what you want in natural language:

```
Run python train.py on gpu-v100 for 4 hours with 2 GPUs on DARWIN
Check my job status on DARWIN
On DARWIN cluster, Show log for job 12345
Cancel job 12345 on DARWIN
```

Under the hood, the skill constructs and executes the appropriate SSH + SLURM commands on your behalf.

## Prerequisites

1. **SSH access to DARWIN** - You must have an account on `darwin.hpc.udel.edu` and be able to SSH in without a password (key-based authentication). Test with:
   ```bash
   ssh darwin.hpc.udel.edu "echo ok"
   ```
   If this doesn't print `ok`, set up your SSH key first.

2. **An AI agent that supports skills** - e.g., [Claude Code](https://claude.com/claude-code), or any compatible AI coding agent.

3. **A DARWIN allocation** - You need an active workgroup/allocation (e.g., `it_css`) to submit jobs.

## Installation

1. Copy the `darwin/SKILL.md` file to your AI agent's skills directory (e.g. for Claude Code: `~/.claude/skills/darwin/`):
   ```bash
   mkdir -p ~/.claude/skills/darwin
   cp darwin/SKILL.md ~/.claude/skills/darwin/
   ```

2. **Edit the connection info** in `SKILL.md` to match your account. Update these fields at the top of the file:
   ```
   - **Host**: `darwin.hpc.udel.edu`
   - **User**: `your_username`
   - **Workgroup**: `your_workgroup`
   - **Home**: `/home/your_username`
   - **Lustre Work**: `/lustre/your_workgroup/users/your_username`
   ```

3. Restart your AI agent. The `darwin` skill should now be available.

## Usage

Just tell the skill what you want in natural language. Traditional SLURM-style commands (e.g., `sbatch`, `squeue`, `scancel`) also work — the agent understands both. Here are some examples by category:

### Job Submission

| What you say | What it does |
|-------------|-------------|
| `run python train.py on gpu-v100 for 4 hours with 2 GPUs` | Submit a GPU job with auto-generated SLURM script |
| `run python preprocess.py on standard for 2 hours with 64G memory` | Submit a CPU job with custom memory |
| `submit my_experiment.qs` | Submit an existing SLURM script file |

### Jupyter Notebook

| What you say | What it does |
|-------------|-------------|
| `launch jupyter on gpu-v100 for 8 hours` | Start Jupyter Lab on a V100 node |
| `start jupyter on gpu-t4 for 4 hours` | Start Jupyter Lab on a T4 node |
| `get the tunnel command for job 12345` | Get the SSH tunnel command for a running Jupyter job |

### Job Management

| What you say | What it does |
|-------------|-------------|
| `check my job status` | Show your current job queue |
| `show log for job 12345` | View job output (last 100 lines) |
| `show details for job 12345` | Show detailed SLURM job info |
| `cancel job 12345` | Cancel a running or pending job |

### Cluster Resources

| What you say | What it does |
|-------------|-------------|
| `show available partitions` | Show partitions and node status |
| `list my slurm scripts` | List your `.qs` / `.slurm` script files |
| `check disk usage` | Check disk quota |

## Available Partitions

> Partition details sourced from the [DARWIN cluster documentation](https://docs.hpc.udel.edu/abstract/darwin/runjobs/queues).

### CPU Partitions

| Partition | Cores/Node | Memory/Node | Max Time |
|-----------|-----------|-------------|----------|
| `standard` | 64 | 512 GiB | 7 days |
| `large-mem` | 64 | 1024 GiB | 7 days |
| `xlarge-mem` | 64 | 2048 GiB | 7 days |
| `extended-mem` | 64 | 1024 GiB + 2.73 TiB NVMe | 7 days |

### GPU Partitions

| Partition | Cores/Node | GPUs/Node | GPU Memory | Max Time |
|-----------|-----------|-----------|------------|----------|
| `gpu-t4` | 64 | 1x NVIDIA T4 | 16 GB | 7 days |
| `gpu-v100` | 48 | 4x NVIDIA V100 | 32 GB each | 7 days |
| `gpu-mi50` | 64 | 1x AMD MI50 | - | 7 days |
| `gpu-mi100` | 64 | 1x AMD MI100 | - | 7 days |

### Special

| Partition | Description |
|-----------|-------------|
| `idle` | All node types, preemptible (jobs may be killed), no SU charge |

## Examples

**Run a Python training script on V100 GPUs:**
```
On DARWIN, run python train.py training 100 epochs on gpu-v100 with 2 GPUs for 12 hours 
```

**Run a CPU-only data processing job:**
```
On DARWIN, run python preprocess.py on standard for 2 hours with 64G memory
```

**Launch Jupyter on a T4 GPU node:**
```
Launch jupyter on gpu-t4 for 4 hours on DARWIN
```

**Check what's running:**
```
Check my job status now
```

**Submit an existing script:**
```
submit my_experiment.qs on ...
```

## Storage Layout

| Path | Purpose | Quota |
|------|---------|-------|
| `/home/<user>` | Config files, small scripts | 20 GiB |
| `/lustre/<workgroup>/users/<user>` | Job data, large files, output | Workgroup quota |
| `$TMPDIR` (on compute node) | Fast node-local scratch | ~1.75 TiB SSD |

Jobs should read/write data from Lustre (`/lustre/...`), not from the home directory.

## Troubleshooting

**"Pre-flight check failed"** - The skill checks SSH connectivity before every operation. If it fails:
- Verify you can SSH manually: `ssh darwin.hpc.udel.edu "echo ok"`
- Check your SSH key: `ls ~/.ssh/id_*`
- Check your SSH config: `cat ~/.ssh/config`
- Make sure you're on the right network / VPN

**"All jobs must explicitly request a partition"** - DARWIN requires a `--partition` flag on every job. Use `-p` to specify one.

**Job terminates after ~1 minute** - Likely a time format error. Use `HH:MM:SS` (e.g., `1:00:00` for 1 hour) or `D-HH:MM:SS` (e.g., `1-00:00:00` for 1 day). A bare `60` means 60 *minutes*, not 60 hours.

**Permission denied on job submission** - Make sure your workgroup is correctly set in the skill's Connection Info section.

## Customization

Edit `~/.claude/skills/darwin/SKILL.md` to:
- Change your username, workgroup, or storage paths
- Add new partition configurations
- Modify default resource values (time, memory, GPUs)
- Add custom module loading or environment setup to job templates

## Contact

For questions or issues, reach out to **mqsun@udel.edu**.

## Reference

- [DARWIN Documentation](https://docs.hpc.udel.edu/abstract/darwin/darwin)
- [DARWIN Partitions](https://docs.hpc.udel.edu/abstract/darwin/runjobs/queues)
- [DARWIN Job Scheduling](https://docs.hpc.udel.edu/abstract/darwin/runjobs/schedule_jobs)
- [Slurm Documentation](https://slurm.schedmd.com/documentation.html)
