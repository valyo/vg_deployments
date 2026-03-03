UniFi Network Application
=========

This role deploys UniFi Network Application with Docker Compose, including a local MongoDB container.

- Compose and `.env` are written to `/storage/apps/unifi-network` (default `workdir`).
- Persistent bind mounts are created under `{{ workdir }}`:
  - `{{ unifi_env.UNIFI_CONFIG_PATH }}`
  - `{{ unifi_env.MONGO_DB_PATH }}`
  - `{{ unifi_env.MONGO_INIT_SCRIPT_PATH }}`


Requirements
------------

- Docker + Docker Compose plugin installed on target host.
- Ports required by UniFi Network must be reachable on the host:
  - `8443/tcp` (UI/API)
  - `8080/tcp` (inform)
  - `3478/udp` (STUN)
  - `10001/udp` (device discovery)
  - `8880/tcp`, `8843/tcp` (guest portals)
  - `6789/tcp` (speed test)
  - `5514/udp` (remote syslog)


Role Variables
--------------

Variables in `vars/main.yml`:

    ---
    workdir: /storage/apps/unifi-network
    unifi_env:
      IMAGE_VERSION: "<tag>"
      MONGO_IMAGE_VERSION: "<tag>"
      PUID: "<uid>"
      PGID: "<gid>"
      TZ: "<timezone>"
      MONGO_USER: "<db_user>"
      MONGO_PASS: "<db_password>"
      MONGO_PORT: "<port>"
      MONGO_DBNAME: "<db_name>"
      MONGO_STAT_DBNAME: "<stat_db_name>"
      MONGO_DB_PATH: "<host_path_to_db>"
      MONGO_INIT_SCRIPT_PATH: "<host_path_to_init_script>"
      UNIFI_CONFIG_PATH: "<host_path_to_unifi_config>"
      MEM_LIMIT: "<mb>"
      MEM_STARTUP: "<mb>"
      MONGO_TLS: "<true|false|empty>"
      MONGO_AUTHSOURCE: "<authsource|empty>"
      BIND_PRIV: "<true|false>"
      RUNAS_UID0: "<true|false>"
      UNIFI_UI_PORT: "<port>"
      UNIFI_INFORM_PORT: "<port>"
      UNIFI_STUN_PORT: "<port>"
      UNIFI_DISCOVERY_PORT: "<port>"
      UNIFI_GUEST_HTTP_PORT: "<port>"
      UNIFI_GUEST_HTTPS_PORT: "<port>"
      UNIFI_SPEEDTEST_PORT: "<port>"
      UNIFI_SYSLOG_PORT: "<port>"

Optional:

- `unifi_data_dir_owner`
- `unifi_data_dir_group`

Recommended:

- Encrypt `roles/unifi_network/vars/main.yml` with Ansible Vault (contains DB credentials).
- Keep README examples generic; store real values in Vault-encrypted host/group vars.


Tags
----

- `copy` - write compose/env and ensure data directories
- `up` - `docker compose up -d`
- `down` - `docker compose down`
- `pull` - `docker compose pull`
- `restart` - `docker compose up -d --force-recreate`


Example Playbook
----------------

`deploy_unifi_network.yml`:

    - hosts: "{{ host }}"
      roles:
        - unifi_network

Run:

    ansible-playbook deploy_unifi_network.yml -e host=infra


Migration from existing deployment
----------------------------------

If this is not a fresh deployment, copy existing bind mount data into `{{ workdir }}` before starting:

- `config/`
- `mongo/db/`
- existing `init-mongo.js` (if you already have one)

Important:

- The role writes `init-mongo.js` with `force: false`, so an existing file is preserved.
- Keep Mongo-related env values (`MONGO_USER`, `MONGO_PASS`, `MONGO_DBNAME`, `MONGO_AUTHSOURCE`) matching your old deployment.
