# Linux Disk Cleanup and Audit Cheatsheet

A practical Bash command collection for auditing disk usage, locating large files, and safely deciding what can be cleaned on shared Linux/TPU servers.

This guide is based on a real TPU VM cleanup workflow involving:

- root disk usage checks
- `/home`, `/tmp`, `/var`, and user directory audits
- model/checkpoint directory inspection
- fixed filename search
- file tree depth control with `du`
- safe deletion strategies
- cleanup manifest generation

> **Rule of thumb:** audit first, delete later. On shared servers, never delete another user's model, checkpoint, virtual environment, or cache before confirming ownership and purpose.

---

## 0. Quick Start

Run these first when the server disk is almost full.

```bash
# Show total, used, and available space on root disk
df -h /

# Show whether /home, /tmp, and /var share the same root disk
df -h / /home /tmp /var

# Show inode usage
df -ih /

# Show top-level directory usage under root
sudo du -xhd1 / 2>/dev/null | sort -h
```

Example interpretation:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        97G   90G  7.0G  93% /
```

Meaning:

```text
Size   = total disk size
Used   = used disk space
Avail  = available disk space
Use%   = usage percentage
Mounted on = mount point
```

---

## 1. Root-Level Disk Audit

Use this to understand where root disk space is going.

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Typical output:

```text
2.1G    /usr
5.0G    /tmp
12G     /var
48G     /home
67G     /
```

Important flags:

```text
-x  = stay on the same filesystem
-h  = human-readable sizes
-d1 = depth 1 only
```

If sorting makes the command look stuck, run without `sort` first:

```bash
sudo du -xhd1 / 2>/dev/null
```

---

## 2. Audit `/home`

Check which user owns the most data.

```bash
sudo du -xhd1 /home 2>/dev/null | sort -h
```

Check a specific user directory:

```bash
sudo du -xhd1 /home/<user_name> 2>/dev/null | sort -h
```

Example:

```bash
sudo du -xhd1 /home/jason 2>/dev/null | sort -h
```

Typical output:

```text
6.0G    /home/jason/.vscode-server
7.7G    /home/jason/.cache
8.6G    /home/jason/.venvs
19G     /home/jason/algorithms
42G     /home/jason
```

---

## 3. File Tree Depth Control

Depth control is useful when a directory is huge.

```bash
# Depth 1
sudo du -xhd1 /path/to/dir 2>/dev/null | sort -h

# Depth 2
sudo du -xhd2 /path/to/dir 2>/dev/null | sort -h

# Depth 3
sudo du -xhd3 /path/to/dir 2>/dev/null | sort -h

# Show only largest 30 entries
sudo du -xhd2 /path/to/dir 2>/dev/null | sort -h | tail -30
```

---

## 4. Audit Experiment Runs

For model training output folders such as `runs/`, first inspect directory sizes.

```bash
sudo du -xhd1 /home/jason/algorithms/runs 2>/dev/null | sort -h
```

Show only the largest run directories:

```bash
sudo du -xhd1 /home/jason/algorithms/runs 2>/dev/null | sort -h | tail -30
```

Inspect one run in detail:

```bash
sudo du -xhd2 /home/jason/algorithms/runs/<run_dir_name> 2>/dev/null | sort -h
```

Example:

```bash
sudo du -xhd2 /home/jason/algorithms/runs/training_seed1 2>/dev/null | sort -h
```

Typical structure:

```text
2.6M    tensorboard
102M    checkpoints
2.9G    exported_model
3.0G    run_dir
```

In this case, `exported_model/` is the main space consumer.

---

## 5. Search for Fixed Files

### Find all `model.safetensors`

```bash
sudo find /home/jason/algorithms/runs -type f -name "model.safetensors" \
  -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
  | sort -k3 -n \
  | numfmt --field=3 --to=iec
```

### Find all exported models

```bash
cd /home/jason/algorithms/runs

sudo find . -path "*/exported_model" -type d \
  -exec du -sh {} \; 2>/dev/null | sort -h
```

### Find files larger than 1GB

```bash
sudo find /home/jason -xdev -type f -size +1G \
  -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
  | sort -k3 -n \
  | numfmt --field=3 --to=iec
```

### Find files larger than 500MB

```bash
sudo find /home/jason -xdev -type f -size +500M \
  -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
  | sort -k3 -n \
  | numfmt --field=3 --to=iec
```

---

## 6. Classify Experiment Outputs Before Deletion

Before deleting training outputs, classify them by importance.

### Usually keep

```text
SFT base model
results/
logs/
current experiment checkpoints
current active run
config files
metric JSON/CSV files
```

Example:

```text
training_seed1_2026
results/
logs/
```

### Usually safe to consider deleting

```text
smoke runs
old duplicate runs
old flip/noise runs no longer used
exported full models that can be regenerated from base model + LoRA checkpoint
```

### Be careful with

```text
checkpoints/
.cache/huggingface
.venvs/
.vscode-server/
```

---

## 7. Delete Only `exported_model/`, Not the Whole Run

If a DPO run contains:

```text
2.9G exported_model
102M checkpoints
2.6M tensorboard
```

a safer cleanup is to delete only:

```text
exported_model/
```

and keep:

```text
checkpoints/
tensorboard/
logs/
results/
```

### Dry-run: print candidate directories

```bash
cd /home/jason/algorithms/runs

find . -path "./training_seed1*_20260417_013847/exported_model" \
  -type d \
  -print
```

### Safety check: make sure SFT is not included

```bash
find . -path "./training_seed1_*_20260417_013847/exported_model" \
  -type d \
  -print | grep "Qwen" && echo "WARNING: SFT included!" || echo "OK: SFT not included"
```

### Delete candidate exported models

```bash
cd /home/jason/algorithms/runs

for d in training_seed1_*_20260417_013847; do
  if [ -d "$d/exported_model" ]; then
    echo "Deleting $d/exported_model"
    sudo rm -rf "$d/exported_model"
  fi
done

df -h /
```

---

## 8. Clean Smoke Runs

Smoke runs are usually pipeline tests and often safe to delete after confirmation.

### Check size

```bash
cd /home/jason/algorithms/runs

sudo du -sch *smoke* 2>/dev/null
```

### Delete

```bash
sudo rm -rf *smoke*
df -h /
```

---

## 9. Clean Old Flip or Noise Runs

Noise experiments may include names such as:

```text
global_flip20
global_flip40
tail50_flip20
tail50_flip40
no_pref
mismatch
```

### Dry-run

```bash
cd /home/jason/algorithms/runs

sudo du -sh *global_flip* *tail50_flip* 2>/dev/null | sort -h
```

### Save deletion list

```bash
cd /home/jason/algorithms/runs

DELETE_LIST=~/delete_flip_runs_$(date +%Y%m%d_%H%M%S).txt

{
  echo "=== flip/tail flip runs to delete ==="
  sudo du -sh *global_flip* *tail50_flip* 2>/dev/null | sort -h
} | tee "$DELETE_LIST"

echo "Saved to: $DELETE_LIST"
```

### Delete after confirmation

```bash
sudo rm -rf *global_flip* *tail50_flip*
df -h /
```

---

## 10. Clean Old Duplicate Seed Runs

If the same setting was run twice, for example:

```text
seed1_20260422_220956
seed1_20260424_163542
```

keep the final version and delete the old duplicate only after confirmation.

### Check old duplicates

```bash
cd /home/jason/algorithms/runs

sudo du -sch *20260422* 2>/dev/null
sudo du -sh *20260422* 2>/dev/null | sort -h
```

### Delete old duplicates

```bash
sudo rm -rf *20260422*
df -h /
```

---

## 11. Inspect and Clean Cache

### Check cache size

```bash
sudo du -xhd1 /home/jason/.cache 2>/dev/null | sort -h
sudo du -xhd2 /home/jason/.cache 2>/dev/null | sort -h | tail -50
```

### Usually safe cache cleanup

```bash
sudo rm -rf /home/jason/.cache/pip
sudo rm -rf /home/jason/.cache/matplotlib
sudo rm -rf /home/jason/.cache/jax
sudo rm -rf /home/jason/.cache/torch
```

### Hugging Face cache

Inspect first:

```bash
sudo du -xhd1 /home/jason/.cache/huggingface 2>/dev/null | sort -h
```

Be careful:

```text
Deleting Hugging Face cache may force model or dataset re-downloads.
```

---

## 12. Inspect Virtual Environments

Never delete the currently active environment.

### Check active environment

```bash
echo $VIRTUAL_ENV
which python
which pip
```

### Check virtual environment sizes

```bash
sudo du -xhd1 /home/jason/.venvs 2>/dev/null | sort -h
ls -lah /home/jason/.venvs
```

Example:

```text
804M    .venvs/env
2.3G    .venvs/env1
5.6G    .venvs/env2
```

If current prompt shows:

```text
(DPO)
```

do not delete `.venvs/env`.

Delete only after confirmation:

```bash
sudo rm -rf /home/jason/.venvs/DPO-EVAL-OFFLINE
```

---

## 13. Inspect VS Code Server

VS Code Remote installs server files under `.vscode-server`.

### Check size

```bash
sudo du -xhd1 /home/jason/.vscode-server 2>/dev/null | sort -h
sudo du -xhd2 /home/jason/.vscode-server 2>/dev/null | sort -h | tail -50
```

### Be careful

Do not delete `.vscode-server` if you are currently using VS Code Remote with that Linux account.

If deleted, VS Code usually reinstalls it next time, but the current session may be interrupted.

---

## 14. Inspect `/tmp`

```bash
sudo du -xhd1 /tmp 2>/dev/null | sort -h
```

Important candidates:

```text
/tmp/models
/tmp/tpu_logs
```

### Inspect `/tmp/models`

```bash
sudo du -xhd2 /tmp/models 2>/dev/null | sort -h
```

Be careful if it contains a base model:

```text
/tmp/models/training_seed1
```

### Inspect `/tmp/tpu_logs`

```bash
sudo du -xhd1 /tmp/tpu_logs 2>/dev/null | sort -h
sudo ls -lahtr /tmp/tpu_logs | tail -50
```

### Clean old TPU logs

Dry-run:

```bash
sudo find /tmp/tpu_logs -type f -mtime +2 -print
```

Delete old files:

```bash
sudo find /tmp/tpu_logs -type f -mtime +2 -delete
df -h /
```

Keep only the latest 50 TPU driver logs:

```bash
cd /tmp/tpu_logs

sudo ls -1tr tpu_driver.*.log.* 2>/dev/null | head -n -50 | sudo xargs -r rm -f

df -h /
sudo du -sh /tmp/tpu_logs
```

---

## 15. Inspect `/var`

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Common large subdirectories:

```text
/var/log
/var/lib
/var/cache
```

### Inspect `/var/log`

```bash
sudo du -xhd1 /var/log 2>/dev/null | sort -h

sudo find /var/log -type f -size +100M \
  -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
  | sort -k3 -n \
  | numfmt --field=3 --to=iec
```

### Clean systemd journal safely

```bash
sudo journalctl --disk-usage
sudo journalctl --vacuum-size=500M
df -h /
```

### Remove old rotated logs

First check whether they are open:

```bash
sudo lsof /var/log/syslog.1 /var/log/kern.log.1 2>/dev/null
```

If no output, they are not currently open:

```bash
sudo rm -f /var/log/syslog.1 /var/log/kern.log.1
df -h /
```

Do not worry if `/var/log/lastlog` appears huge in `ls` or `find` output. It is often a sparse file; `du` shows the real disk usage.

---

## 16. Inspect Docker Without Breaking TPU Runtime

On TPU VMs, Docker containers may be system services.

```bash
sudo docker ps
sudo docker ps -a
sudo docker images
sudo docker system df
```

If you see active TPU service containers such as:

```text
healthagent
tpu-runtime
instance_agent
google-runtime-monitor
vbarcontrolagent
google-collectd
monitoringagent
```

do not stop, remove, or prune them aggressively.

### Check Docker directory size

```bash
sudo du -xhd1 /var/lib/docker 2>/dev/null | sort -h
```

### Check large Docker files

```bash
sudo find /var/lib/docker -type f -size +100M \
  -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
  | sort -k3 -n \
  | numfmt --field=3 --to=iec
```

### Do not run these unless you fully understand the impact

```bash
sudo docker system prune -a
sudo docker rm ...
sudo docker rmi ...
sudo rm -rf /var/lib/docker
```

If `docker system df` shows:

```text
RECLAIMABLE 0B
```

then Docker cleanup will not help.

---

## 17. Find Deleted but Still Open Files

Sometimes `df` shows high usage but `du` does not. This can happen when deleted files are still held open by running processes.

```bash
sudo lsof +L1 2>/dev/null | sort -k7 -n | tail -30
```

If you see a very large deleted file, restart the process that holds it, or reboot the machine if appropriate.

---

## 18. Full Disk Audit Script

Save a complete audit report:

```bash
AUDIT=~/disk_audit_$(date +%Y%m%d_%H%M%S).txt

{
  echo "=== df -h / ==="
  df -h /

  echo
  echo "=== df -h / /home /tmp /var ==="
  df -h / /home /tmp /var

  echo
  echo "=== inode usage ==="
  df -ih /

  echo
  echo "=== root level ==="
  sudo du -xhd1 / 2>/dev/null | sort -h

  echo
  echo "=== /home level ==="
  sudo du -xhd1 /home 2>/dev/null | sort -h

  echo
  echo "=== professor home level ==="
  sudo du -xhd1 /home/jason 2>/dev/null | sort -h

  echo
  echo "=== algorithms level ==="
  sudo du -xhd1 /home/jason/algorithms 2>/dev/null | sort -h

  echo
  echo "=== algorithms/runs level ==="
  sudo du -xhd1 /home/jason/algorithms/runs 2>/dev/null | sort -h

  echo
  echo "=== cache level ==="
  sudo du -xhd1 /home/jason/.cache 2>/dev/null | sort -h

  echo
  echo "=== virtual environments ==="
  sudo du -xhd1 /home/jason/.venvs 2>/dev/null | sort -h

  echo
  echo "=== /tmp level ==="
  sudo du -xhd1 /tmp 2>/dev/null | sort -h

  echo
  echo "=== /var level ==="
  sudo du -xhd1 /var 2>/dev/null | sort -h

  echo
  echo "=== /var/log level ==="
  sudo du -xhd1 /var/log 2>/dev/null | sort -h

  echo
  echo "=== /var/lib level ==="
  sudo du -xhd1 /var/lib 2>/dev/null | sort -h

  echo
  echo "=== large files >1G under professor home ==="
  sudo find /home/jason -xdev -type f -size +1G \
    -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
    | sort -k3 -n \
    | numfmt --field=3 --to=iec

} | tee "$AUDIT"

echo "Saved to: $AUDIT"
```

---

## 19. Cleanup Review Script for `runs/`

This script summarizes candidate categories before deletion.

```bash
cd /home/jason/algorithms/runs

REVIEW=~/runs_cleanup_review_$(date +%Y%m%d_%H%M%S).txt

{
  echo "=== current df ==="
  df -h /

  echo
  echo "=== keep: sft/results/logs ==="
  sudo du -sh training_seed1 results logs 2>/dev/null

  echo
  echo "=== smoke runs ==="
  sudo du -sch *smoke* 2>/dev/null

  echo
  echo "=== flip/tail flip runs ==="
  sudo du -sch *global_flip* *tail50_flip* 2>/dev/null

  echo
  echo "=== old 20260414 runs ==="
  sudo du -sch *20260414* 2>/dev/null

  echo
  echo "=== old duplicate 20260422 runs ==="
  sudo du -sch *20260422* 2>/dev/null

  echo
  echo "=== exported models ==="
  sudo find . -path "*/exported_model" -type d \
    -exec du -sh {} \; 2>/dev/null | sort -h

  echo
  echo "=== model.safetensors files ==="
  sudo find . -type f -name "model.safetensors" \
    -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
    | sort -k3 -n \
    | numfmt --field=3 --to=iec

} | tee "$REVIEW"

echo "Saved to: $REVIEW"
```

---

## 20. Recommended Cleanup Order

Recommended low-risk order:

```text
1. Delete smoke runs.
2. Delete old exported_model/ directories if checkpoints and results are kept.
3. Delete old flip/tail flip runs if they are no longer used.
4. Delete duplicate old seed runs.
5. Clean pip/jax/torch/matplotlib cache.
6. Vacuum old journal logs.
7. Clean old /tmp/tpu_logs.
```

Medium-risk items:

```text
.cache/huggingface
.venvs/
.vscode-server/
```

High-risk items:

```text
SFT base model
current run checkpoints
results/
logs/
Docker system containers
/tmp/models if used as base model
```

---

## 21. Confirmation Checklist Before Deletion

Before deleting any model or run directory, confirm:

```text
[ ] It is not the SFT base model.
[ ] It is not the current active experiment.
[ ] It is not needed as input to another script.
[ ] Metrics/results are already saved.
[ ] Logs/configs are preserved.
[ ] It can be regenerated from code + base model + checkpoint if needed.
[ ] The owner or teammate confirmed it can be removed.
```

---

## 22. Useful One-Liners

### Check total disk

```bash
df -h /
```

### Check root usage

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

### Check professor home

```bash
sudo du -xhd1 /home/jason 2>/dev/null | sort -h
```

### Check runs

```bash
sudo du -xhd1 /home/jason/algorithms/runs 2>/dev/null | sort -h
```

### Find large model files

```bash
sudo find /home/jason -xdev -type f -size +1G \
  -printf '%TY-%Tm-%Td %TH:%TM  %s  %p\n' 2>/dev/null \
  | sort -k3 -n \
  | numfmt --field=3 --to=iec
```

### Clean journal logs

```bash
sudo journalctl --vacuum-size=500M
```

### Check deleted-but-open files

```bash
sudo lsof +L1 2>/dev/null | sort -k7 -n | tail -30
```
