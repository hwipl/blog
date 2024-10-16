# Firewall with nftables and Ansible

This document describes how you can deploy Linux firewalls with nftables and
Ansible. It shows the nftables configuration as well as the Ansible role,
playbook and configuration that are used to install and configure each firewall
server automatically.

## Overview

## Firewall Configuration

## Ansible

### Role

```
roles/nftables/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── nftables.conf.j2
```

#### Handlers

```yaml
---
# handlers for nftables

- name: Restart nftables
  become: true
  ansible.builtin.service:
    name: nftables
    state: restarted
```

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

TODO: add better example, individual template here or in config?

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
head
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
