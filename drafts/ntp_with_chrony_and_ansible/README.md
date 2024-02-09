# NTP with Chrony and Ansible

This document describes how you can deploy chrony NTP servers with Ansible. It
shows the NTP configuration of chrony as well as the Ansible role, playbook and
configuration that are used to install and configure each NTP server
automatically.

## Overview

## NTP Configuration

```
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst

# Allow access to this ntp server from the following hosts/networks
allow 10.20.1.0/24
allow 10.20.2.0/24
```

## Ansible

### Role

The Ansible role is called `chronyd` and structured as shown in the listing
below:

```
roles/chronyd/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── chrony.conf.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and one template file. The tasks use the template `chrony.conf.j2` to
create the NTP configuration.

#### Handlers

A handler is defined as follows in the file `roles/chronyd/handlers/main.yml`:

```yaml
---
# handlers for chronyd

- name: Restart chrony
  become: true
  ansible.builtin.service:
    name: chrony
    state: restarted
```

#### Tasks

roles/chronyd/tasks/main.yml:

```yaml
---
# these tasks install and configure chrony ntp servers

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure chrony is installed
  become: true
  ansible.builtin.apt:
    name: chrony
    state: present
- name: Create chrony configuration from template
  become: true
  ansible.builtin.template:
    src: chrony.conf.j2
    dest: "/etc/chrony/chrony.conf"
    owner: root
    group: root
    mode: '0644'
  notify:
    - Restart chrony
```

#### Templates

roles/chronyd/templates/chrony.conf.j2:

```jinja
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst

# Allow access to this ntp server from the following hosts/networks
{% for allow in allows %}
allow {{ allow }}
{% endfor %}
```

### Playbook

chronyd.yml:

```yaml
---
- name: Configure ntp servers
  hosts: ntp_servers

  roles:
    - chronyd
```
### Configuration

#### Hosts

hosts:

```ini
[ntp_servers]
node1
```

(no host-specific vars)

#### Groups

site1/group_vars/ntp_servers:

```yaml
---
# ntp server configuration

allows:
  - 10.20.1.0/24
  - 10.20.2.0/24
```

site2/group_vars/ntp_servers:

```yaml
---
# ntp server configuration

allows:
  - 10.20.1.0/24
  - 10.20.2.0/24
```

### Deployment

```console
$ # site 1
$ ansible-playbook -i site1/hosts chronyd.yml
$ # site 2
$ ansible-playbook -i site2/hosts chronyd.yml
```

## Conclusion

## Appendix: Code
