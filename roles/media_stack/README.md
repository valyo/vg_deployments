Media Stack
=========

Deploys a Docker Compose media stack (Transmission, Radarr, Sonarr, Jackett, FlareSolverr, Prowlarr, Jellyfin, optional Gluetun VPN).

Requirements
------------

- Docker and Docker Compose plugin installed on target host.
- Host bind paths configured in env values (`TX_DOWNLOADS`, `RADARR_DATA`, `SONARR_DATA`, `JELLYFIN_DATA`).
- If using VPN profile, valid VPN credentials in env values.

Role Variables
--------------

Defined in `roles/media_stack/vars/main.yml`:

```yaml
workdir: "<app_workdir_path>"
media_stack_profile: "<stack_profile>"

media_stack_env:
  PUID: "<uid>"
  PGID: "<gid>"
  TZ: "<timezone>"
  OPENVPN_USER: "<set-in-vault>"
  OPENVPN_PASSWORD: "<set-in-vault>"
  TX_USER: "<set-in-vault>"
  TX_PASSWORD: "<set-in-vault>"
  TX_DOWNLOADS: "<downloads_bind_path>"
  RADARR_DATA: "<radarr_data_bind_path>"
  SONARR_DATA: "<sonarr_data_bind_path>"
  JELLYFIN_CONFIG: "<jellyfin_config_bind_path>"
  JELLYFIN_DATA: "<jellyfin_data_bind_path>"
  JELLYFIN_IP: "<iot_vlan_static_ip>"
  COMPOSE_PROJECT_NAME: "<compose_project_name>"
  FLARESOLVERR_IMAGE_VERSION: "{{ media_stack_flaresolverr_image_version }}"
  FLARESOLVERR_LOG_LEVEL: "{{ media_stack_flaresolverr_log_level }}"
```

Only image version defaults are kept in `defaults/main.yml`:

- `media_stack_gluetun_image_version`
- `media_stack_transmission_image_version`
- `media_stack_radarr_image_version`
- `media_stack_sonarr_image_version`
- `media_stack_jackett_image_version`
- `media_stack_prowlarr_image_version`
- `media_stack_jellyfin_image_version`
- `media_stack_flaresolverr_image_version`
- `media_stack_flaresolverr_log_level`

`media_stack_env` is rendered to `{{ workdir }}/.env` via `templates/env.j2`.
Use a fixed `COMPOSE_PROJECT_NAME` to keep Docker named volumes stable across different directories/deploy paths.

Store real values in Vault-encrypted variables and override there. Do not commit concrete credentials, tokens, or host-specific infrastructure values in plaintext.

Runtime user
------------

LinuxServer containers use `PUID` and `PGID` environment variables for runtime permissions.
The `vpn` container requires elevated network capabilities (`NET_ADMIN`).

Jellyfin network
----------------

Jellyfin is attached to an external Docker network with static IP:

- `JELLYFIN_IP` (set to your reserved IoT VLAN address)

FlareSolverr / Jackett indexer proxy
-------------------------------------

`flaresolverr` runs alongside Jackett (profiles `jackett`, `stack-1`) and lets Jackett solve
Cloudflare-style challenges for indexers that require it, per
[Jackett's FlareSolverr docs](https://github.com/Jackett/Jackett#configuring-flaresolverr).

It is deliberately **not** published to a host port — FlareSolverr's own docs warn against
exposing it to the internet, so it's only reachable from other containers on the compose
project's default network, at `http://flaresolverr:8191`.

After deploying, configure it once in the Jackett UI (this is a Jackett server setting, not
something this role manages):

1. Open Jackett -> Settings.
2. Set **FlareSolverr API URL** to `http://flaresolverr:8191`.
3. Leave **FlareSolverr Max Timeout** at its default unless an indexer needs more time.
4. Save. Indexers that need it will now route challenge-solving through FlareSolverr automatically.

Tags
----

- `copy` - write compose/env and ensure workdir
- `up` - `docker compose --profile {{ media_stack_profile }} up -d`
- `down` - `docker compose --profile {{ media_stack_profile }} down`
- `pull` - `docker compose --profile {{ media_stack_profile }} pull`
- `restart` - `docker compose --profile {{ media_stack_profile }} up -d --force-recreate`

Example Playbook
----------------

```yaml
---
- hosts: "{{ host }}"
  roles:
    - media_stack
```
