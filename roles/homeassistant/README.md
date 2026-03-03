Home Assistant on VLAN 30
=========

This role deploys Home Assistant with Docker Compose.

- Compose and `.env` are written to `/storage/apps/homeassistant` (default `workdir`).
- Home Assistant container runs with a static IP on VLAN 30.
- The role ensures a VLAN sub-interface exists on the host (for Docker parent interface).


Requirements
------------

- Docker + Docker Compose plugin installed on the target host.
- Host NIC uplink must carry VLAN 30 tagged.
- Router/switch must route your VLAN 30 subnet with gateway reachable from the host.


Role Variables
--------------

Main variables in `vars/main.yml`:

    ---
    workdir: /storage/apps/homeassistant
    homeassistant_use_vlan30: <true|false>
    homeassistant_env:
      IMAGE_VERSION: "<tag>"
      HOMEASSISTANT_UID: "<uid>"
      HOMEASSISTANT_GID: "<gid>"
      CONTAINER_IP: "<vlan30_ip>"
      DOCKER_HOST_NIC_VLAN30: "<parent_iface.vlan_id>"   # fallback if detection fails
      NETWORK_SUBNET: "<cidr>"
      NETWORK_GATEWAY: "<gateway_ip>"

Variables in `defaults/main.yml`:

    homeassistant_vlan30_id: "30"

Optional overrides:

- `homeassistant_vlan30_physical_nic` (for example `eth0`)
- `homeassistant_env.HOMEASSISTANT_UID` / `homeassistant_env.HOMEASSISTANT_GID` (also used as ownership for `{{ workdir }}/ha_config`)
- Put real values in encrypted host/group vars (Vault); keep README examples generic.


Generated Files
---------------

In `{{ workdir }}`:

- `compose.yaml`
- `.env`
- `ha_config/`

System files on host:

- `/usr/local/bin/vlan30-homeassistant.sh`
- `/etc/systemd/system/vlan30-homeassistant.service`


Tags
----

- `copy` - render/copy compose/env and directory setup
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
