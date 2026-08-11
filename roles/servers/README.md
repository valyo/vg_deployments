# servers

Ansible role for bootstrapping and configuring home servers. Supports three server types via `server_role`: **zfs**, **infra**, and **backup**.

## Server types

### zfs_server (primary)

Primary storage server with ZFS pools. Runs Docker (with ZFS storage driver), CasaOS, and hosts the main data in `/burkpool`. Pools are auto-discovered and imported on fresh installs or recovery — no need to hardcode pool names. A ZFS dataset (`<pool>/docker`) is automatically created for Docker storage. The playbook will fail explicitly if `docker_zfs_storage` is enabled but no ZFS pool can be found.

Also provides:
- Passwordless `sudo rsync` so the backup server can pull data remotely
- `/burkpool/backups/infra` directory for receiving infra server backups
- `/storage` ownership set to `local_user`
- `radeontop` for GPU monitoring

### infra_server (laptop)

Infrastructure server running Docker containers on a laptop. Lid close events are ignored so the server stays running when closed. Pushes local backup data to the ZFS server on a cron schedule.

### backup_server

Offsite/secondary backup server. Pulls `/burkpool` from the ZFS server via rsync. Also manages an external exFAT drive for cold storage archival. Does not install Docker directly — CasaOS brings in its own Docker dependency.

## What the role installs (all servers)

- Common packages: htop, vim, lm-sensors, rsync, tmux, tcpdump, dnsutils, git-core, nmap, smartmontools, etc.
- Passwordless `sudo smartctl` for disk monitoring
- Locale (en_US.UTF-8), timezone, NTP
- SSH keys, `~/.ssh/config` (auto-generated entries for all other servers), bashrc (with `ssh-agent` auto-start), tmux config and plugins
- Avahi/Samba service discovery
- CasaOS
- Snapd removal
- History backup cron (daily at 03:00, saves `.bash_history` to `/burkpool/history_files/<hostname>` on the ZFS server)

## Conditional features

| Feature | Variable | Default | Servers |
|---|---|---|---|
| Docker | `install_docker` | `true` | zfs, infra |
| Docker ZFS storage | `docker_zfs_storage` | `false` | zfs |
| CasaOS | `install_casaos` | `true` | all |
| ZFS | `install_zfs` | `false` | zfs |
| HFS+ support | `install_hfs` | `false` | zfs, backup |
| Extra user | `add_extra_user` | `false` | zfs |
| radeontop | `server_role` | — | zfs |
| Disk status script | `server_role` | — | zfs, backup |
| Lid close ignore | `server_role` | — | infra |

## Variables

### Common defaults (`inventory/group_vars/all.yml`)

| Variable | Default | Description |
|---|---|---|
| `install_docker` | `true` | Install Docker CE |
| `install_casaos` | `true` | Install CasaOS |
| `install_zfs` | `false` | Install ZFS utilities |
| `add_extra_user` | `false` | Create an additional user |
| `install_hfs` | `false` | Install HFS+ filesystem support |
| `server_timezone` | `Europe/Stockholm` | System timezone |
| `ntp_server` | `0.se.pool.ntp.org` | NTP server |

### ZFS server specific (`inventory/group_vars/zfs_server.yml`)

| Variable | Description |
|---|---|
| `zpools` | Explicit pool names to import (empty = auto-discover) |
| `docker_zfs_storage` | Use ZFS as Docker storage driver |

### Sensitive variables (vault-encrypted)

Stored in `inventory/host_vars/<hostname>/vault.yml`:

- `ansible_host` — server IP address
- `ansible_user` — SSH user
- `backup_source_host` — ZFS server IP (backup_server only)
- `backup_target_host` — ZFS server IP (infra_server only)
- `backup_target_dir` — remote backup destination path (infra_server only)
- `backup_paths` — local paths to back up (infra_server only)
- `external_drive_uuid` — UUID of external backup drive (backup_server only)

Stored in `roles/servers/vars/main.yml` (vault-encrypted):

- `new_user` — username for the extra user (used when `add_extra_user: true`)

SSH keys in `roles/servers/files/` are also vault-encrypted.

### Backup server specific (`inventory/group_vars/backup_server.yml`)

| Variable | Description |
|---|---|
| `disk_mounts` | List of internal disks to mount (device, path, fstype) |
| `backup_local_dir` | Local path for ZFS backup data |
| `backup_source_dir` | Remote path to pull from ZFS server |
| `external_drive_fstype` | Filesystem type of external drive |
| `external_drive_mount` | Mount point for external drive |
| `backup_rsync_opts` | Rsync flags for backup pull |

### Infra server specific (`inventory/group_vars/infra_server.yml`)

| Variable | Description |
|---|---|
| `backup_target_dir` | Remote path on ZFS server to push backups to |
| `backup_paths` | List of local paths to back up |
| `backup_cron_minute` | Cron schedule minute |
| `backup_cron_hour` | Cron schedule hour |
| `enable_backup_cron` | Enable/disable the backup cron job |
| `backup_rsync_opts` | Rsync flags for backup push |

## Usage

### Running the playbook

All server types use the same playbook:

```bash
ansible-playbook -i inventory/hosts setup_servers.yml -e host=zfs_server --ask-vault-pass
ansible-playbook -i inventory/hosts setup_servers.yml -e host=infra_server --ask-vault-pass
ansible-playbook -i inventory/hosts setup_servers.yml -e host=backup_server --ask-vault-pass
```

The inventory can also be used with other roles in this repo:

```bash
ansible-playbook -i inventory/hosts deploy_samba.yml -e host=burkeng --ask-vault-pass
```

### Using tags

Run specific sections only:

```bash
# Common setup only
ansible-playbook -i inventory/hosts setup_servers.yml -e host=backup_server --ask-vault-pass --tags common

# ZFS tasks only
ansible-playbook -i inventory/hosts setup_servers.yml -e host=zfs_server --ask-vault-pass --tags zfs

# Backup tasks only
ansible-playbook -i inventory/hosts setup_servers.yml -e host=backup_server --ask-vault-pass --tags backup

# Disk mounts only
ansible-playbook -i inventory/hosts setup_servers.yml -e host=backup_server --ask-vault-pass --tags mounts

# External drive setup only
ansible-playbook -i inventory/hosts setup_servers.yml -e host=backup_server --ask-vault-pass --tags external

# Infra laptop config only
ansible-playbook -i inventory/hosts setup_servers.yml -e host=infra_server --ask-vault-pass --tags infra
```

Available tags: `common`, `bootstrap`, `apt`, `ssh-keys`, `zfs`, `backup`, `mounts`, `external`, `snap`, `users`, `casaos`, `docker`, `infra`, `laptop`.

## Scripts deployed on servers

### ZFS and backup servers

**`/usr/local/bin/disk_status.sh`** — Shows model, capacity, and temperature for all drives.

```bash
disk_status.sh
```

### All servers

**`/usr/local/bin/save_history.sh`** — Saves `.bash_history` to `/burkpool/history_files/<hostname>` on the ZFS server. Runs daily at 03:00 via cron. On the ZFS server it copies locally; on other servers it uses `scp`.

### Backup server

**`/usr/local/bin/backup_from_primary.sh`** — Pulls the entire `/burkpool` from the ZFS server into `/backup_burkpool` using rsync over SSH. Runs as root (uses `sudo rsync` on the remote ZFS server to read all files with preserved ownership).

```bash
sudo /usr/local/bin/backup_from_primary.sh           # full sync
sudo /usr/local/bin/backup_from_primary.sh --dry-run  # preview only
```

**`/usr/local/bin/sync_from_external.sh`** — Mounts the external exFAT drive, syncs `archive_primary_copy/` into `/cold_storage/archive_backup/`, then unmounts. Runs as your regular user (uses sudo only for mount/umount). Uses `--modify-window=2` to handle exFAT timestamp precision.

```bash
/usr/local/bin/sync_from_external.sh           # full sync
/usr/local/bin/sync_from_external.sh --dry-run  # preview only
```

### Infra server

**`/usr/local/bin/backup_to_burkeng.sh`** — Pushes local paths (defined in `backup_paths`) to the ZFS server (burkeng). Runs automatically via cron (default: daily at 03:00) when `enable_backup_cron: true`.

To restore that data after a rebuild, use the separate **`infra_restore`** role / `restore_infra.yml` playbook (not part of `setup_servers.yml`):

```bash
# Dry run
ansible-playbook -i inventory/hosts restore_infra.yml -e host=infra --ask-vault-pass -e restore_dry_run=true

# Full restore (starts Docker if needed; per-stack compose failures warn only)
ansible-playbook -i inventory/hosts restore_infra.yml -e host=infra --ask-vault-pass

# Data only (skip compose up)
ansible-playbook -i inventory/hosts restore_infra.yml -e host=infra --ask-vault-pass -e restore_start_stacks=false
```

After restore, re-run relevant `deploy_*` playbooks when compose/`.env`/image tags must match vault (e.g. UniFi if the host still has a stale `latest` image).

See `roles/infra_restore/README.md`.

## ZFS pool management

Pools are handled automatically:

1. Checks for already-imported pools (`zpool list`)
2. Discovers pools available for import (`zpool import`) — covers fresh installs and recovery
3. Imports from `zpools` list if provided, otherwise imports all discovered pools
4. Verifies pool health after import

If `docker_zfs_storage: true`, the playbook:
- Fails explicitly if no pool is available
- Creates a `<pool>/docker` dataset at `/var/lib/docker` if it doesn't exist
- Writes `/etc/docker/daemon.json` with the ZFS storage driver
- Ensures Docker is running

## File structure

```
roles/servers/
├── README.md
├── defaults/
│   └── main.yml               # Default variables
├── vars/
│   └── main.yml               # Vault-encrypted variables (new_user)
├── handlers/
│   └── main.yml               # Service restart handlers (avahi, docker, udev, logind)
├── meta/
│   └── main.yml               # Role metadata
├── files/
│   ├── id_ed25519             # Vault-encrypted SSH private key
│   ├── id_ed25519.pub         # Vault-encrypted SSH public key
│   ├── id_ed25519_user.pub    # Vault-encrypted extra user public key
│   ├── bashrc                 # Shell config (includes ssh-agent auto-start)
│   ├── .tmux.conf             # Tmux config
│   └── samba.service          # Avahi service file
├── templates/
│   ├── backup_pull_script.sh.j2    # Backup server: pull from ZFS
│   ├── backup_to_burkeng.sh.j2    # Infra server: push to ZFS (burkeng)
│   ├── sync_from_external.sh.j2   # Backup server: sync external drive
│   ├── disk_status.sh.j2          # ZFS/backup: disk health overview
│   ├── save_history.sh.j2         # All servers: bash history backup
│   └── ssh_config.j2              # All servers: SSH config for inter-server access
├── tasks/
│   ├── main.yml               # Entry point, conditional includes
│   ├── common.yml             # Packages, Docker, SSH, locale, timezone, smartctl
│   ├── zfs.yml                # ZFS install, pool auto-discovery/import, Docker ZFS driver
│   ├── infra.yml              # Infra: lid close ignore, backup push setup
│   ├── backup_pull.yml        # Backup server setup (pull, mounts, external)
│   ├── casaos.yml             # CasaOS installation
│   ├── add_user.yml           # Extra user creation
│   └── remove_snap.yml        # Snapd removal
└── tests/
    ├── inventory              # Test inventory
    └── test.yml               # Test playbook
```
