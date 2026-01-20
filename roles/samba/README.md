Samba File Server
=========

This role deploys a Samba file server in a Docker container using the [dockurr/samba](https://hub.docker.com/r/dockurr/samba) image. It provides SMB/CIFS file sharing with user authentication and guest access support.

Requirements
------------

- Docker and Docker Compose installed on your server
- Storage path must exist on the host system
- Network ports 445 and 139 available

Role Variables
--------------

All configuration is centralized in `vars/main.yml` (encrypted with Ansible Vault):

```yaml
---
# vars file for samba
workdir:              # Directory where docker compose files are deployed
samba_name:           # NetBIOS name and share name
samba_share_name:     # Share name (can differ from samba_name)
samba_user:           # Samba username
samba_group:          # Samba group for write access
samba_password:       # Samba user password
samba_uid:            # User ID for file ownership
samba_gid:            # Group ID for file ownership
samba_storage:        # Host path to share
```

**Note:** The `vars/main.yml` file is encrypted with Ansible Vault. To edit it:

```bash
ansible-vault edit roles/samba/vars/main.yml
```

### Generated Files

The role uses Jinja2 templates to generate configuration files from the variables above:

1. **`.env`** (from `templates/env.j2`):
   - Docker Compose environment variables
   - SAMBA_NAME, SAMBA_USER, SAMBA_PASSWORD, etc.

2. **`smb.conf`** (from `templates/smb.conf.j2`):
   - Samba server configuration
   - NetBIOS name, share settings, permissions

### Samba Configuration

The deployed Samba share has the following settings:
- **Protocol**: SMB2 minimum
- **Security**: User-based authentication
- **Guest access**: Enabled (read-only)
- **User access**: Write access for users in the `smb` group
- **Ports**: 445 (SMB), 139 (NetBIOS)

Dependencies
------------

None.

Example Playbook
----------------

```yaml
---
- hosts: file_servers
  roles:
    - samba
```

### Available Tags

- `copy` - Copy configuration files only
- `up` - Start the Samba container
- `down` - Stop the Samba container
- `pull` - Pull the latest image
- `restart` - Recreate and restart the container
- `test` - Display host network information

### Common Operations

**Deploy Samba:**
```bash
ansible-playbook deploy_samba.yml
```

**Update configuration and restart:**
```bash
ansible-playbook deploy_samba.yml --tags copy,restart
```

**Stop Samba:**
```bash
ansible-playbook deploy_samba.yml --tags down
```

**Update image and restart:**
```bash
ansible-playbook deploy_samba.yml --tags pull,restart
```

### Accessing the Share

After deployment, the Samba share will be accessible at:

**Windows:**
```
\\your-server\<share_name>
```

**macOS:**
```
smb://your-server/<share_name>
```

**Linux:**
```bash
# Mount the share
sudo mount -t cifs //your-server/<share_name> /mnt/share -o username=<samba_user>,password=your_password

# Or in fstab:
//your-server/<share_name> /mnt/share cifs username=<samba_user>,password=your_password,uid=1000,gid=1000 0 0
```

**Guest access (read-only):**
```
smb://your-server/<share_name>
# No credentials required for read access
```

### Troubleshooting

**Check container logs:**
```bash
docker logs samba
```

**Check if ports are open:**
```bash
netstat -tulpn | grep -E '445|139'
```

**Test connectivity:**
```bash
smbclient -L your-server -U <samba_user>
```

License
-------

BSD

Author Information
------------------

Valentin Georgiev valyo@me.com
