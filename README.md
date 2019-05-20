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
    - [Production environment](#production-environment)
        - [Running](#running-1)
        - [Backup](#backup)
- [Development](#development)

# Usage

Requirements:
- a recent version of Docker (`docker.io` or `docker-ce` package)
- Ansible (`pip3 install ansible\>=2.7`)

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

Your user needs to be a member of the `docker` group

### Hostname

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
    hexaa-service-entityids-generator
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


## Production environment

Your should:

 * Copy and customize `group_vars/prod_template`,
 * Copy `prod_template.yml` to a matching name,
 * Set your installation target in `inventory` in the `prod` section.
   `ansible_become=true` is useful if the remote login user has sudo
   rights.

If multiple people will run the playbook, we recommend using one user to
login on the target host or setting `ansible_become=true` to prevent
unnecessary ownership changes and errors.

If you choose to latter, the login user(s) needs to have `sudo` rights
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
```

# Architecture

Components installed by two main roles:

![HEXAA core components](http://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.github.com/hexaaproject/ansible-hexaa/master/doc/hexaa_core.puml)

Local environment components and their (somewhat simplified) interactions:

![HEXAA environment components](http://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.github.com/hexaaproject/ansible-hexaa/master/doc/hexaa_env.puml)
