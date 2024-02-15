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

Ansible allows for automatic installation and configuration of the NTP
servers. Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist
of tasks, templates and handlers. Tasks are the individual installation and
configuration steps. They use the [templates][templates] to generate
configuration files and trigger events that are handled by the
[handlers][handlers].

A role and a playbook are used to deploy the NTP servers. The playbook assigns
the role to all nodes of a group defined in the Ansible [inventory][inventory].
The configuration of each NTP server is derived from host and group variables
in the inventory. Each site uses a different inventory to allow for
site-specific configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

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

The handler is called `Restart chrony` and restarts the NTP server with the
[service module][service] when it is triggered. It requires root privileges to
manipulate the state of the system services, so [become][become] is set to
`true` for [privilege escalation][privilege]. To restart the NTP server, it
sets the system service `chrony` to state `restarted`.

#### Tasks

The tasks are defined as follows in the file `roles/chronyd/tasks/main.yml`:

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

These tasks install the chrony NTP server with the [apt module][apt], configure
it with the template and the [template module][template] and trigger the
restart event with [notify][notify] if the configuration is actually changed:

All tasks need root privileges to manipulate the system configuration. So,
[become][become] is set to `true`. The file owner of the configuration file is
set to `root`, group is set to `root` and privileges are set to `644`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs the chrony NTP server with `apt` if it is not
   already installed.
3. The third task creates or updates the configuration of the NTP server in
   file `/etc/chrony/chrony.conf` from the template `chrony.conf.j2`. If the
   file is changed, the restart event is triggered.

#### Templates

The template for the file `/etc/chrony/chrony.conf` is a [Jinja2
template][jinja2]. It is defined as follows in the file
`roles/chronyd/templates/chrony.conf.j2`:

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

The template reflects the chrony configuration shown in the NTP configuration
section above with parts dynamically generated based on the Ansible
configuration: The allowed IP address ranges are taken from the Ansible list
variable `allows`.

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
