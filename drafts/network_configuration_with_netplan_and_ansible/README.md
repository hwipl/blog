# Network Configuration with Netplan and Ansible

This document describes how you can configure Linux network settings with
Netplan and Ansible. It shows the Netplan configuration as well as the Ansible
role, playbook and configuration that are used to configure each host
automatically.

## Overview

The hosts in this document run Ubuntu 22.04 LTS. Their network settings are
configured automatically with Ansible. The network configuration assumes the
network in the following figure:

```
                   Other Networks
                          |
..........................|...............................................
: Site 1                  |                            : Site 2          :
: 10.20.1.0/24            |                            : 10.20.2.0/24    :
:                         |                            :                 :
:                  +----[ext0]---+                   __:                 :
:                  |             |                  |  :                 :
:                  | Node 1      |                  |  :                 :
:                  | Router      |                  |  :                 :
:                  | DNS Server  |                  |  :                 :
:                  | VPN Server  |                  |  :                 :
:                  | 10.20.21.1  |                  |  :                 :
:                  | 10.20.1.1   |                  |  :.................:
:                  |             |                  |  : Local Network 1 :
:                  |  [int-br0]  |                  |__: 10.21.0.0/16    :
:                  +----[int0]---+                  |  :                 :
:          _______________|_______________          |  :.................:
:         |               |               |         |  : Local Network 2 :
:  +----[int0]---+ +----[int0]---+ +-------------+  |__: 10.22.0.0/16    :
:  |  [int-br0]  | |             | | Node 4      |  |  :                 :
:  |             | |             | |             |__|  :.................:
:  | Node 2      | | Node 3      | | Router      |  |  : Local Network 3 :
:  | 10.20.1.2   | | 10.20.1.3   | | 10.20.1.201 |  |__: 10.23.0.0/16    :
:  +-------------+ +-------------+ +-------------+     :                 :
:......................................................:.................:
```

The example network consists of the two sites `Site 1` and `Site 2` and three
additional local networks `Local Network 1`, `Local Network 2` and `Local
Network 3`. `Site 1` contains three Linux nodes: `Node 1`, `Node 2`, `Node 3`.
The fourth node, `Node 4`, is a router that connects the site to the other site
and the three additional local networks. The configuration of `Node 4` is not
examined in this document. `Node 1` has multiple roles: it is a router, DNS
server and VPN server. It connects VPN clients with the site and the site to
other networks, e.g., the Internet. `Site 2` is, except for addresses,
identical to `Site 1`. To save space, it is not examined further in this
document but its configuration is listed in the appendix below. Also, the three
local networks just affect the routing configuration and are not discussed in
more detail.

The three Linux nodes in `Site 1` possess the following network devices. On
`Node 1`, there are multiple network devices. Device `ext0` connects the node
with other networks. Device `int0` connects the node with the other nodes in
the site. Device `int-br0` is a Linux software bridge running on top of `int0`
that connects, e.g., node-local virtual machines to the site. There is also a
device for the VPN but it is configured by the VPN software and not covered in
more detail in this document. On `Node 2`, there are two network devices with a
similar configuration. Device `int0` connects the node to the site and there is
also a Linux software bridge called `int-br0` running on top of `int0`. On
`Node 3`, there is only one device: `int0` connects the node to the site.

The IPv4 addresses relevant for the configuration of the network devices and
the routing in `Site 1` are summarized in the following table:

| Entity          | Device  | IPv4 Address  |
|-----------------|---------|---------------|
| Site 1          |         | 10.20.1.0/24  |
| - VPN           |         | 10.20.21.0/24 |
| - Node 1        | int-br0 | 10.20.1.1     |
|                 |         | 10.20.21.1    |
| - Node 2        | int-br0 | 10.20.1.2     |
| - Node 3        | int0    | 10.20.1.3     |
| - Node 4        |         | 10.20.1.201   |
| Site 2          |         | 10.20.2.0/24  |
| Local Network 1 |         | 10.21.0.0/16  |
| Local Network 2 |         | 10.22.0.0/16  |
| Local Network 3 |         | 10.23.0.0/16  |

The IP addresses in `Site 1` and `Site 2` are `10.20.1.0/24` and
`10.20.2.0/24`. The IP addresses in the three local networks are
`10.21.0.0/16`, `10.22.0.0/16` and `10.23.0.0/16`. In `Site 1`, `Node 1` uses
address `10.20.1.1` on device `int-br0`, `Node 2` uses `10.20.1.2` on `int-br0`
and `Node 3` uses `10.20.1.3` on `int0`. The VPN in `Site 1` uses IP addresses
in `10.20.21.0/24` and `Node 1` uses address `10.20.21.1` in the VPN.

Further details of the three Linux nodes in `Site 1` and their network devices
are shown in the following.

The settings of the network devices are listed in the following table:

| Node   | Device   | MAC Address       | MTU  | Bridge Info         |
|--------|----------|-------------------|------|---------------------|
| Node 1 | ext0     | ca:fe:ca:fe:11:01 | 1500 | No bridge           |
|        | int0     | ca:fe:ca:fe:11:03 | 1500 | Member of int-br0   |
|        | int-br0  | ca:fe:ca:fe:11:03 | 1500 | Members: int0       |
|        |          |                   |      |                     |
| Node 2 | int0     | ca:fe:ca:fe:12:01 | 1500 | Member of int-br0   |
|        | int-br0  | ca:fe:ca:fe:12:01 | 1500 | Members: int0       |
|        |          |                   |      |                     |
| Node 3 | int0     | ca:fe:ca:fe:13:01 | 1500 | No bridge           |

On `Node 1`, the device `ext0` with the MAC address `ca:fe:ca:fe:11:01` is not
a member of a software bridge. The device `int0` with the MAC address
`ca:fe:ca:fe:11:03` is the only member of the software bridge `int-br0`. The
MAC address of the software bridge (device `int-br0`) is the same as its member
device. On `Node 2`, the device `int0` with the MAC address `ca:fe:ca:fe:12:01`
is the only member of the software bridge `int-br0`. The MAC address of the
software bridge (device `int-br0`) is the same as its member device. On `Node
3`, the device `int0` with the MAC address `ca:fe:ca:fe:13:01` is not a member
of a software bridge. The MTU of all network devices on all nodes is set to
`1500`.

The IPv4 configuration of the network devices on these nodes is shown in the
following table:

| Node   | Device   | IPv4 Address | IPv4 Routes                  |
|--------|----------|--------------|------------------------------|
| Node 1 | ext0     | DHCP         | DHCP                         |
|        | int0     | None         | None                         |
|        | int-br0  | 10.20.1.1/24 | 10.20.0.0/16 via 10.20.1.201 |
|        |          |              | 10.21.0.0/16 via 10.20.1.201 |
|        |          |              | 10.22.0.0/16 via 10.20.1.201 |
|        |          |              | 10.23.0.0/16 via 10.20.1.201 |
|        |          |              |                              |
| Node 2 | int0     | None         | None                         |
|        | int-br0  | 10.20.1.2/24 | default via 10.20.1.1        |
|        |          |              | 10.20.21.0/24 via 10.20.1.1  |
|        |          |              | 10.20.0.0/16 via 10.20.1.201 |
|        |          |              | 10.21.0.0/16 via 10.20.1.201 |
|        |          |              | 10.22.0.0/16 via 10.20.1.201 |
|        |          |              | 10.23.0.0/16 via 10.20.1.201 |
|        |          |              |                              |
| Node 3 | int0     | 10.20.1.3/24 | default via 10.20.1.1        |
|        |          |              | 10.20.21.0/24 via 10.20.1.1  |
|        |          |              | 10.20.0.0/16 via 10.20.1.201 |
|        |          |              | 10.21.0.0/16 via 10.20.1.201 |
|        |          |              | 10.22.0.0/16 via 10.20.1.201 |
|        |          |              | 10.23.0.0/16 via 10.20.1.201 |

On `Node 1`, the device `ext0` retrieves its IPv4 address and routes via DHCP.
No addresses and no routes are configured on device `int0`, because it's just a
member of the bridge `int-br0`. The bridge device `int-br0` has the address
`10.20.1.1/24` and no additional routes. On `Node 2`, there is also no IP
address or route on device `int0`, because it's a member of the bridge
`int-br0`. The bridge device `int-br0` has IP address `10.20.1.2/24` and the
default route is set to `10.20.1.1` (i.e. `Node 1`). On `Node 3`, device `int0`
has IP address `10.20.1.3/24` and the default route is also set to `10.20.1.1`
(i.e. `Node 1`).

The DNS configuration of the network devices on the three nodes is shown in the
following table:

| Node   | Device  | DNS Server | Search Domains              |
|--------|---------|------------|-----------------------------|
| Node 1 | ext0    | DHCP       | DHCP                        |
| Node 2 | int-br0 | 10.20.1.1  | s1.network.lan, network.lan |
| Node 3 | int0    | 10.20.1.1  | s1.network.lan, network.lan |

The DNS server on `Node 1` is used by the other nodes in the site. It resolves
all DNS names for them including the internal domains `network.lan` and
`s1.network.lan`. So the DNS configuration on `Node 2` and `Node 3` sets the IP
address of `Node 1` as DNS server as well as `s1.network.lan` and `network.lan`
as search domains for their network devices. `Node 1` retrieves its DNS
configuration via DHCP on the network device that connects the node to the
other networks (i.e. `ext0`).

## Network Configuration

The network configuration of the three Linux nodes that results from the
example network shown above is described in this section. The network
configuration of a node is stored on the node in a [YAML][yaml] file with the
[Netplan structure][structure] in the directory `/etc/netplan`. These
configuration files start with a `network` block that contains all settings.
Common entries in `network` are `version` that sets the version of the Netplan
configuration format to `2`, `renderer` that sets `networkd` as underlying
configuration tool and `ethernets` that configures network devices.
Additionally, there is also `bridges` on Nodes 1 and 2 that contains the
configuration of the Linux software bridges. The node-specific settings in
`ethernets` and `bridges` are described together with the respective
configuration file of each node below.

### Node 1

On Node 1, the network is configured as follows in the file
`/etc/netplan/90-site1-node1.yaml`:

```yaml
# Network configuration of Node 1
network:
  version: 2
  renderer: networkd
  ethernets:
    ext0:
      match:
        macaddress: "ca:fe:ca:fe:11:01"
      set-name: ext0
      mtu: 1500
      dhcp4: true
    int0:
      match:
        macaddress: "ca:fe:ca:fe:11:03"
      set-name: int0
      mtu: 1500
      dhcp4: false
  bridges:
    int-br0:
      interfaces:
      - int0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:11:03"
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.1.1/24
      routes:
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.1.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.1.201
      - to: 10.22.0.0/16
        via: 10.20.1.201
      - to: 10.23.0.0/16
        via: 10.20.1.201
```

The network devices `ext0` and `int0` are configured in `ethernets`. Both
devices are identified by their MAC addresses with `macaddress` in `match`.
The names of the network devices in Linux are set to `ext0` and `int0` with
`set-name`. The MTU of the devices is set to `1500` in `mtu`. In `dhcp4`,
DHCPv4 is enabled on `ext0` with `true` and disabled on `int0` with `false`.
The Linux software bridge device `int-br0` is configured in `bridges`. The
member device `int0` is set in `interfaces`. Parameters of the bridge are set
in `parameters`: forward delay is set to `15` seconds in `forward-delay` and
STP is disabled with `stp: false`. The MAC address of the bridge device is set
to `ca:fe:ca:fe:11:03` (i.e. device `int0`) in `macaddress`, the MTU is set to
`1500` in `mtu` and DHCPv4 is disabled with `dhcpv4: false`. The IP addresses
of the bridge are configured in `addresses`. Only address `10.20.1.1/24` is set
there. Additional routes are configured on the bridge in `routes`. The routes
to `10.20.0.0/16`, `10.21.0.0/16`, `10.22.0.0/16` and `10.23.0.0/16` are all
set to go via `10.20.1.201` with `to` and `via`.

### Node 2

On Node 2, the network is configured as follows in the file
`/etc/netplan/90-site1-node2.yaml`:

```yaml
# Network configuration of Node 2
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: "ca:fe:ca:fe:12:01"
      set-name: int0
      mtu: 1500
      dhcp4: false
  bridges:
    int-br0:
      interfaces:
      - int0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:12:01"
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.1.2/24
      routes:
      - to: default
        via: 10.20.1.1
      # vpn traffic
      - to: 10.20.21.0/24
        via: 10.20.1.1
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.1.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.1.201
      - to: 10.22.0.0/16
        via: 10.20.1.201
      - to: 10.23.0.0/16
        via: 10.20.1.201
      nameservers:
        addresses:
        - 10.20.1.1
        search:
        - s1.network.lan
        - network.lan
```

The network device `int0` is configured in `ethernets`. In `match`, it is
identified by its MAC address with `macaddress: "ca:fe:ca:fe:12:01"`. Its name
is set to `int0` with `set-name: int0` and its MTU to `1500` with `mtu: 1500`.
DHCPv4 is disabled with `dhcp4: false`. The software bridge `int-br0` is
configured in `bridges`. Its only device is set to `int0` in `interfaces`. The
bridge parameters are set in `parameters`: forward delay is set to `15` seconds
with `forward-delay: 15` and STP is disabled with `stp: false`. The MAC address
of the bridge is set to the MAC address of device `int0` with `macaddress:
"ca:fe:ca:fe:12:01"` and its MTU to `1500` with `mtu: 1500`. DHCPv4 is disabled
with `dhcp4: false`. The IP addresses of the bridge are set in `addresses`.
Only address `10.20.1.2/24` is set on the device. Additional routes are
configured in `routes`. The default route is set to go via `10.20.1.1`. The VPN
route `10.20.21.0/24` also goes via `10.20.1.1`. The other routes to
`10.20.0.0/16`, `10.21.0.0/16`, `10.22.0.0/16` and `10.23.0.0/16` are all
configured to go through `10.20.1.201`. Finally, name server settings are
configured for the bridge in `nameservers`. The addresses of the name servers
are set in `addresses`. The only name server is `10.20.1.1`. The search domains
are set to `s1.network.lan` and `network.lan` in `search`.

### Node 3

On Node 3, the network is configured as follows in the file
`/etc/netplan/90-site1-node3.yaml`:

```yaml
# Network configuration of Node 3
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: "ca:fe:ca:fe:13:02"
      set-name: int0
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.1.3/24
      routes:
      - to: default
        via: 10.20.1.1
      # vpn traffic
      - to: 10.20.21.0/24
        via: 10.20.1.1
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.1.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.1.201
      - to: 10.22.0.0/16
        via: 10.20.1.201
      - to: 10.23.0.0/16
        via: 10.20.1.201
      nameservers:
        addresses:
        - 10.20.1.1
        search:
        - s1.network.lan
        - network.lan
```

In `ethernets`, network device `int0` is configured. In `match`, `macaddress:
"ca:fe:ca:fe:13:02"` identifies the device by its MAC address. `set-name: int0`
sets its name in Linux to `int0`. `mtu: 1500` sets its MTU to `1500`. `dhcp4:
false` disables DHCPv4 on the device. `addresses` configures the IP addresses
of the device. Only `10.20.1.3/24` is set on the device. Additional routes are
configured in `routes`. The default route and the VPN route to `10.20.21.0/24`
go via `10.20.1.1`. The routes to `10.20.0.0/16`, `10.21.0.0/16`,
`10.22.0.0/16` and `10.23.0.0/16` go via `10.20.1.201`. `nameservers`
configures the DNS settings of the device. Here, `addresses` sets the only name
server to `10.20.1.1` and `search` sets the search domains to `s1.network.lan`
and `network.lan`.

## Ansible

Ansible allows for automatic network configuration with Netplan. Ansible uses
[roles][roles] and [playbooks][playbooks]. Roles consist of tasks, templates
and handlers. Tasks are the individual installation and configuration steps.
They use the [templates][templates] to generate configuration files and trigger
events that are handled by the [handlers][handlers].

A role and a playbook are used to deploy the network settings. The playbook
assigns the role to all nodes of a group defined in the Ansible
[inventory][inventory]. The configuration of the nodes is derived from
variables in the inventory. Each site uses a different inventory to allow for
site-specific configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

### Role

The Ansible role is called `netplan` and structured as shown in the listing
below:

```
roles/netplan/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── netplan-network.yaml.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and one template file. The tasks use the template
`netplan-network.yaml.j2` to create the configuration file for Netplan.

#### Handlers

A handler is defined as follows in the file `roles/netplan/handlers/main.yml`:

```yaml
---
# Netplan handlers

- name: Apply Netplan configuration
  become: true
  ansible.builtin.command: netplan apply
```

The handler is called `Apply Netplan configuration` and applies the network
configuration with the [command module][command] when it is triggered. It
requires root privileges to manipulate the system configuration, so
[become][become] is set to `true` for [privilege escalation][privilege]. It
calls the command `netplan apply` to apply the network configuration.

#### Tasks

The tasks are defined as follows in the file `roles/netplan/tasks/main.yml`:

```yaml
---
# these tasks setup a network configuration with Netplan

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure Netplan is installed
  become: true
  ansible.builtin.apt:
    name: netplan.io
    state: present
- name: Create Netplan configuration from template
  become: true
  ansible.builtin.template:
    src: netplan-network.yaml.j2
    dest: "{{ netplan_config_file }}"
    owner: root
    group: root
    mode: '0600'
  notify:
    - Apply Netplan configuration
```

These tasks make sure Netplan is installed with the [apt module][apt],
configure it with the template and the [template module][template] and trigger
the event to apply the network configuration with [notify][notify] if the
configuration file is changed. All tasks need root privileges to manipulate the
system configuration. So, [become][become] is set to `true`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs Netplan with `apt` if it is not already installed.
   In Ubuntu, it should already be installed by default.
3. The third task creates or updates the network configuration from the
   template `netplan-network.yaml.j2`. The directory and file name are read
   from the Ansible variable `netplan_config_file`. The file owner and group
   are set to `root`. File permissions are set to `600`. If the configuration
   changed, the task triggers the event to apply the network configuration.

#### Templates

The template for the network configuration file is defined as [Jinja2
template][jinja2]. It is defined as follows in the file
`roles/netplan/templates/netplan-network.yaml.j2`:

```jinja
network:
  {{ network | to_nice_yaml(indent=2,sort_keys=false) | indent(width=2) | trim }}
```

The template creates a Netplan configuration as shown in the Network
Configuration section above from the Ansible configuration. To this end, it
converts the network configuration in the Ansible variable `network` to a yaml
file that Netplan can use. So, the content of the `network` variable must be a
valid Netplan configuration (see Configuration below).

### Playbook

The playbook is defined as follows in the file `netplan.yml`:

```yaml
---
- name: Configure network using Netplan
  hosts: netplan_hosts

  roles:
    - netplan
```

The playbook assigns the role `netplan` that is described above to all hosts in
the group `netplan_hosts`. On execution, this playbook runs all the tasks of
the role on all the hosts in the group to configure the network settings.

### Configuration

The configuration is derived from host files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use to configure the nodes. Group-specific configuration in `group_vars` is not
used here. As mentioned before, only the configuration of Site 1 is described
here but you can find the configuration of Site 2 in the appendix below.

The Ansible hosts are defined as follows in the files `site1/hosts`:

```ini
[netplan_hosts]
node1
node2
node3
```

The hosts file defines the group `netplan_hosts` and assigns the nodes `node1`,
`node2` and `node3` to it. Thus, the network settings of the three Linux nodes
Node 1, Node 2 and Node 3 are configured by the playbook.

The host-specific configuration of the nodes is in the `host_vars` of
each node. The respective files are shown in the following.

#### Node 1

The host-specific configuration of Node 1 is in the `host_vars` of `node1` in
each site. The configuration for Site 1 defined as follows in the file
`site1/host_vars/node1`:

```yaml
---
# Netplan network configuration
netplan_config_file: /etc/netplan/90-site1-node1.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ext0:
      match:
        macaddress: "ca:fe:ca:fe:11:01"
      set-name: ext0
      mtu: 1500
      dhcp4: true
    int0:
      match:
        macaddress: "ca:fe:ca:fe:11:03"
      set-name: int0
      mtu: 1500
      dhcp4: false
  bridges:
    int-br0:
      interfaces:
      - int0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:11:03"
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.1.1/24
      routes:
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.1.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.1.201
      - to: 10.22.0.0/16
        via: 10.20.1.201
      - to: 10.23.0.0/16
        via: 10.20.1.201
```

The name of the Netplan configuration file is set in the Ansible variable
`netplan_config_file` to the value `/etc/netplan/90-site1-node1.yaml`.
The network configuration is defined in the variable `network`. It is identical
to the configuration of Node 1 shown in the Network Configuration section.

#### Node 2

The host-specific configuration of Node 2 is in the `host_vars` of `node2` in
each site. The configuration for Site 1 defined as follows in the file
`site1/host_vars/node2`:

```yaml
---
# Netplan network configuration
netplan_config_file: /etc/netplan/90-site1-node2.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: "ca:fe:ca:fe:12:01"
      set-name: int0
      mtu: 1500
      dhcp4: false
  bridges:
    int-br0:
      interfaces:
      - int0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:12:01"
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.1.2/24
      routes:
      - to: default
        via: 10.20.1.1
      # vpn traffic
      - to: 10.20.21.0/24
        via: 10.20.1.1
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.1.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.1.201
      - to: 10.22.0.0/16
        via: 10.20.1.201
      - to: 10.23.0.0/16
        via: 10.20.1.201
      nameservers:
        addresses:
        - 10.20.1.1
        search:
        - s1.network.lan
        - network.lan
```

The name of the Netplan configuration file is set in the Ansible variable
`netplan_config_file` to the value `/etc/netplan/90-site1-node2.yaml`.
The network configuration is defined in the variable `network`. It is identical
to the configuration of Node 2 shown in the Network Configuration section.

#### Node 3

The host-specific configuration of Node 3 is in the `host_vars` of `node3` in
each site. The configuration for Site 1 is defined as follows in the file
`site1/host_vars/node3`:

```yaml
---
# Netplan network configuration
netplan_config_file: /etc/netplan/90-site1-node3.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: "ca:fe:ca:fe:13:02"
      set-name: int0
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.1.3/24
      routes:
      - to: default
        via: 10.20.1.1
      # vpn traffic
      - to: 10.20.21.0/24
        via: 10.20.1.1
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.1.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.1.201
      - to: 10.22.0.0/16
        via: 10.20.1.201
      - to: 10.23.0.0/16
        via: 10.20.1.201
      nameservers:
        addresses:
        - 10.20.1.1
        search:
        - s1.network.lan
        - network.lan
```

The name of the Netplan configuration file is set in the Ansible variable
`netplan_config_file` to the value `/etc/netplan/90-site1-node3.yaml`.
The network configuration is defined in the variable `network`. It is identical
to the configuration of Node 3 shown in the Network Configuration section.

### Deployment

The network configuration can be deployed with the Ansible role, configuration
and playbook described above. You can use the following
[ansible-playbook][ansible-playbook] command:

```console
$ # site 1
$ ansible-playbook -i site1/hosts netplan.yml
```

The `ansible-playbook` command runs the playbook `netplan.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
command configures all hosts in Site 1. After successful execution of the
command above, the network settings should be configured and active.

## Conclusion

This document describes how you can automatically configure Linux network
settings with Netplan and Ansible on Ubuntu 22.04 LTS. After presenting an
example network and configuration, an Ansible Role is shown that contains the
installation and configuration tasks as well as templates. The tasks use the
templates together with Ansible variables to create the network configuration.
Finally, an Ansible Playbook is used with the `ansible-playbook` command to run
the tasks and, thus, configure the network settings on all hosts. All sections
contain configuration or code examples you can use as a basis for your own
setup. Also, you can find links to the code and configuration examples in the
appendix below.

## Appendix: Site 2 Configuration

The IPv4 addresses relevant for the configuration of the network devices and
the routing in `Site 2` are summarized in the following table:

| Entity          | Device  | IPv4 Address  |
|-----------------|---------|---------------|
| Site 1          |         | 10.20.1.0/24  |
| Site 2          |         | 10.20.2.0/24  |
| - VPN           |         | 10.20.22.0/24 |
| - Node 1        | int-br0 | 10.20.2.1     |
|                 |         | 10.20.22.1    |
| - Node 2        | int-br0 | 10.20.2.2     |
| - Node 3        | int0    | 10.20.2.3     |
| - Node 4        |         | 10.20.2.201   |
| Local Network 1 |         | 10.21.0.0/16  |
| Local Network 2 |         | 10.22.0.0/16  |
| Local Network 3 |         | 10.23.0.0/16  |

Further details of the three Linux nodes in `Site 2` and their network devices
are shown in the following.

The settings of the network devices are listed in the following table:

| Node   | Device   | MAC Address       | MTU  | Bridge Info         |
|--------|----------|-------------------|------|---------------------|
| Node 1 | ext0     | ca:fe:ca:fe:21:01 | 1500 | No bridge           |
|        | int0     | ca:fe:ca:fe:21:03 | 1500 | Member of int-br0   |
|        | int-br0  | ca:fe:ca:fe:21:03 | 1500 | Members: int0       |
|        |          |                   |      |                     |
| Node 2 | int0     | ca:fe:ca:fe:22:01 | 1500 | Member of int-br0   |
|        | int-br0  | ca:fe:ca:fe:22:01 | 1500 | Members: int0       |
|        |          |                   |      |                     |
| Node 3 | int0     | ca:fe:ca:fe:23:01 | 1500 | No bridge           |

The IPv4 configuration of the network devices on these nodes is shown in the
following table:

| Node   | Device   | IPv4 Address | IPv4 Routes                  |
|--------|----------|--------------|------------------------------|
| Node 1 | ext0     | DHCP         | DHCP                         |
|        | int0     | None         | None                         |
|        | int-br0  | 10.20.2.1/24 | 10.20.0.0/16 via 10.20.2.201 |
|        |          |              | 10.21.0.0/16 via 10.20.2.201 |
|        |          |              | 10.22.0.0/16 via 10.20.2.201 |
|        |          |              | 10.23.0.0/16 via 10.20.2.201 |
|        |          |              |                              |
| Node 2 | int0     | None         | None                         |
|        | int-br0  | 10.20.2.2/24 | default via 10.20.2.1        |
|        |          |              | 10.20.22.0/24 via 10.20.2.1  |
|        |          |              | 10.20.0.0/16 via 10.20.2.201 |
|        |          |              | 10.21.0.0/16 via 10.20.2.201 |
|        |          |              | 10.22.0.0/16 via 10.20.2.201 |
|        |          |              | 10.23.0.0/16 via 10.20.2.201 |
|        |          |              |                              |
| Node 3 | int0     | 10.20.2.3/24 | default via 10.20.2.1        |
|        |          |              | 10.20.22.0/24 via 10.20.2.1  |
|        |          |              | 10.20.0.0/16 via 10.20.2.201 |
|        |          |              | 10.21.0.0/16 via 10.20.2.201 |
|        |          |              | 10.22.0.0/16 via 10.20.2.201 |
|        |          |              | 10.23.0.0/16 via 10.20.2.201 |

The DNS configuration of the network devices on the three nodes is shown in the
following table:

| Node   | Device  | DNS Server | Search Domains              |
|--------|---------|------------|-----------------------------|
| Node 1 | ext0    | DHCP       | DHCP                        |
| Node 2 | int-br0 | 10.20.2.1  | s2.network.lan, network.lan |
| Node 3 | int0    | 10.20.2.1  | s2.network.lan, network.lan |

The Ansible configuration is shown in the following.

The Ansible hosts are defined as follows in the file `site2/hosts`:

```ini
[netplan_hosts]
node1
node2
node3
```

The host-specific configuration of Node 1 is defined as follows in the file
`site2/host_vars/node1`:

```yaml
---
# Netplan network configuration
netplan_config_file: /etc/netplan/90-site2-node1.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ext0:
      match:
        macaddress: "ca:fe:ca:fe:21:01"
      set-name: ext0
      mtu: 1500
      dhcp4: true
    int0:
      match:
        macaddress: "ca:fe:ca:fe:21:03"
      set-name: int0
      mtu: 1500
      dhcp4: false
  bridges:
    int-br0:
      interfaces:
      - int0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:21:03"
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.2.1/24
      routes:
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.2.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.2.201
      - to: 10.22.0.0/16
        via: 10.20.2.201
      - to: 10.23.0.0/16
        via: 10.20.2.201
```

The host-specific configuration of Node 2 is defined as follows in the file
`site2/host_vars/node2`:

```yaml
---
# Netplan network configuration
netplan_config_file: /etc/netplan/90-site2-node2.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: "ca:fe:ca:fe:22:01"
      set-name: int0
      mtu: 1500
      dhcp4: false
  bridges:
    int-br0:
      interfaces:
      - int0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:22:01"
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.2.2/24
      routes:
      - to: default
        via: 10.20.2.1
      # vpn traffic
      - to: 10.20.22.0/24
        via: 10.20.2.1
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.2.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.2.201
      - to: 10.22.0.0/16
        via: 10.20.2.201
      - to: 10.23.0.0/16
        via: 10.20.2.201
      nameservers:
        addresses:
        - 10.20.2.1
        search:
        - s2.network.lan
        - network.lan
```

The host-specific configuration of Node 3 is defined as follows in the file
`site2/host_vars/node3`:

```yaml
---
# Netplan network configuration
netplan_config_file: /etc/netplan/90-site2-node3.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: "ca:fe:ca:fe:23:02"
      set-name: int0
      mtu: 1500
      dhcp4: false
      addresses:
      - 10.20.2.3/24
      routes:
      - to: default
        via: 10.20.2.1
      # vpn traffic
      - to: 10.20.22.0/24
        via: 10.20.2.1
      # other site traffic
      - to: 10.20.0.0/16
        via: 10.20.2.201
      # other local network traffic
      - to: 10.21.0.0/16
        via: 10.20.2.201
      - to: 10.22.0.0/16
        via: 10.20.2.201
      - to: 10.23.0.0/16
        via: 10.20.2.201
      nameservers:
        addresses:
        - 10.20.2.1
        search:
        - s2.network.lan
        - network.lan
```

The configuration can be deployed like in Site 1 with the following command:

```console
$ # site 2
$ ansible-playbook -i site2/hosts netplan.yml
```

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/netplan/roles/netplan)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/netplan/netplan.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/netplan/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/netplan/examples/site2)

[yaml]: https://yaml.org/
[structure]: https://netplan.readthedocs.io/en/stable/reference/
[roles]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
[playbooks]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html
[templates]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html
[handlers]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html
[inventory]: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
[ansible-playbook]: https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html
[command]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html
[become]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html#become-directives
[privilege]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html
[apt]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html
[template]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html
[notify]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html#notifying-handlers
[jinja2]: https://jinja.palletsprojects.com/en/latest/templates/
