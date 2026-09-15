Authelia
========

Deploys [Authelia](https://www.authelia.com/) via Docker Compose — a forward-auth SSO / 2FA portal
that sits in front of Nginx Proxy Manager (NPM) proxy hosts, gating access to services that have
weak or no native authentication (e.g. Jackett, Radarr, Sonarr, Transmission).

Authentication is local (file-based users, optionally with TOTP/WebAuthn 2FA) — Authelia does not
support delegating login to GitHub/Google; it can only act as an OIDC *provider* for other apps
that support "login via OIDC" natively (a separate, optional feature not covered by this role).

Requirements
------------

- Docker and Docker Compose plugin installed on target host.
- The `nginx_proxy_manager` role deployed first — Authelia joins its `proxy` external Docker
  network so NPM can reach it by container name. Nginx snippets for the NPM integration live in
  that role (`roles/nginx_proxy_manager/files/snippets/`), not here.

Role Variables
--------------

Defined in `roles/authelia/vars/main.yml`:

```yaml
workdir: "<app_workdir_path>"

authelia_owner: "<uid>"
authelia_group: "<gid>"

authelia_env:
  AUTHELIA_IMAGE_VERSION: "{{ authelia_image_version }}"
  TZ: "<timezone>"
  AUTHELIA_CONFIG_PATH: "<host_path_to_config>"
  PROXY_NETWORK_NAME: "<must_match_nginx_proxy_manager_role's_npm_env.PROXY_NETWORK_NAME>"
  # Reuse authelia_owner/authelia_group so the container's process matches the
  # ownership Ansible sets on the bind-mounted config directory:
  PUID: "{{ authelia_owner }}"
  PGID: "{{ authelia_group }}"

authelia_jwt_secret: "<random, see below>"
authelia_session_secret: "<random, see below>"
authelia_storage_encryption_key: "<random, see below>"

authelia_totp_issuer: "<your_domain>"
authelia_root_domain: "<your_domain>"
authelia_portal_hostname: "auth.<your_domain>"
authelia_default_redirection_url: "https://<landing_page>"
authelia_session_expiration: "1 hour"
authelia_session_inactivity: "5 minutes"
authelia_log_level: "info"

authelia_network_groups:
  - name: "<group_name>"
    networks:
      - "<cidr>"

authelia_access_control_rules:
  - domain: "<protected_hostname>"
    policy: "one_factor" # or two_factor, bypass, deny
    networks: # optional, references authelia_network_groups
      - "<group_name>"

authelia_users:
  - username: "<username>"
    displayname: "<display_name>"
    password_hash: "<argon2id_hash, see below>"
    email: "<email>"
    groups:
      - "<group>"
```

Only the image version default is kept in `defaults/main.yml`:

- `authelia_image_version`

Store real values in Vault-encrypted variables. Do not commit concrete secrets, password hashes,
or host-specific infrastructure values in plaintext:

    ansible-vault encrypt roles/authelia/vars/main.yml

Generating secrets
-------------------

`authelia_jwt_secret`, `authelia_session_secret`, and `authelia_storage_encryption_key` are plain
random strings (no external dependency needed):

    openssl rand -hex 64

Password hashes for `authelia_users[].password_hash` (argon2id):

    docker run --rm -it authelia/authelia:{{ authelia_image_version }} authelia crypto hash generate argon2

Adding/removing protected hosts is a one-line change to `authelia_access_control_rules` — no GUI
clicking required, matching how the rest of this repo treats deployment config as code. Keep the
portal's own hostname (`authelia_portal_hostname`) out of that list; it must never protect itself.

To let LAN clients skip the login prompt for a given host (e.g. so accessing it from home never
triggers a redirect to `auth.<your_domain>`) while still requiring auth from anywhere else, add a
network group under `authelia_network_groups` and give that domain two rules — a `bypass` rule
scoped to that network, followed by the normal fallback rule. Rules are evaluated top-to-bottom and
the first match wins, so the network-scoped rule for a domain must come *before* its general rule:

```yaml
authelia_network_groups:
  - name: "internal"
    networks:
      - "<cidr>"

authelia_access_control_rules:
  - domain: "npm.<your_domain>"
    policy: "bypass"
    networks:
      - "internal"
  - domain: "npm.<your_domain>"
    policy: "one_factor"
```

NPM integration
----------------

Authelia joins NPM's `proxy` external Docker network (see `files/compose.yaml`), so NPM reaches it
directly by container name — `http://authelia:9091` — no host IP involved, matching Authelia's own
recommended deployment (proxy and Authelia sharing a Docker network).

The `nginx_proxy_manager` role mounts `roles/nginx_proxy_manager/files/snippets/` into NPM's
container at `/snippets/` and copies in the three required files:
- `proxy.conf`
- `authelia-location.conf` (already points at `http://authelia:9091`)
- `authelia-authrequest.conf`

1. Deploy/redeploy `nginx_proxy_manager` first, so the snippets are in place and the container is
   recreated with the new `/snippets` volume.
2. Create a new NPM Proxy Host for the portal itself:
   - Domain: `auth.<your_domain>` (matches `authelia_portal_hostname`)
   - Scheme: `http`, Forward Hostname/IP: `authelia`, Forward Port: `9091`
   - SSL tab: request a certificate, Force SSL: on
   - Advanced tab:
     ```nginx
     location / {
         include /snippets/proxy.conf;
         proxy_pass $forward_scheme://$server:$port;
     }
     ```
3. For each protected app's existing/new Proxy Host, Advanced tab:
   ```nginx
   include /snippets/authelia-location.conf;

   location / {
       include /snippets/proxy.conf;
       include /snippets/authelia-authrequest.conf;
       proxy_pass $forward_scheme://$server:$port;
   }
   ```
   Do **not** use NPM's separate "Custom Locations" tab for this — locations defined there bypass
   the Authelia check entirely.

Full reference: <https://www.authelia.com/integration/proxies/nginx-proxy-manager/>

Tags
----

Running with no `--tags` at all applies `copy`, `pull`, and `restart` — the usual "apply config/image
changes" flow. `up` and `down` are opt-in only (tagged `never`), since they're for the initial
deploy or an explicit teardown, not routine changes:

- `copy` - write compose/env/config files and ensure directories exist (runs by default)
- `pull` - `docker compose pull` (runs by default)
- `restart` - `docker compose up -d --force-recreate` (runs by default)
- `up` - `docker compose up -d` (opt-in, e.g. first deploy: `--tags up`)
- `down` - `docker compose down` (opt-in: `--tags down`)

Example Playbook
----------------

```yaml
---
- hosts: "{{ host }}"
  roles:
    - authelia
```
