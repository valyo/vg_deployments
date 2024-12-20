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
