Solberg Honung portal for management of bookings, customers and invoicing
=========

This role installs SH_portal and Mailcather (temporarily) in separate Docker containers. It uses the image built in [GitHub repo](https://github.com/valyo/sh-portal)

The deployment includes:
- **SH Portal** application (Flask-based web app)
- **Mailcatcher** (for local email testing during development)

Requirements
------------

- Docker and Docker Compose installed on your server
- Access to the GitHub Container Registry (ghcr.io)
- Google Sheets API credentials (service-account.json) if using Google Sheets integration

Role Variables
--------------

Variables in `vars/main.yml`:

```yaml
---
# vars file for sh_portal
workdir: workdir/    # Directory on the host where docker compose files are copied
```

Variables in `.env` file (this file is encrypted with Ansible Vault in `files/.env`):

```bash
# Application configuration
PORT=8087
FLASK_SECRET_KEY=
FLASK_APP=main.py
FLASK_ENV=production
HOSTNAME=sh_portal

# Database configuration
DB_NAME=filename.db
DATABASE_URL=sqlite:///${DB_NAME}

# Mail configuration
MAIL_SERVER=mailcatcher      # Use 'mailcatcher' for dev, 'smtp.gmail.com' for production
MAIL_PORT=1025                # Use 1025 for mailcatcher, 587 for production SMTP
MAIL_USE_TLS=False            # Set to True for production SMTP
MAIL_USE_SSL=False
MAIL_USERNAME=                # Required for production SMTP
MAIL_PASSWORD=                # Required for production SMTP

# GitHub OAuth configuration (required for authentication)
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
OAUTH_REDIRECT_URI=https://your-domain.com/callback

# Google Sheets integration (optional)
GOOGLE_APPLICATION_CREDENTIALS=/app/credentials/service-account.json
```

**Note:** The `.env` file is stored encrypted with Ansible Vault. To edit it:

```bash
ansible-vault edit roles/sh_portal/files/.env
```

**Important Notes:**

- **GitHub OAuth App Setup**: Create an OAuth App at https://github.com/settings/developers
  - Homepage URL: `https://your-domain.com`
  - Authorization callback URL: `https://your-domain.com/callback` (no port if using reverse proxy)
  
- **Files to place in directories:**
  - `service-account.json`: Place in `data/credentials/` directory (for Google Sheets integration)
  
- **When using Nginx Proxy Manager or reverse proxy:**
  - `OAUTH_REDIRECT_URI` should use standard HTTPS port (443), not the internal port (8087)
  - Example: `https://shportal.example.com/callback` not `https://shportal.example.com:8087/callback`

- **After changing `.env` file according to your setup, for manual start:**
  ```bash
  cd workdir
  docker compose down
  docker compose up -d
  ```


Dependencies
------------

None.

Example Playbook
----------------

```yaml
---
- hosts: - hosts: "{{ host }}"
  roles:
    - sh_portal
```

After deployment, the application will be accessible at:
- **SH Portal**: `http://your-server:8087` (or custom PORT from .env)
- **Mailcatcher Web UI**: `http://your-server:1088` (for development/testing)

License
-------

BSD

Author Information
------------------

Valentin Georgiev valyo@me.com
