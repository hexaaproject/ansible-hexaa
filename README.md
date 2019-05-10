# ansible-hexaa

Ansible playbook for installing a complete HEXAA environment.


## Usage

Requirements:
- a recent version of Docker (`docker.io` or `docker-ce` package)
- Ansible (`pip3 install ansible\>=2.7`)

Installing the roles:

```sh
ansible-galaxy install -r requirements.yml -p ./roles
```


### Local environment

Your user needs to:

* be a member of the `docker` group
* be able to use `sudo` without password (`NOPASSWD` in `/etc/sudoers`)
  or you need to provide your password to Ansible:
  use the `--ask-become-pass` flag and enter it when asked.

**Hostname:**

You should add an entry like this to your `/etc/hosts` config:
```
127.0.0.1	hexaa.local metadata.hexaa.local idp1.hexaa.local idp2.hexaa.local sp1.hexaa.local sp2.hexaa.local mail.hexaa.local
```

**Running:**

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


### Production environment

Your should:

 * Copy and customize `group_vars/prod_template`,
 * Copy `prod_template.yml` to a matching name,
 * Set your installation target in `inventory` in the `prod` section.
   `ansible_become=true` is useful if the remote login user has sudo
   rights.

If you plan to version control your configuration, you should consider
putting master keys and secrets into an Ansible vault (see the
`ansible-vault(1)` manpage and the `--vault-password-file` option of
`ansible-playbook`).

**Running:**

```sh
ansible-playbook -i inventory <env_name>.yml
```

**Backup:**

You only need to make backups of the contents of `/opt/hexaa/mysql`.


## Development

You can fork the playbook and role repositories, then clone the two
roles into the subdirectory, and make changes to the 3 repositories at
the same time:

```sh
git clone https://github.com/hexaaproject/ansible-hexaa.git
cd ansible-hexaa/roles
git clone https://github.com/hexaaproject/ansible-role-hexaa-frontend.git
git clone https://github.com/hexaaproject/ansible-role-hexaa-backend.git
```

```plantuml
@startuml
package "Hexaa CORE" {
    package "backend" {
        [hexaa-backend-web]
        note top: Nginx
        [hexaa-backend]
        note top: PHP-fpm
        [hexaa-service-entityids-generator]
        [mysql]
        [memcached]
        [hexaa-ssp-aa]
        note top: SimpleSAMLphp
    }
    note top of backend: ansible-role-backend
    package "frontend" {
        [hexaa-frontend]
        note top: PHP-fpm
        [hexaa-frontend-web]
        note top: Apache HTTPD + Shibboleth SP
    }
    note top of frontend: ansible-role-frontend
}

@enduml
```
