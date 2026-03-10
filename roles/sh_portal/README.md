SH Portal
=========

Deploys SH Portal for production with Docker Compose.

The role manages:
- Docker Compose file
- Generated `.env` from Ansible variables
- Data directories under `{{ workdir }}/data`
- Optional seeded SQLite DB file

Requirements
------------

- Docker and Docker Compose installed on target host
- Access to `ghcr.io/valyo/sh-portal`
- `service-account.json` in role files if Google Sheets integration is enabled

Role Variables
--------------

Defined in `roles/sh_portal/vars/main.yml`:

```yaml
workdir: /storage/apps/sh_portal
sh_portal_seed_db_file: ""  # optional file in roles/sh_portal/files/

sh_portal_env:
  IMAGE_VERSION: "<tag>"
  SH_PORTAL_UID: "<uid>"
  SH_PORTAL_GID: "<gid>"
  PORT: "<port>"
  DB_NAME: "<sqlite_file_name>"
  DATABASE_URL: "sqlite:///<sqlite_file_name>"
  FLASK_DEBUG: "<True|False>"
  BASE_URL: "<public_base_url>"
  MAIL_BACKEND: "<mailcatcher|google>"
  # ... see vars/main.yml for full list
```

`sh_portal_env` is rendered into `{{ workdir }}/.env` via `templates/env.j2`.

If you store sensitive values in Vault, keep the same keys in `sh_portal_env` and override them in encrypted host/group vars.

Tags
----

- `copy` - create/update files and directories
- `up` - `docker compose up -d`
- `down` - `docker compose down`
- `pull` - `docker compose pull`
- `restart` - `docker compose up -d --force-recreate`

Example Playbook
----------------

```yaml
---
- hosts: "{{ host }}"
  roles:
    - sh_portal
```

Operational Notes
-----------------

- SH Portal is exposed on `PORT` (default `8087`)
- Mailcatcher SMTP/Web ports are controlled by:
  - `MAILCATCHER_SMTP_PORT` (default `1025`)
  - `MAILCATCHER_WEB_PORT` (default `1088`)
- Mail behavior follows app logic in `__init__.py`:
  - set `MAIL_BACKEND` to switch mode (`google` or `mailcatcher`)
  - for `google`, set `MAIL_USERNAME` and `MAIL_PASSWORD` (app password)
  - SMTP host/port/TLS are taken from app defaults
- Mailcatcher runs only in the optional Compose `dev` profile.
- Set `BASE_URL` to the externally reachable URL for OAuth callback/login flow.
