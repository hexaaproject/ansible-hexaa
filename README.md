# ansible-hexaa

Ansible playbook for installing a complete HEXAA environment.

# Table of Contents

- [ansible-hexaa](#ansible-hexaa)
- [Table of Contents](#table-of-contents)
- [Usage](#usage)
    - [Local environment](#local-environment)
        - [Hostname](#hostname)
        - [Running](#running)
        - [Web access](#web-access)
        - [Possible errors](#possible-errors)
    - [Production environment](#production-environment)
        - [Running](#running-1)
        - [Backup](#backup)
- [Development](#development)
- [Architecture](#architecture)

# Usage

Requirements:
- **target and control:** Python3 with pip (`python3-pip` on Debian-based distros)
- **target**: a recent version of Docker engine (`docker.io` or
  `docker-ce` package) and Python packages
  (`pip3 install docker pyopenssl`), the playbook installs these unless
  you skip the `docker` tag
- **control machine:** Ansible (`pip3 install ansible\>=2.8`)

Installing the roles:

```sh
ansible-galaxy install -r requirements.yml -p ./roles
```


## Local environment

This setup is intended for local development and trying out HEXAA's
features. The `local.yml` playbook installs:
* the backend and frontend of HEXAA with all their dependencies,
* a metadata exchange,
* two identity providers (with two accounts each, `user1:user1pass` and
  `user2:user2pass`),
* two service providers (for testing the release of attributes by HEXAA,
  SSP AA)
* a local SMTP server with web interface for testing invitations and
  enabling services inside HEXAA.

Your user needs to be a member of the `docker` group. The playbook sets
this up for you, but `sudo` rights are needed for this (see
`--ask-become-pass`). You can add `--skip-tags docker` to the ansible
command, and manually install docker (`docker.io`/`docker-ce` OS package
**and** the `docker`, `pyopenssl`  pip packages), add your user.

If your user was added to the `docker` group now, you need to re-login
or run ansible commands with:
`sg docker "<ansible command>"`

The used docker images take up about 4GB:
```
REPOSITORY                                       TAG       IMAGE ID       CREATED        SIZE
hexaaproject/hexaa-frontend                      staging   95a0e653d65d   2 days ago     535MB
hexaaproject/hexaa-backend                       staging   01150b3d92f9   2 days ago     605MB
mariadb                                          latest    c1c9e6fba07a   2 days ago     355MB
memcached                                        latest    829146221029   3 days ago     82.2MB
jwilder/nginx-proxy                              alpine    1aa571816748   7 days ago     54.5MB
nginx                                            latest    540a289bab6c   3 weeks ago    126MB
hexaaproject/ssp-aa                              latest    c5d739bc29b2   2 months ago   463MB
hexaaproject/hexaa-service-entityids-generator   latest    f8b3e7021d87   4 months ago   93.6MB
hexaaproject/apache-shib                         latest    683d18789660   4 months ago   199MB
szabogyula/attributes                            latest    43ab8f0e81d9   6 months ago   479MB
szabogyula/test-saml-idp                         latest    f366ec85803a   8 months ago   435MB
mailhog/mailhog                                  latest    e00a21e210f9   13 months ago  19.2MB
leifj/pyff                                       latest    e9ff2d9b8164   3 years ago    629MB
```

### Hostname

By default, the services are configured to run under `hexaa.local`.

You should add an entry like this to your `/etc/hosts` config:
```
127.0.0.1	hexaa.local metadata.hexaa.local idp1.hexaa.local idp2.hexaa.local sp1.hexaa.local sp2.hexaa.local mail.hexaa.local
```

### Running

```sh
ansible-playbook -i inventory local.yml
```

You can restart with a blank environment by removing the containers and
generated files (**Warning:** this deletes the database content and
modifications made inside the containers):

```bash
docker rm -f sp1.hexaa.local sp2.hexaa.local \
    idp1.hexaa.local idp2.hexaa.local \
    metadata.hexaa.local nginx-proxy \
    hexaa-backend hexaa-backend-ssp-aa hexaa-backend-web \
    hexaa-frontend hexaa-frontend-web \
    hexaa-service-entityids-generator \
    mail.hexaa.local memcached mysql
find local_files -type f \! -name '.git*' -exec rm {} \;
```

### Web access

If you didn't change the default config, after running the playbook, you
can access the following services:

* HEXAA frontend: https://hexaa.local
* Service providers:
    * https://sp1.hexaa.local
    * https://sp2.hexaa.local
* SSP IdP:
    * https://idp1.hexaa.local/simplesaml/
    * https://idp2.hexaa.local/simplesaml/
* SSP AA: https://hexaa.local:8443/simplesaml/
* Metadata exchange: https://metadata.hexaa.local/
* Local mail: https://mail.hexaa.local

### Possible errors

* If you have a firewall set up on your host, it can prevent
  communication between the containers. If you use ufw, you should allow
  this traffic: `ufw allow proto tcp from 172.16.0.0/12 to
  172.16.0.0/12`, given that all docker networks are subnets of the
  range.
* The installation of pyopenssl Python module might fail (on older
  Ubuntu systems), install the `python3-cryptography` or `libssl-dev`
  package to fix this.


## Production environment

You should:

 * Copy and customize `group_vars/prod_template`,
 * Copy `prod_template.yml` to a matching name,
 * Set your installation target in `inventory` in the `prod` section.
   `ansible_become=true` is useful if the remote login user has sudo
   rights.

If multiple people will run the playbook, we recommend using one user to
login on the target host or setting `ansible_become=true` to prevent
unnecessary ownership changes and errors.

If you choose the latter, the login user(s) needs to have `sudo` rights
without password (`NOPASSWD` in `/etc/sudoers`) or you need to provide
your password to Ansible: use the `--ask-become-pass` flag and enter it
when asked.

If you plan to version control your configuration, you should consider
putting the master keys and secrets into an Ansible vault (see the
`ansible-vault(1)` manpage and the `--vault-password-file` option of
`ansible-playbook`).

### Running

```sh
ansible-playbook -i inventory <env_name>.yml
```

### Backup

You only need to make backups of the contents of `/opt/hexaa/mysql`.


# Development

You can fork the playbook and role repositories, then clone the two
roles into the subdirectory, and make changes to the 3 repositories at
the same time:

```sh
git clone https://github.com/hexaaproject/ansible-hexaa.git
cd ansible-hexaa/roles
git clone https://github.com/hexaaproject/ansible-role-hexaa-frontend.git
git clone https://github.com/hexaaproject/ansible-role-hexaa-backend.git
git clone https://github.com/hexaaproject/ansible-role-hexaa-test-env.git
```

# Architecture

Components installed by the three HEXAA roles:

![HEXAA core components](https://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.github.com/hexaaproject/ansible-hexaa/master/doc/hexaa_core.puml)

Local environment components and their (somewhat simplified) interactions:

![HEXAA environment components](https://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.github.com/hexaaproject/ansible-hexaa/master/doc/hexaa_env.puml)
