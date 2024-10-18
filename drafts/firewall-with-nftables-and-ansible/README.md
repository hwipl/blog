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

#### Templates

TODO: add better example, individual template here or in config? add flush ruleset

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

### Playbook

nftables.yml:

```yaml
---
- name: Configure nftables
  hosts: nftables_hosts

  roles:
    - nftables
```

### Configuration

```ini
[nftables_hosts]
node1
```

```yaml
---
# nftables configuration
nftables_conf: "{{ inventory_dir }}/nftables/node1-nftables.conf.j2"
```

```yaml
---
# # nftables configuration
# nftables_conf: "nftables.conf.j2"
```

### Deployment

```console
$ # site 1
$ ansible-playbook -i site1/hosts nftables.yml
$ # site 2
$ ansible-playbook -i site2/hosts nftables.yml
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
