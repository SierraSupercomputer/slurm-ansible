# SLURM Compute Node Bootstrap (Ansible)

Turns a fresh Ubuntu host into a working SLURM compute node by copying the
`munge.key` and SLURM config files straight from your existing controller,
rather than hand-authoring them.

## Layout

```
slurm-compute-node/
├── ansible.cfg
├── inventory.ini        # edit this: your controller + new node(s)
├── group_vars/all.yml   # tunables: paths, uids/gids, packages, ports
├── site.yml              # the playbook
└── README.md
```

## What it does

1. **Play 1 (runs against `slurm_controller`)** — fetches `munge.key`,
   `slurm.conf`, and any of `cgroup.conf` / `gres.conf` / `topology.conf` /
   etc. that exist, staging them locally under `/tmp/slurm_cluster_files`.
2. **Play 2 (runs against `compute_new`)** — installs `munge` + `slurmd`,
   creates the `munge` and `slurm` system users/groups with fixed UID/GID
   (critical — these must match across every node or munge auth breaks),
   deploys the staged configs, sets up spool/log dirs, opens the firewall
   for the slurmd port, and starts services.
3. **Play 3 (tag `reconfigure`, runs against `slurm_controller`)** — runs
   `scontrol reconfigure` and prints `sinfo -N -l` so you can confirm the
   new node shows up.

## Prerequisites

- SSH access from wherever you run Ansible to both the controller and the
  new node(s), with sudo.
- **The new node must already be listed as a `NodeName` line in the
  controller's `slurm.conf`** before you run this — the playbook copies
  `slurm.conf` verbatim, it does not add nodes to it. Add the node there
  first (and to any `PartitionName` line you want it in), then run this
  playbook, then let Play 3 reconfigure `slurmctld`.
- The `slurm_uid`/`slurm_gid`/`munge_uid`/`munge_gid` in `group_vars/all.yml`
  should match what's already in use on your controller/other compute
  nodes. Check with `id munge` / `id slurm` on the controller and update
  the vars to match if your cluster uses different values than the
  defaults (981/982).

## Usage

```bash
cd slurm-compute-node
# edit inventory.ini with real hostnames/IPs
ansible-playbook site.yml

# just one node:
ansible-playbook site.yml --limit node01

# skip the controller reconfigure step:
ansible-playbook site.yml --skip-tags reconfigure
```

## Notes / things you may want to adjust

- **Package source**: Ubuntu's default repos carry reasonably recent SLURM
  packages, but if your cluster relies on a specific SLURM version (e.g.
  to match a shared NFS-mounted `/opt/slurm` build), you may need to add
  the SchedMD PPA or build from source instead of `apt install slurmd`
  before this playbook's package task will pull the right version.
- **NFS / shared storage**: if your cluster mounts shared `/home` or
  scratch storage via NFS, that's not handled here — add an NFS client
  role/tasks as needed.
- **cgroups**: `cgroup.conf` is copied if present, but this playbook
  doesn't install `cgroup-tools` or configure cgroup v1/v2 mode — add that
  if your `cgroup.conf` requires it.
- **Firewall**: the `ufw` task is best-effort (`failed_when: false`) so it
  won't break the run on hosts where ufw isn't installed/enabled. Set
  `open_firewall: false` in `group_vars/all.yml` to skip it entirely, or
  swap in `firewalld` tasks if that's what your distro image uses.
