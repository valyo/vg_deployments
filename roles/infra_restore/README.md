# infra_restore

Restores infra server app data from the burkeng (ZFS) backup produced by
`backup_to_burkeng.sh`.

## How it maps to backup

Backup pushes each path in `backup_paths` into `backup_target_dir/` on burkeng
(directory name preserved):

| Local (`backup_paths`) | On burkeng |
|---|---|
| `/storage/apps/homeassistant` | `…/backups/infra/homeassistant/` |
| `/storage/apps/npm` | `…/backups/infra/npm/` |

Restore rsyncs each remote dir **into** the matching local path
(`…/homeassistant/` → `/storage/apps/homeassistant/`), not into the parent.

## Cleaning up a bad restore

If an older version flattened app contents into `/storage/apps/`, wipe the
mess and re-run (backup on burkeng is unchanged):

```bash
# On infra — only if /storage/apps is the flattened mess, not real app dirs
sudo rm -rf /storage/apps
sudo mkdir -p /storage/apps
sudo chown "$USER:$USER" /storage/apps
```

Then re-run the restore playbook.

## Requirements

- Infra already bootstrapped (`setup_servers.yml`) so SSH keys and Docker exist
- Vault vars in `inventory/host_vars/infra/vault.yml`:
  - `backup_target_host`
  - `backup_target_dir`
  - `backup_paths`
- Reachable burkeng with the same SSH key used for backups
  (`~/.ssh/id_ed25519_servers`)

## Variables

| Variable | Default | Description |
|---|---|---|
| `restore_rsync_opts` | `-avz --numeric-ids --progress` | Rsync flags (no `--delete` by default — safer than backup) |
| `restore_rsync_delete` | `false` | Pass `--delete` (mirror restore) |
| `restore_dry_run` | `false` | Preview only |
| `restore_stop_stacks` | `true` | `docker compose down` before restore |
| `restore_start_stacks` | `true` | `docker compose up -d` after restore (starts Docker if needed; per-stack failures warn but do not fail the play). Use `false` for data-only restore. |
| `restore_run_via_ansible` | `true` | Run restore via playbook (false = install script only) |
| `restore_paths` | `[]` | Subset of `backup_paths` (empty = all) |
| `restore_compose_dirs` / `infra_restore_compose_dirs` | `[]` | Compose dirs to stop/start (empty = auto-discover) |

Ownership is preserved by rsync `--numeric-ids` (same UIDs as on burkeng). No post-restore chown.

Infra backup uses `--delete` so burkeng mirrors the source; restore omits `--delete` unless `restore_rsync_delete: true`.

## Usage

```bash
# Dry run
ansible-playbook -i inventory/hosts restore_infra.yml -e host=infra --ask-vault-pass -e restore_dry_run=true

# Full restore
ansible-playbook -i inventory/hosts restore_infra.yml -e host=infra --ask-vault-pass

# Install on-host script only
ansible-playbook -i inventory/hosts restore_infra.yml -e host=infra --ask-vault-pass \
  -e restore_run_via_ansible=false --tags script
```

`--tags script` skips preflight validation; vault `backup_*` vars are still required to render the script.

On the host after the script is installed:

```bash
sudo -u "$USER" /usr/local/bin/restore_from_burkeng.sh --dry-run
sudo -u "$USER" /usr/local/bin/restore_from_burkeng.sh
```

## Recommended recovery order

1. Bootstrap infra: `setup_servers.yml -e host=infra`
2. Dry-run restore, then restore with this role
3. Re-run deploy playbooks so compose/`.env` match vault (NPM requires data dirs first)

Auto-start after restore only runs where a `compose.yaml` / `docker-compose.yaml` already exists under the restored paths. If compose files were missing from the backup (or need vault refresh), use `deploy_*` — start alone may be a no-op for those apps.

```bash
ansible-playbook -i inventory/hosts deploy_homeassistant.yml -e host=infra --ask-vault-pass
ansible-playbook -i inventory/hosts deploy_pihole_unbound.yml -e host=infra --ask-vault-pass
ansible-playbook -i inventory/hosts deploy_npm.yml -e host=infra --ask-vault-pass
# ...other deploy_* playbooks as needed
```

## Tags

`restore`, `preflight`, `script`, `postflight`

## Scripts deployed

**`/usr/local/bin/restore_from_burkeng.sh`** — Inverse of `backup_to_burkeng.sh`.
Pulls each restored path from burkeng via rsync over SSH.
