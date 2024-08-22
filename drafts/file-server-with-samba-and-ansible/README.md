# File Server with Samba and Ansible

This document describes how you can deploy Samba file servers with Ansible. It
shows the file server configuration as well as the Ansible role, playbook and
configuration that are used to install and configure each Samba server
automatically.

## Overview

## File Server Configuration

The configuration of the files servers is described in this section. The
servers in the two sites are configured as follows in the file
`/etc/samba/smb.conf`:

```ini
[global]
        map to guest = Bad User
        log file = /var/log/samba/log.%m
        log level = 1
        server role = standalone server

[guest]
        # This share allows anonymous (guest) access
        # without authentication!
        path = /srv/samba/guest
        read only = yes
        guest ok = yes
        guest only = yes
```

The section `[global]` specifies the global settings of the Samba server. The
option `map to guest` is set to `Bad User` and maps not existing users to the
guest account. The option `log file` sets the log file to
`/var/log/samba/log.%m` where `%m` is replaced with the name of client
machines. The option `log level` sets the logging level to `1` which is the
lowest level. The option `server role` sets the server to `standalone server`
and requires clients to log on with a valid username and password.

The section `[guest]` specifies the settings of a share called `guest`.  The
option `path` sets the path of the share in the server's file system to
`/srv/samba/guest`. The option `read only` forbids write access to the share.
The option `guest ok` allows guest connections to the share with no password.
The option `guest only` allows only guest connections to the share.

TODO: remove sites?

## Ansible

Ansible allows for automatic installation and configuration of the VPN servers.
Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist of tasks,
templates and handlers. Tasks are the individual installation and configuration
steps. They use the [templates][templates] to generate configuration files and
trigger events that are handled by the [handlers][handlers].

A role and a playbook are used to deploy the file servers. The playbook assigns
the role to all nodes of a group defined in the Ansible [inventory][inventory].
The configuration of each file server is derived from variables in the
inventory. Each site uses a different inventory to allow for site-specific
configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

TODO: remove sites?

### Role

The Ansible role is called `samba` and structured as shown in the listing
below:

```
roles/samba/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── smb.conf.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and one template file. The tasks use the template `smb.conf.j2` to create
the configuration file for the file servers.

#### Handlers

A handler is defined as follows in the file `roles/samba/handlers/main.yml`:

```yaml
---
# handlers for samba

- name: Restart samba
  become: true
  ansible.builtin.service:
    name: "{{ item }}"
    state: restarted
  loop:
    - smbd
    - nmbd
```

The handler is called `Restart samba` and restarts the file server with the
[service module][service] when it is triggered. It requires root privileges to
manipulate the state of the system services, so [become][become] is set to
`true` for [privilege escalation][privilege]. To restart file server, it sets
the system services `smbd` and `nmbd` to state `restarted`.

#### Tasks

The tasks are defined as follows in the file `roles/samba/tasks/main.yml`:

```yaml
---
# these tasks setup samba with a simple guest share

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure samba is installed
  become: true
  ansible.builtin.apt:
    name: samba
    state: present
- name: Ensure share directory exists
  become: true
  ansible.builtin.file:
    path: "{{ samba_path }}"
    state: directory
    owner: root
    group: root
    mode: '0775'
- name: Create samba configuration from template
  become: true
  ansible.builtin.template:
    src: smb.conf.j2
    dest: "/etc/samba/smb.conf"
    owner: root
    group: root
    mode: '0644'
    backup: true
  notify:
    - Restart samba
```

These tasks install the file server with the [apt module][apt], configure it
with the template and the [template module][template] and trigger the restart
event with [notify][notify] if the configuration file is changed. All tasks
need root privileges to manipulate the system configuration. So,
[become][become] is set to `true`. The file owner and group of all files are
set to `root`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs Samba with `apt` if it is not already installed.
3. The third task creates the directory for the file share if it does not
   exist. The path name is taken from the Ansible variable `samba_path`. File
   permissions are set to `775`.
4. The fourth task creates or updates the Samba configuration in
   `/etc/samba/smb.conf` from the template `smb.conf.j2`. File permissions are
   set to `644`. An existing configuration file is backed up in the process. If
   the configuration changed, the task triggers the restart event.

#### Templates

The template for the Samba configuration file is defined as [Jinja2
template][jinja2]. It is defined as follows in the file
`roles/samba/templates/smb.conf.j2`:

```jinja
[global]
        map to guest = Bad User
        log file = /var/log/samba/log.%m
        log level = 1
        server role = standalone server

[{{ samba_share }}]
        # This share allows anonymous (guest) access
        # without authentication!
        path = {{ samba_path }}
        read only = yes
        guest ok = yes
        guest only = yes
```

The template reflects the server configuration shown in the File Server
Configuration section above with parts dynamically generated based on the
Ansible configuration:

- The name of the file share is read from the variable `samba_share`
- The path of the file share is read from the variable `samba_path`

### Playbook

The playbook is defined as follows in the file `samba.yml`:

```yaml
---
- name: Configure smb servers
  hosts: smb_servers

  roles:
    - samba
```

The playbook assigns the role `samba` that is described above to all hosts in
the group `smb_servers`. On execution, this playbook runs all the tasks of the
role on all the hosts in the group to install and configure the file servers.

### Configuration

The configuration is derived from host files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use. Each site uses a different inventory and, thus, different host files to
allow for site-specific configurations as shown below. There is no other
group-specific configuration of the Samba servers in the `group_vars` in each
site.

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[smb_servers]
node1
```

The hosts file of each site defines the group `smb_servers` and assigns
the node `node1` to it. Thus, `node1` is defined as Samba server for the
playbook.

The host-specific configuration of the Samba servers is in the `host_vars` of
`node1` in each site. The configuration for the file server in Site 1 and Site
2 is defined as follows in the files `site1/host_vars/node1` and
`site2/host_vars/node1`:

```yaml
# smb server configuration
samba_path: "/srv/samba/guest"
samba_share: "guest"
```

Both files set the configuration of Samba in Site 1 and Site 2 as described
in the File Server Configuration section above.

The path of the Samba file share is set to `/srv/samba/guest` in the
`samba_path` variable. The name of the Samba share is set to `guest` in the
variable `samba_share`.

TODO: remove sites?

### Deployment

The file servers can be installed and configured with the Ansible role,
configuration and playbook described above. You can use the following
[ansible-playbook][ansible-playbook] commands:

```console
$ # site 1
$ ansible-playbook -i site1/hosts samba.yml
$ # site 2
$ ansible-playbook -i site2/hosts samba.yml
```

Both `ansible-playbook` commands run the playbook `samba.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
first command installs and configures all Samba servers in Site 1 and the
second command in Site 2. After successful execution of the commands above, the
file servers should be configured and running.

TODO: remove sites?

## Conclusion

This document describes how you can automatically install and configure Samba
file servers on Ubuntu 22.04 LTS with Ansible. After presenting an example
network and file server configuration, an Ansible Role is shown that contains
the installation and configuration tasks as well as templates. The tasks use
the templates together with Ansible variables to create the Samba
configuration. Finally, an Ansible Playbook is used with the `ansible-playbook`
command to run the tasks and, thus, install and configure all file servers.
All sections contain configuration or code examples you can use as a basis for
your own setup. Also, you can find links to the code and configuration examples
in the appendix below.

TODO: remove/change example network part?

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/samba/roles/samba)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/samba/samba.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/samba/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/samba/examples/site2)

TODO: remove sites?

[roles]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
[playbooks]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html
[templates]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html
[handlers]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html
[inventory]: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
[ansible-playbook]: https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html
[service]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html
[become]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html#become-directives
[privilege]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html
[apt]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html
[template]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html
[notify]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html#notifying-handlers
[jinja2]: https://jinja.palletsprojects.com/en/latest/templates/
