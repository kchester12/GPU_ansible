# gpu_harden

Ansible role that hardens Slurm GPU compute nodes on Rocky Linux 9. It covers
the node-side items from the GPU hardening checklist. Fabric (IB P_Keys / Subnet
Manager), firmware and filesystem permissions are out of scope.

## What it does

| Area | Change |
|------|--------|
| Slurm | Sets `ConstrainDevices/Cores/RAMSpace` in `cgroup.conf`, `AutoDetect=nvml` in `gres.conf`. **Checks** (never edits) `slurm.conf` for `proctrack/cgroup`, `task/cgroup`, `PrologFlags=Contain`, `GresTypes=gpu` and the epilog path. |
| SSH | Adds `pam_slurm_adopt` to `/etc/pam.d/sshd`, with a `pam_access` bypass for admin users/groups. Comments out `pam_systemd` in `password-auth`. |
| Driver | `NVreg_RestrictProfilingToAdminUsers=1`, `dnf versionlock` on the installed driver packages, `nvidia-persistenced` enabled, boot-time compute mode unit. |
| Epilog | `/etc/slurm/epilog.d/50-gpu-cleanup.sh`: kills leftover processes on the job's GPUs, drains the node if they won't die, resets clocks and compute mode. |
| Containers | `allow setuid = no` in `apptainer.conf`; optionally updates `nvidia-container-toolkit`. |
| Monitoring | Timer exports `gpu_untracked_processes` (GPU processes outside any Slurm job) to the node_exporter textfile collector. |
| DCGM | systemd drop-in binds `nv-hostengine` to `127.0.0.1`. |

## Before you run it

1. **Admin bypass.** Set `gpu_harden_pam_admin_users` / `gpu_harden_pam_admin_groups`
   in `group_vars`. If your own account isn't covered, you'll be locked out of
   idle nodes (root via console/BMC still works).
2. **slurm.conf (cluster-wide, edit on the controller):**
   ```
   ProctrackType=proctrack/cgroup
   TaskPlugin=task/cgroup,task/affinity
   PrologFlags=Contain
   GresTypes=gpu
   Epilog=/etc/slurm/epilog.d/*
   ```
   `Epilog` globs need Slurm 21.08+. If you already have an Epilog script, call
   `50-gpu-cleanup.sh` from it instead.
3. **Configless Slurm?** Set `gpu_harden_manage_cgroup_conf: false` and
   `gpu_harden_manage_gres_conf: false` and put those settings on the controller.
4. **node_exporter** must run with
   `--collector.textfile.directory=/var/lib/node_exporter/textfile_collector`.
5. **Compute mode.** `EXCLUSIVE_PROCESS` breaks codes that put several processes
   on one GPU (multiple MPI ranks per GPU, MPS). Set `gpu_harden_compute_mode: ""`
   to skip it.

## Running

```bash
# Dry run on one node
ansible-playbook -i inventory gpu_harden.yml --check --diff -l gpunode01

# Apply to one node, then the rest (25% at a time by default)
ansible-playbook -i inventory gpu_harden.yml -l gpunode01
ansible-playbook -i inventory gpu_harden.yml

# Only some parts
ansible-playbook -i inventory gpu_harden.yml --tags slurm,epilog
```

Tags: `slurm`, `pam`, `driver`, `epilog`, `containers`, `monitoring`, `dcgm`.

Every run ends with a per-node summary: missing `slurm.conf` settings, warnings,
and whether a reboot is needed (the profiling restriction only takes effect
when the nvidia module reloads).

## Verify after applying

```bash
# Device isolation: should list exactly one GPU
srun --gres=gpu:1 nvidia-smi -L

# SSH as a normal user to a node where you have no job: should be refused
ssh gpunode02

# Profiling restriction (after reboot): should print 1
grep RmProfilingAdminOnly /proc/driver/nvidia/params

# Epilog output
journalctl -t gpu-epilog --since today

# Stray-process metric
cat /var/lib/node_exporter/textfile_collector/gpu_stray.prom
```

## Prometheus alert example

```yaml
- alert: GPUProcessOutsideSlurmJob
  expr: gpu_untracked_processes > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "GPU process outside a Slurm job on {{ $labels.instance }}"
    description: "Check: journalctl -t gpu-stray on the node."
```

## Known limits

- The `pam_systemd` change edits the authselect-generated `password-auth`.
  `authselect apply-changes` will undo it; make a custom authselect profile to
  keep it.
- With MIG enabled, the epilog's minor-number mapping covers the parent GPU,
  not individual MIG instances. Test it on a MIG node before relying on it.
- The DCGM drop-in replaces `ExecStart`. Check that
  `gpu_harden_dcgm_hostengine_args` matches `systemctl cat nvidia-dcgm` for your
  DCGM version.
