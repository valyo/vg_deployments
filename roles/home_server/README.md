# home_server

Ansible role for bootstrapping and configuring home servers. Supports three server types via `server_role`: **zfs**, **infra**, and **backup**.

## Server types

### zfs_server (primary)

Primary storage server with ZFS pools. Runs Docker (with ZFS storage driver), CasaOS, and hosts the main data in `/burkpool`. Also provides a passwordless `sudo rsync` rule so the backup server can pull data remotely.

### infra_server (laptop)

Infrastructure server running Docker containers on a laptop. Lid close events are ignored so the server stays running when closed. Pushes local backup data to the ZFS server on a cron schedule.

### backup_server

Offsite/secondary backup server. Pulls `/burkpool` from the ZFS server via rsync. Also manages an external exFAT drive for cold storage archival. Does not install Docker directly — CasaOS brings in its own Docker dependency.

## What the role installs (all servers)

- Common packages: htop, vim, lm-sensors, rsync, tmux, tcpdump, dnsutils, git-core, nmap, smartmontools, etc.
- Passwordless `sudo smartctl` for disk monitoring
- Locale (en_US.UTF-8), timezone, NTP
- SSH keys, bashrc, tmux config and plugins
- Avahi/Samba service discovery
- CasaOS
- Snapd removal

## Conditional features

| Feature | Variable | Default | Servers |
|---|---|---|---|
| Docker | `install_docker` | `true` | zfs, infra |
| CasaOS | `install_casaos` | `true` | all |
| ZFS | `install_zfs` | `false` | zfs |
| HFS+ support | `install_hfs` | `false` | zfs, backup |
| Extra user | `add_extra_user` | `false` | zfs |
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

### Sensitive variables (vault-encrypted)

Stored in `inventory/host_vars/<hostname>/vault.yml`:

- `ansible_host` — server IP address
- `ansible_user` — SSH user
- `backup_source_host` — ZFS server IP (backup_server only)
- `backup_target_host` — ZFS server IP (infra_server only)
- `external_drive_uuid` — UUID of external backup drive (backup_server only)

Stored in `roles/home_server/vars/main.yml` (vault-encrypted):

- `new_user` — username for the extra user (used when `add_extra_user: true`)

SSH keys in `roles/home_server/files/` are also vault-encrypted.

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
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=zfs_server --ask-vault-pass
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=infra_server --ask-vault-pass
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=backup_server --ask-vault-pass
```

The inventory can also be used with other roles in this repo:

```bash
ansible-playbook -i inventory/hosts deploy_samba.yml -e host=burkeng --ask-vault-pass
```

### Using tags

Run specific sections only:

```bash
# Common setup only
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=backup_server --ask-vault-pass --tags common

# ZFS tasks only
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=zfs_server --ask-vault-pass --tags zfs

# Backup tasks only
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=backup_server --ask-vault-pass --tags backup

# Disk mounts only
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=backup_server --ask-vault-pass --tags mounts

# External drive setup only
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=backup_server --ask-vault-pass --tags external

# Infra laptop config only
ansible-playbook -i inventory/hosts setup_home_server.yml -e host=infra_server --ask-vault-pass --tags infra
```

Available tags: `common`, `bootstrap`, `apt`, `ssh-keys`, `zfs`, `backup`, `mounts`, `external`, `snap`, `users`, `casaos`, `docker`, `infra`, `laptop`.

## Scripts deployed on servers

### ZFS and backup servers

**`/usr/local/bin/disk_status.sh`** — Shows model, capacity, and temperature for all drives.

```bash
disk_status.sh
```

### Backup server

**`/usr/local/bin/backup_from_primary.sh`** — Pulls the entire `/burkpool` from the ZFS server into `/backup_burkpool` using rsync over SSH. Runs as root (uses `sudo rsync` on the remote ZFS server to read all files with preserved ownership).

```bash
sudo /usr/local/bin/backup_from_primary.sh           # full sync
sudo /usr/local/bin/backup_from_primary.sh --dry-run  # preview only
```

**`/usr/local/bin/sync_from_external.sh`** — Mounts the external exFAT drive, syncs its contents into `/cold_storage/archive_backup/`, then unmounts. Runs as your regular user (uses sudo only for mount/umount).

```bash
/usr/local/bin/sync_from_external.sh           # full sync
/usr/local/bin/sync_from_external.sh --dry-run  # preview only
```

### Infra server

**`/usr/local/bin/backup_to_primary.sh`** — Pushes local paths (defined in `backup_paths`) to the ZFS server. Runs automatically via cron (default: daily at 03:00) when `enable_backup_cron: true`.

## File structure

```
roles/home_server/
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
│   ├── bashrc                 # Shell config
│   ├── .tmux.conf             # Tmux config
│   └── samba.service          # Avahi service file
├── templates/
│   ├── backup_pull_script.sh.j2    # Backup server: pull from ZFS
│   ├── backup_script.sh.j2        # Infra server: push to ZFS
│   ├── sync_from_external.sh.j2   # Backup server: sync external drive
│   └── disk_status.sh.j2          # ZFS/backup: disk health overview
├── tasks/
│   ├── main.yml               # Entry point, conditional includes
│   ├── common.yml             # Packages, Docker, SSH, locale, timezone, smartctl
│   ├── zfs.yml                # ZFS install, pool import, Docker ZFS driver
│   ├── infra.yml              # Infra: lid close ignore, backup push setup
│   ├── backup_pull.yml        # Backup server setup (pull, mounts, external)
│   ├── casaos.yml             # CasaOS installation
│   ├── add_user.yml           # Extra user creation
│   └── remove_snap.yml        # Snapd removal
└── tests/
    ├── inventory              # Test inventory
    └── test.yml               # Test playbook
```
