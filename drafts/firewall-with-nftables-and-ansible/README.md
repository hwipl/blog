# Firewall with nftables and Ansible

This document describes how you can deploy Linux firewalls with nftables and
Ansible. It shows the nftables configuration as well as the Ansible role,
playbook and configuration that are used to install and configure each firewall
server automatically.

## Overview

## Firewall Configuration

## Ansible

Ansible allows for automatic installation and configuration of the firewalls.
Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist of tasks,
templates and handlers. Tasks are the individual installation and configuration
steps. They use the [templates][templates] to generate configuration files and
trigger events that are handled by the [handlers][handlers].

A role and a playbook are used to deploy the firewalls. The playbook assigns
the role to all nodes of a group defined in the Ansible [inventory][inventory].
The configuration of each firewall is derived from variables in the inventory.
Each site uses a different inventory to allow for site-specific configurations.
The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

### Role

The Ansible role is called `nftables` and structured as shown in the listing
below:

```
roles/nftables/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── nftables.conf.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and one template file. The tasks can use the template `nftables.conf.j2`
to create the configuration file for the firewalls.

TODO: other templates possible in inventory

#### Handlers

A handler is defined as follows in the file `roles/nftables/handlers/main.yml`:

```yaml
---
# handlers for nftables

- name: Restart nftables
  become: true
  ansible.builtin.service:
    name: nftables
    state: restarted
```

The handler is called `Restart nftables`. It restarts the firewall with the
[service module][service] when it is triggered. It requires root privileges to
manipulate the state of the system services, so [become][become] is set to
`true` for [privilege escalation][privilege]. To restart the firewall, it sets
the system service `nftables` to state `restarted`. The nftables service then
sets the firewall rules.

#### Tasks

The tasks are defined as follows in the file `roles/nftables/tasks/main.yml`:

TODO: update apt cache, ensure nftables is installed?

```yaml
---
# these tasks setup nftables

- name: Ensure nftables service is enabled
  become: true
  ansible.builtin.service:
    name: nftables
    enabled: true
- name: Create nftables configuration from template
  become: true
  ansible.builtin.template:
    src: "{{ nftables_conf }}"
    dest: "/etc/nftables.conf"
    owner: root
    group: root
    mode: '0644'
    backup: true
  notify:
    - Restart nftables
```

These tasks install nftables with the [apt module][apt], enable the nftables
service with the [service module][service], configure the firewall with a
template and the [template module][template] and trigger the restart event with
[notify][notify] if the configuration file is changed. All tasks need root
privileges to manipulate the system configuration. So, [become][become] is set
to `true`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs nftables with `apt` if it is not already installed.
3. The third task enables the nftables service if it is not enabled.
4. The fourth task creates or updates the firewall configuration in
   `/etc/nftables.conf` from the template in the ansible variable
   `nftables_conf`. File owner and group are set to root. File permissions are
   set to `644`. An existing configuration file is backed up in the process. If
   the configuration changed, the task triggers the restart event.

#### Templates

The template for the firewall configuration file is defined as [Jinja2
template][jinja2]. It is defined as follows in the file
`roles/nftables/templates/nftables.conf.j2`:

TODO: add better example, individual template here or in config? add flush
ruleset, or maybe do not and come up with a better version that does not
interfere with other firewall rules by, e.g., docker and libvirt. docker and
libvirt should be started later and set the rules themselves. But restarting at
a later point in time is a problem.

TODO: add note that you need to be careful to not lock you out of the host or
mess up other services.

```jinja
#!/usr/bin/nft -f
# vim:set ts=2 sw=2 et:

# IPv4/IPv6 Simple & Safe firewall ruleset.
# More examples in /usr/share/nftables/ and /usr/share/doc/nftables/examples/.

table inet filter
delete table inet filter
table inet filter {
  chain input {
    type filter hook input priority filter
    policy drop

    ct state invalid drop comment "early drop of invalid connections"
    ct state {established, related} accept comment "allow tracked connections"
    iifname lo accept comment "allow from loopback"
    ip protocol icmp accept comment "allow icmp"
    meta l4proto ipv6-icmp accept comment "allow icmp v6"
    tcp dport ssh accept comment "allow sshd"
    pkttype host limit rate 5/second counter reject with icmpx type admin-prohibited
    counter
  }
  chain forward {
    type filter hook forward priority filter
    policy drop
  }
}
```

The template reflects the configuration shown in the Firewall Configuration
section above.

### Playbook

The playbook is defined as follows in the file `nftables.yml`:

```yaml
---
- name: Configure nftables
  hosts: nftables_hosts

  roles:
    - nftables
```

The playbook assigns the role `nftables` that is described above to all hosts
in the group `nftables_hosts`. On execution, this playbook runs all the tasks
of the role on all the hosts in the group to configure the firewalls.

### Configuration

The configuration is derived from host files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use. Each site uses a different inventory and, thus, different host files to
allow for site-specific configurations as shown below. There is no other
group-specific configuration of the firewalls in the `group_vars` in each site.

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[nftables_hosts]
node1
```

TODO: add more nodes?

The hosts file of each site defines the group `nftables_hosts` and assigns
the node `node1` to it. Thus, `node1` is defined as firewall host for the
playbook.

TODO: terminology: use host or node or server or reword to firewall (config)?

The host-specific configuration of the firewall is in the `host_vars` of
`node1` in each site. The configuration for Site 1 and Site 2 is defined as
follows in the files `site1/host_vars/node1` and `site2/host_vars/node1`:

```yaml
---
# nftables configuration
nftables_conf: "{{ inventory_dir }}/nftables/node1-nftables.conf.j2"
```

Both files set the firewall configuration for Site 1 and Site 2 as described in
the Firewall Configuration section above. The respective nftables configuration
file is set to the node-specific template file `node1-nftables.conf.j2` in the
subdirectory `nftables` of the Ansible inventory (`{{ inventory_dir }}`): In
Site 1 the resulting template file is `site1/nftables/node1-nftables.conf.j2`
and in Site 2 it is `site2/nftables/node1-nftables.conf.j2`.

TODO: add other node with "default" config? or add a group with the "default"
config?

```yaml
---
# # nftables configuration
# nftables_conf: "nftables.conf.j2"
```

### Deployment

The firewalls can be set up with the Ansible role, configuration and playbook
described above. You can use the following [ansible-playbook][ansible-playbook]
commands:

```console
$ # site 1
$ ansible-playbook -i site1/hosts nftables.yml
$ # site 2
$ ansible-playbook -i site2/hosts nftables.yml
```

Both `ansible-playbook` commands run the playbook `nftables.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
first command configures all firewalls in Site 1 and the second command in
Site 2. After successful execution of the commands above, the firewalls should
be configured and running.

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
