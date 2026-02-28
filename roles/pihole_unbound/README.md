Pi-hole as All-Around DNS Solution
=========

This role installs Pi-hole and Unbound in one Docker container. It uses the image built in [GitHub repo](https://github.com/valyo/docker-pihole-unbound)


Requirements
------------


- When running the container in a ´macvlan´ docker network (not sure if it is really better than than using a default bridge network), the host-container communication issue can be solved like this: [Using Docker macvlan networks](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/); additional info on the subject: [Docker Macvlan network inside container is not reaching to its own host](https://stackoverflow.com/questions/49600665/docker-macvlan-network-inside-container-is-not-reaching-to-its-own-host)

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

Variables in vars/main.yml are:

    ---
    # vars file for pihole_unbound
    workdir:              # Directory on the host where docker compose files are copied
    pihole_data_root:     # Host path for Pi-hole data. Contains pihole/ and dnsmasq.d/ subdirs.
    macvlan_name:         # Name of the macvlan interface
    macvlan_ip:           # IP address for the macvlan
    macvlan_route:        # Static route to add

Variables in .env file are (see [my image repo]((https://github.com/valyo/docker-pihole-unbound)) and [official pihole container repo](https://github.com/pi-hole/docker-pi-hole) for all variables):

    FTLCONF_LOCAL_IPV4=
    TZ=
    WEBPASSWORD=
    REV_SERVER=
    REV_SERVER_DOMAIN=
    REV_SERVER_TARGET=
    REV_SERVER_CIDR=
    VIRTUAL_HOST=       # domain name fo accessing the interface instead of pi.hole
    PIHOLE_DNS_=
    DNSSEC=
    HOSTNAME=           # container hostname
    PIHOLE_DOMAIN=      # contained domain
    DOCKER_HOST_NIC=
    DOCKER_NETWORK_SUBNET=
    DOCKER_NETWORK_GATEWAY=
    PIHOLE_DATA_ROOT=    # host path for Pi-hole data (e.g. /storage/apps/pihole)


Migration from Docker named volumes to bind mount
-------------------------------------------------

Follow these steps **in order** to move Pi-hole data from Docker named volumes to `/storage/apps/pihole`.

**All commands below are on the Pi-hole host** unless noted.

1. **Stop the stack** (from your compose workdir, e.g. `~/pihole_unbound`):

   ```bash
   docker compose down
   ```

2. **Create the host directories** (if they don't exist):

   ```bash
   sudo mkdir -p /storage/apps/pihole/pihole /storage/apps/pihole/dnsmasq.d
   ```

3. **Copy data from the old volumes** into the new paths:

   ```bash
   sudo docker run --rm -v etc_pihole-unbound:/from -v /storage/apps/pihole/pihole:/to alpine sh -c "cp -a /from/. /to/"
   sudo docker run --rm -v etc_pihole_dnsmasq-unbound:/from -v /storage/apps/pihole/dnsmasq.d:/to alpine sh -c "cp -a /from/. /to/"
   ```

4. **Set ownership** so the container (UID 1000) can read and write:

   ```bash
   sudo chown -R 1000:1000 /storage/apps/pihole/pihole /storage/apps/pihole/dnsmasq.d
   ```

5. **From your workstation**, run the playbook so the role deploys the compose and .env that use the bind mount:

   ```bash
   ansible-playbook deploy_pihole_unbound.yml -e host=infra
   ```

6. **On the host**, start the stack again (from the compose workdir):

   ```bash
   docker compose up -d
   ```

7. **Check** Pi-hole (UI, DNS). When everything works, **optionally** remove the old volumes:

   ```bash
   docker volume rm etc_pihole-unbound etc_pihole_dnsmasq-unbound
   ```


Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

BSD

Author Information
------------------

Valentin Georgiev valyo@me.com
