# Sheep Journal

Deploys [Sheep Journal](https://github.com/valyo/sheep-journal) (stalljournal för får och getter) with Docker Compose.

The role manages:

- `compose.yaml`
- Generated `.env` from Ansible variables
- Persistent data directory `{{ workdir }}/data/instance` (bind-mounted to `/app/instance`), holding
  the SQLite database `sheep_journal.sqlite`, animal photos in `uploads/` and import reports
- Optional seeded SQLite DB file

## Requirements

- Docker and Docker Compose plugin installed on target host
- Access to `ghcr.io/valyo/sheep-journal` (built and published by
  [`.github/workflows/docker-build.yml`](https://github.com/valyo/sheep-journal/blob/main/.github/workflows/docker-build.yml)
  on push to `main`, tagged with the app's `VERSION` and `latest`)
- Real environment values stored in Vault-encrypted vars

## Role Variables

Defined in `roles/sheep_journal/vars/main.yml` (Vault-encrypted):

```yaml
workdir: /storage/apps/sheep_journal
sheep_journal_seed_db_file: ""  # optional file in roles/sheep_journal/files/

sheep_journal_env:
  IMAGE_VERSION: "{{ sheep_journal_image_version }}"
  SHEEP_JOURNAL_UID: "<uid>"
  SHEEP_JOURNAL_GID: "<gid>"
  PORT: "<host_port>"
  SECRET_KEY: "<set-in-vault>"     # at least 32 random characters
  SHEEP_JOURNAL_TZ: "Europe/Stockholm"
```

Only the image version default is kept in `defaults/main.yml`:

- `sheep_journal_image_version` (the app's `VERSION`, e.g. `0.1.0`; tags have no `v` prefix)

`sheep_journal_env` is rendered into `{{ workdir }}/.env` via `templates/env.j2`.
The role refuses to run if `SECRET_KEY` is missing or shorter than 32 characters; the app
itself also refuses to start without one.

## Tags

- `copy` - create/update files and directories
- `up` - `docker compose up -d`
- `down` - `docker compose down`
- `pull` - `docker compose pull`
- `restart` - `docker compose up -d --force-recreate`

Typical deploy/update: `--tags copy,pull,restart`.

## Example Playbook

```yaml
---
- hosts: "{{ host }}"
  roles:
    - sheep_journal
```

## Operational Notes

- Container listens on port `8000` internally (gunicorn), exposed via `PORT` on the host.
- The app runs as `SHEEP_JOURNAL_UID:SHEEP_JOURNAL_GID`; the data directories are created with
  matching ownership so SQLite and photo uploads can write.
- Database migrations run automatically when the container starts.
- **The app has no login.** Put it behind nginx_proxy_manager with Authelia (or keep it on the
  LAN only); this role does not register the proxy host itself.
- Seeding: set `sheep_journal_seed_db_file` to a file in `roles/sheep_journal/files/` (e.g. a
  copy of a local `instance/sheep_journal.sqlite`). It is copied only when no database exists,
  so a later run never overwrites live data. Animal photos are not seeded: copy a local
  `instance/uploads/` into `{{ workdir }}/data/instance/uploads/` by hand.
- Backups: `/storage/apps/sheep_journal` (database and photos together) is picked up
  automatically by the infra backup, which backs up every directory under `backup_root`.
