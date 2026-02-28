Home Assistant on VLAN 30
=========

This role deploys Home Assistant with Docker Compose.

- Compose and `.env` are written to `/storage/apps/homeassistant` (default `workdir`).
- Home Assistant container runs with a static IP on VLAN 30 (default `192.168.30.3`).
- The role ensures a VLAN sub-interface exists on the host (for Docker parent interface).
- IPv6 is disabled in the container and `gai.conf` is provided to prefer IPv4.


Requirements
------------

- Docker + Docker Compose plugin installed on the target host.
- Host NIC uplink must carry VLAN 30 tagged.
- Router/switch must route VLAN 30 (`192.168.30.0/24`) with gateway reachable from the host.


Role Variables
--------------

Main variables in `vars/main.yml`:

    ---
    workdir: /storage/apps/homeassistant
    homeassistant_use_vlan30: true
    homeassistant_env:
      IMAGE_VERSION: "stable"
      CONTAINER_IP: "192.168.30.3"
      DOCKER_HOST_NIC_VLAN30: "enp2s0.30"   # fallback if detection fails
      NETWORK_SUBNET: "192.168.30.0/24"
      NETWORK_GATEWAY: "192.168.30.1"

Variables in `defaults/main.yml`:

    homeassistant_vlan30_id: "30"

Optional overrides:

- `homeassistant_vlan30_physical_nic` (for example `enp2s0f0`)
- `homeassistant_config_owner`
- `homeassistant_config_group`


Generated Files
---------------

In `{{ workdir }}`:

- `compose.yaml`
- `.env`
- `gai.conf`
- `config/`

System files on host:

- `/usr/local/bin/vlan30-homeassistant.sh`
- `/etc/systemd/system/vlan30-homeassistant.service`


Tags
----

- `copy` - render/copy compose/env/gai.conf and directory setup
- `up` - run `docker compose up -d`
- `restart` - run `docker compose up -d --force-recreate`
- `upgrade` - run upgrade-only tasks
  - includes:
    - `down` (`docker compose down`)
    - `pull` (`docker compose pull`)

Notes:

- `down` and `pull` are tagged with `never` and only run when explicitly selected (for example `--tags upgrade` or `--tags down` / `--tags pull`).


Example Playbook
----------------

`deploy_homeassistant.yml`:

    - hosts: "{{ host }}"
      roles:
        - homeassistant

Run:

    ansible-playbook deploy_homeassistant.yml -e host=infra

Upgrade flow:

    ansible-playbook deploy_homeassistant.yml -e host=infra --tags upgrade

