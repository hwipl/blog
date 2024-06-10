# VPN with WireGuard and Ansible

This document describes how you can deploy WireGuard VPN servers with Ansible.
It shows the WireGuard configuration as well as the Ansible role, playbook and
configuration that are used to install and configure each VPN server
automatically.

## Overview

## VPN Configuration

## Ansible

### Role

```
roles/wireguard/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── wg-client.conf.j2
    └── wg.conf.j2
```

#### Handlers

```yaml
---
# handlers for wireguard

- name: Restart wireguard connection
  become: true
  ansible.builtin.service:
    name: "wg-quick@{{ wireguard_interface }}"
    state: restarted
```

#### Tasks

```yaml
---
# these tasks install and configure wireguard servers

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure wireguard is installed
  become: true
  ansible.builtin.apt:
    name: wireguard
    state: present
- name: Copy wireguard server key file
  become: true
  ansible.builtin.copy:
    src: "{{ wireguard_private_key_file }}"
    dest: "/etc/wireguard/{{ wireguard_interface }}.key"
    owner: root
    group: root
    mode: '0600'
  notify: Restart wireguard connection
- name: Create wireguard server config file
  become: true
  ansible.builtin.template:
    src: wg.conf.j2
    dest: "/etc/wireguard/{{ wireguard_interface }}.conf"
    owner: root
    group: root
    mode: '0600'
  notify: Restart wireguard connection
- name: Enable wireguard connection
  become: true
  ansible.builtin.service:
    name: "wg-quick@{{ wireguard_interface }}"
    enabled: true
  notify: Restart wireguard connection
- name: Create local directory for wireguard client configs
  ansible.builtin.file:
    path: "{{ inventory_dir }}/wireguard-clients"
    mode: '0750'
    state: directory
  delegate_to: localhost
  connection: local
  become: false
  run_once: true
- name: Create local wireguard client configs
  vars:
    wireguard_public_key: "{{ lookup('file', wireguard_public_key_file) }}"
  ansible.builtin.template:
    src: wg-client.conf.j2
    dest: "{{ inventory_dir}}/wireguard-clients/{{ item.name }}.conf"
    mode: '0600'
  loop: "{{ wireguard_peers }}"
  delegate_to: localhost
  connection: local
  become: false
  run_once: true
```

#### Templates


wg-client.conf.j2:

```jinja
[Interface]
PrivateKey = INSERT_YOUR_PRIVATE_KEY_HERE
ListenPort = {{ wireguard_listen_port }}
Address = {{ item.client_address }}

[Peer]
PublicKey = {{ wireguard_public_key }}
Endpoint = {{ item.client_endpoint }}
AllowedIPs = {{ item.client_allowed_ips }}
```

wg.conf.j2

```jinja
[Interface]
ListenPort = {{ wireguard_listen_port }}
Address = {{ wireguard_address }}
PostUp = wg set %i private-key /etc/wireguard/{{ wireguard_interface }}.key
{% for peer in wireguard_peers %}

[Peer]
PublicKey = {{ peer.public_key }}
AllowedIPs = {{ peer.allowed_ips }}
{% endfor %}
```

### Playbook

wireguard.yml:

```yaml
---
- name: Create and start wireguard servers
  hosts: wireguard_servers

  roles:
    - wireguard
```

### Configuration

`site1/hosts` and `site2/hosts`:

```ini
[wireguard_servers]
head
```

`site1/host_vars/node1`:

```yaml
# wireguard configuration
# Server IP 10.20.21.1
wireguard_interface: "wg0"
wireguard_listen_port: 51000
wireguard_address: 10.20.21.1/24
wireguard_private_key_file: "~/cluster/wireguard/server/server-private.key"
wireguard_public_key_file: "~/cluster/wireguard/server/server-public.key"
wireguard_peers:
  # Client IP 10.20.21.2:
  - name: peer1  # staff, admin key
    # server side
    public_key: INSERT_PEER1_REAL_KEY_HERE
    allowed_ips: 10.20.21.2
    # client side
    client_address: 10.20.21.2/24
    client_endpoint: site1.cluster.company.lan:51000
    client_allowed_ips: "10.20.0.0/16, 10.21.0.0/16, 10.22.0.0/16, 10.23.0.0/16"
  # Client IP 10.20.21.3:
  - name: peer2  # staff, other user key
    # server side
    public_key: INSERT_PEER2_REAL_KEY_HERE
    allowed_ips: 10.20.21.3
    # client side
    client_address: 10.20.21.3/24
    client_endpoint: site1.cluster.company.lan:51000
    client_allowed_ips: "10.20.0.0/16, 10.21.0.0/16, 10.22.0.0/16, 10.23.0.0/16"
```

`site2/host_vars/node1`:

```yaml
# wireguard configuration
# Server IP 10.20.22.1
wireguard_interface: "wg0"
wireguard_listen_port: 51000
wireguard_address: 10.20.22.1/24
wireguard_private_key_file: "~/cluster/wireguard/server/server-private.key"
wireguard_public_key_file: "~/cluster/wireguard/server/server-public.key"
wireguard_peers:
  # Client IP 10.20.22.2:
  - name: peer1  # staff, admin key
    # server side
    public_key: INSERT_PEER1_REAL_KEY_HERE
    allowed_ips: 10.20.22.2
    # client side
    client_address: 10.20.22.2/24
    client_endpoint: site2.cluster.company.lan:51000
    client_allowed_ips: "10.20.0.0/16, 10.21.0.0/16, 10.22.0.0/16, 10.23.0.0/16"
  # Client IP 10.20.22.3:
  - name: peer2  # staff, other user key
    # server side
    public_key: INSERT_PEER2_REAL_KEY_HERE
    allowed_ips: 10.20.22.3
    # client side
    client_address: 10.20.22.3/24
    client_endpoint: site2.cluster.company.lan:51000
    client_allowed_ips: "10.20.0.0/16, 10.21.0.0/16, 10.22.0.0/16, 10.23.0.0/16"
```

No group_vars.

### Deployment

```console
$ # site 1
$ ansible-playbook -i site1/hosts wireguard.yml
$ # site 2
$ ansible-playbook -i site2/hosts wireguard.yml
```

## Conclusion

## Appendix: Code