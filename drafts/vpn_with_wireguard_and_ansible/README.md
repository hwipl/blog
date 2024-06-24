# VPN with WireGuard and Ansible

This document describes how you can deploy WireGuard VPN servers with Ansible.
It shows the WireGuard configuration as well as the Ansible role, playbook and
configuration that are used to install and configure each VPN server
automatically.

## Overview

```
           +--------------+ +--------------+ +--------------+
           | VPN Client 1 | | VPN Client 2 | | VPN Client 3 |
           | 10.20.21.2   | | 10.20.21.3   | | 10.20.21.4   |
           | 10.20.22.2   | | 10.20.22.3   | | 10.20.22.4   |
           +--------------+ +--------------+ +--------------+
                   _|_______________|_______________|_
                  |                                   |
..................|...................................|..................
: Site 1          |                 :                 |          Site 2 :
: 10.20.1.0/24    |                 :                 |    10.20.2.0/24 :
:                 |                 :                 |                 :
:          +-------------+          :          +-------------+          :
:          | Node 1      |          :          | Node 1      |          :
:          | VPN Server  |          :          | VPN Server  |          :
:          | vpn.s1.lan  |          :          | vpn.s2.lan  |          :
:          | 10.20.21.1  |          :          | 10.20.22.1  |          :
:          | 10.20.1.1   |          :          | 10.20.2.1   |          :
:          +-------------+          :          +-------------+          :
:          _______|_______          :          _______|_______          :
:         |               |         :         |               |         :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:  | Node 2      | | Node 3      |  :  | Node 2      | | Node 3      |  :
:  | 10.20.1.2   | | 10.20.1.3   |  :  | 10.20.2.2   | | 10.20.2.3   |  :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:...................................:...................................:
                        Network: 10.20.0.0/16
```

## VPN Configuration

TODO: add key generation somewhere?

TODO: use "real" keys instead of placeholders?

Key generation on client and server:

```console
$ umask 077
$ wg genkey | tee privatekey | wg pubkey > publickey
```

### Server

Site 1 VPN Server `/etc/wireguard/wg0.conf`:

```jinja
[Interface]
ListenPort = 51000
Address = 10.20.21.1/24
PostUp = wg set %i private-key /etc/wireguard/wg0.key

[Peer]
# public key of client 1
PublicKey = 55BbgL/uvNBfaXko8UodGy93DnaQfiWDWaDCuPhh1z4=
AllowedIPs = 10.20.21.2

[Peer]
# public key of client 2
PublicKey = OinFs2iWV0/X3jDmsjnhAJLZP/127E8Ly13URQQB7BM=
AllowedIPs = 10.20.21.3

[Peer]
# public key of client 3
PublicKey = KqvsNv2Nppt37z3BGIFOZIwfA3QcnNsEqRfTCcNiERY=
AllowedIPs = 10.20.21.4
```

Site 2 VPN Server `/etc/wireguard/wg0.conf`:

```jinja
[Interface]
ListenPort = 51000
Address = 10.20.22.1/24
PostUp = wg set %i private-key /etc/wireguard/wg0.key

[Peer]
# public key of client 1
PublicKey = xImnlRiMQnY1vuOiVFx5W+8YDXYYbPMy3ROMr2HorBw=
AllowedIPs = 10.20.22.2

[Peer]
# public key of client 2
PublicKey = vPOB/d3CDETUuBI0OY/WbK5cjOnTGxYoY8yppwnpR3I=
AllowedIPs = 10.20.22.3

[Peer]
# public key of client 3
PublicKey = UY7GDMxHelkkMojD56kvuUJE+lL2dR38f4V8uCGfGg8=
AllowedIPs = 10.20.22.4
```

### Clients

Site 1 Client config in `/etc/wireguard/wg0.conf`:

```jinja
[Interface]
ListenPort = 51000
Address = 10.20.21.2/24     # Client 1
# Address = 10.20.21.3/24   # Client 2
# Address = 10.20.21.4/24   # Client 3
PostUp = wg set %i private-key /etc/wireguard/wg0.key

[Peer]
# public key of server
PublicKey =dRYtwEYc/3MUdvWEeTNYtOzxAnbhTRv6yllFL3F9fQ8=
Endpoint = vpn.s1.lan:51000
AllowedIPs = "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
```

Site 2 Client config in `/etc/wireguard/wg0.conf`:

```jinja
[Interface]
ListenPort = 51000
Address = 10.20.22.2/24     # Client 1
# Address = 10.20.22.2/24   # Client 2
# Address = 10.20.22.2/24   # Client 3
PostUp = wg set %i private-key /etc/wireguard/wg0.key

[Peer]
# public key of server
PublicKey = /nkmz7U8ryzGqnrNll9sVtzEg04N3ZhXzS2na+uc4Q4=
Endpoint = vpn.s2.lan:51000
AllowedIPs = "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
```

## Ansible

Ansible allows for automatic installation and configuration of the VPN servers.
Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist of tasks,
templates and handlers. Tasks are the individual installation and configuration
steps. They use the [templates][templates] to generate configuration files and
trigger events that are handled by the [handlers][handlers].

A role and a playbook are used to deploy the VPN servers. The playbook assigns
the role to all nodes of a group defined in the Ansible [inventory][inventory].
The configuration of each VPN server is derived from variables in the
inventory. Each site uses a different inventory to allow for site-specific
configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

### Role

The Ansible role is called `wireguard` and structured as shown in the listing
below:

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

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and two template files. The tasks use the templates `wg.conf.j2` and
`wg-client.conf.j2` to create the configuration files for the VPN servers and
clients.

#### Handlers

A handler is defined as follows in the file
`roles/wireguard/handlers/main.yml`:

```yaml
---
# handlers for wireguard

- name: Restart wireguard
  become: true
  ansible.builtin.service:
    name: "wg-quick@{{ wireguard_interface }}"
    state: restarted
```

The handler is called `Restart wireguard` and restarts the VPN server on the
VPN network interface with the [service module][service] when it is triggered.
It requires root privileges to manipulate the state of the system services, so
[become][become] is set to `true` for [privilege escalation][privilege]. To
restart the VPN server, it sets the system service
`wg-quick@{{wireguard_interface }}` to state `restarted`. The variable part
`{{wireguard_interface }}` is replaced with the name of the VPN network
interface, e.g., `wg0`.

#### Tasks

The tasks are defined as follows in the file `roles/wireguard/tasks/main.yml`:

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
  notify: Restart wireguard
- name: Create wireguard server config file
  become: true
  ansible.builtin.template:
    src: wg.conf.j2
    dest: "/etc/wireguard/{{ wireguard_interface }}.conf"
    owner: root
    group: root
    mode: '0600'
  notify: Restart wireguard
- name: Enable wireguard connection
  become: true
  ansible.builtin.service:
    name: "wg-quick@{{ wireguard_interface }}"
    enabled: true
  notify: Restart wireguard
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

These tasks install the VPN server with the [apt module][apt], configure it
with the template and the [template module][template] and trigger the restart
event with [notify][notify] if the configuration file is changed. Additionally,
the last two tasks create configuration files that can be used by the VPN
clients:

The installation and configuration tasks need root privileges to manipulate the
system configuration. So, [become][become] is set to `true`. The file owner and
group of all files are set to `root` and permissions are set to `600`. If the
configuration changed, the tasks trigger the restart event.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs wireguard with `apt` if it is not already installed.
3. The third task copies the private key of the wireguard server from the
   source file defined in the Ansible variable `wireguard_private_key_file` to
   its destination. The destination file name is the name of the wireguard
   network interface with the suffix `.key` in the wireguard configuration
   directory `/etc/wireguard`.
4. The fourth task creates or updates the wireguard configuration from the
   template `wg.conf.j2`. The name of the configuration file is the name of
   the wireguard network interface with the suffix `.conf` in the wireguard
   configuration directory `/etc/wireguard`.
5. The fifth task enables the wireguard VPN server for the network interface.
   It sets the corresponding systemd service to `enabled`.

The last two tasks create the configuration files for the clients. They run
only on the host that executes the Ansible playbook and not on the wireguard
servers. The first task creates the directory `wireguard-clients` in the
directory of the Ansible inventory. The second task creates a configuration
file in this directory for each client from the template `wg-client.conf.j2`.
The file name is the name of the wireguard peer with the suffix `.conf`.

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
  - name: client1
    # server side
    public_key: 55BbgL/uvNBfaXko8UodGy93DnaQfiWDWaDCuPhh1z4=
    allowed_ips: 10.20.21.2
    # client side
    client_address: 10.20.21.2/24
    client_endpoint: vpn.s1.lan:51000
    client_allowed_ips: "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
  # Client IP 10.20.21.3:
  - name: client2
    # server side
    public_key: OinFs2iWV0/X3jDmsjnhAJLZP/127E8Ly13URQQB7BM=
    allowed_ips: 10.20.21.3
    # client side
    client_address: 10.20.21.3/24
    client_endpoint: vpn.s1.lan:51000
    client_allowed_ips: "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
  # Client IP 10.20.21.4:
  - name: client3
    # server side
    public_key: KqvsNv2Nppt37z3BGIFOZIwfA3QcnNsEqRfTCcNiERY=
    allowed_ips: 10.20.21.4
    # client side
    client_address: 10.20.21.4/24
    client_endpoint: vpn.s1.lan:51000
    client_allowed_ips: "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
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
  - name: client1
    # server side
    public_key: xImnlRiMQnY1vuOiVFx5W+8YDXYYbPMy3ROMr2HorBw=
    allowed_ips: 10.20.22.2
    # client side
    client_address: 10.20.22.2/24
    client_endpoint: vpn.s2.lan:51000
    client_allowed_ips: "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
  # Client IP 10.20.22.3:
  - name: client2
    # server side
    public_key: vPOB/d3CDETUuBI0OY/WbK5cjOnTGxYoY8yppwnpR3I=
    allowed_ips: 10.20.22.3
    # client side
    client_address: 10.20.22.3/24
    client_endpoint: vpn.s2.lan:51000
    client_allowed_ips: "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
  # Client IP 10.20.22.4:
  - name: client3
    # server side
    public_key: UY7GDMxHelkkMojD56kvuUJE+lL2dR38f4V8uCGfGg8=
    allowed_ips: 10.20.22.4
    # client side
    client_address: 10.20.22.4/24
    client_endpoint: vpn.s2.lan:51000
    client_allowed_ips: "10.20.1.0/24, 10.20.21.0/24, 10.20.2.0/24, 10.20.22.0/24"
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
