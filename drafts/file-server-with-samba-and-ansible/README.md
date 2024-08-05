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

### Role

```
roles/samba/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── smb.conf.j2
```

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
