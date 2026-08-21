# Trading Log

Deploys the Trend Trading Log app with Docker Compose on infra.

The role manages:

- `compose.yaml`
- Generated `.env` from Ansible variables
- Persistent data directory `{{ workdir }}/data` (bind-mounted to `/app/data`, holds `trades.db`)
- Optional seeded SQLite DB file

## Requirements

- Docker and Docker Compose plugin installed on target host
- Access to `ghcr.io/valyo/trading-log` (public image, built and published by
  [`.github/workflows/docker-publish.yml`](https://github.com/valyo/trading-log) on push to `main`
  and on `v*.*.*` tags)
- Real environment values stored in Vault-encrypted vars

## Role Variables

Defined in `roles/trading_log/vars/main.yml`:

```yaml
workdir: /storage/apps/trading_log
trading_log_seed_db_file: ""  # optional file in roles/trading_log/files/

trading_log_env:
  IMAGE_VERSION: "<tag>"
  TRADING_LOG_UID: "<uid>"
  TRADING_LOG_GID: "<gid>"
  HOST_PORT: "<host_port>"
  NEXTAUTH_SECRET: "<set-in-vault>"
  NEXTAUTH_URL: "<set-in-vault>"     # public URL, e.g. https://trades.example.com
  GOOGLE_CLIENT_ID: "<set-in-vault>"
  GOOGLE_CLIENT_SECRET: "<set-in-vault>"
  ALLOWED_GOOGLE_EMAILS: "<set-in-vault>"
```

Only the image version default is kept in `defaults/main.yml`:

- `trading_log_image_version` (defaults to `latest`; pin to a `vX.Y.Z` tag once a release exists)

`trading_log_env` is rendered into `{{ workdir }}/.env` via `templates/env.j2`.

Store real values (NextAuth secret, Google OAuth client ID/secret, allowed emails, public URL)
in Vault-encrypted variables and override them there. Do not commit concrete credentials, tokens,
or host-specific infrastructure values in plaintext.

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
    - trading_log
```

## Operational Notes

- Container listens on port `3000` internally, exposed via `HOST_PORT` on the host.
- SQLite DB lives at `{{ workdir }}/data/trades.db` (single `./data:/app/data` volume).
- The app runs as `TRADING_LOG_UID:TRADING_LOG_GID` inside the container; the data directory
  is created with matching ownership so SQLite writes succeed.
- Set `NEXTAUTH_URL` to the externally reachable URL (used for the Google OAuth callback:
  `<NEXTAUTH_URL>/api/auth/callback/google`) and register that redirect URI in the Google
  Cloud Console OAuth client.
- Point the reverse proxy (nginx_proxy_manager) at the host's `HOST_PORT`; this role does not
  register the proxy host itself.
