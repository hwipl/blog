# File Server with Samba and Ansible

This document describes how you can deploy Samba file servers with Ansible. It
shows the file server configuration as well as the Ansible role, playbook and
configuration that are used to install and configure each Samba server
automatically.

## Overview

## File Server Configuration

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

`roles/samba/handlers/main.yml`:

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

#### Tasks

`roles/samba/tasks/main.yml`:

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

#### Templates

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

### Playbook

`samba.yml`:

```yaml
---
- name: Configure smb servers
  hosts: smb_servers

  roles:
    - samba
```

### Configuration

#### Hosts

`hosts`:

```ini
[smb_servers]
node1
```

`host_vars/node1`:

```yaml
# smb server configuration
samba_path: "/srv/samba/guest"
samba_share: "guest"
```

#### Groups

### Deployment

## Conclusion

## Appendix: Code

[roles]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
[playbooks]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html
[templates]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html
[handlers]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html
[inventory]: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
[ansible-playbook]: https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html
