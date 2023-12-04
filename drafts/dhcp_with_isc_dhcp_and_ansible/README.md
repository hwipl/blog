# DHCP with ISC DHCP and Ansible

This document describes how you can deploy ISC DHCP servers with Ansible. It
shows the DHCP configuration of ISC DHCP as well as the Ansible role, playbook
and configuration that are used to install and configure each DHCP server
automatically.

## Overview

The DHCP servers in this document run the ISC DHCP Server on Ubuntu 22.04 LTS.
ISC DHCP is installed and configured automatically with Ansible. This includes
the configuration of the used network interfaces as well as the subnets. The
DHCP configuration only contains IPv4 addressing.

The DHCP configuration in this document assumes the network in the following
figure:

```
.........................................................................
: Site 1                            :                            Site 2 :
: 10.20.1.0/24                      :                      10.20.2.0/24 :
: 10.20.201.0/24                    :                    10.20.202.0/24 :
:                                   :                                   :
:          +-------------+          :          +-------------+          :
:          | Node 1      |          :          | Node 1      |          :
:          | DHCP Server |          :          | DHCP Server |          :
:          | 10.20.1.1   |          :          | 10.20.2.1   |          :
:          | 10.20.201.1 |          :          | 10.20.202.1 |          :
:          +-------------+          :          +-------------+          :
:          _______|_______          :          _______|_______          :
:         |               |         :         |               |         :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:  | Node 2      | | Node 3      |  :  | Node 2      | | Node 3      |  :
:  | DHCP Client | | DHCP Client |  :  | DHCP Client | | DHCP Client |  :
:  | 10.20.1.2   | | 10.20.1.3   |  :  | 10.20.2.2   | | 10.20.2.3   |  :
:  | 10.20.201.2 | | 10.20.201.3 |  :  | 10.20.202.2 | | 10.20.202.3 |  :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:...................................:...................................:
                        Network: 10.20.0.0/16
```

The example network consists of the two sites `Site 1` and `Site 2`. Each site
contains three nodes. `Node 1` runs the DHCP server. `Node 2` and `Node 3` are
DHCP clients. The DHCP clients in a site use the DHCP server in the same site.
All nodes have two network interfaces: one for regular network traffic and one
for management access. Both interfaces use different IP address ranges and, on
the clients, they are configured by DHCP.

The network uses IPs in the range `10.20.0.0/16`: Site 1 gets `10.20.1.0/24`
for regular traffic and `10.20.201.0/24` for management. Site 2 gets
`10.20.2.0/24` for regular traffic and `10.20.202.0/24` for management. Nodes
1, 2 and 3 in Site 1 get the IPs `10.20.1.1`, `10.20.1.2` and `10.20.1.3` for
regular traffic. For management, they get `10.20.201.1`, `10.20.201.2` and
`10.20.201.3`. In Site 2, they get `10.20.2.1`, `10.20.2.2`, `10.20.2.3` as
well as `10.20.202.1`, `10.20.202.2` and `10.20.202.3`.

The IP addressing and the MAC addresses of all network interfaces are shown in
the following two tables. The first table shows Site 1:

| Entity            | IP Address     |  MAC Address      |
|-------------------|----------------|-------------------|
| Site 1            | 10.20.1.0/24   |                   |
| - Node 1          | 10.20.1.1      | ca:fe:ca:fe:11:01 |
| - Node 2          | 10.20.1.2      | ca:fe:ca:fe:12:01 |
| - Node 3          | 10.20.1.3      | ca:fe:ca:fe:13:01 |
| Site 1 Management | 10.20.201.0/24 |                   |
| - Node 1          | 10.20.201.1    | ca:fe:ca:fe:11:0a |
| - Node 2          | 10.20.201.2    | ca:fe:ca:fe:12:0a |
| - Node 3          | 10.20.201.3    | ca:fe:ca:fe:13:0a |

The second table shows Site 2:

| Entity            | IP Address     |  MAC Address      |
|-------------------|----------------|-------------------|
| Site 2            | 10.20.2.0/24   |                   |
| - Node 1          | 10.20.2.1      | ca:fe:ca:fe:21:01 |
| - Node 2          | 10.20.2.2      | ca:fe:ca:fe:22:01 |
| - Node 3          | 10.20.2.3      | ca:fe:ca:fe:23:01 |
| Site 2 Management | 10.20.202.0/24 |                   |
| - Node 1          | 10.20.202.1    | ca:fe:ca:fe:21:0a |
| - Node 2          | 10.20.202.2    | ca:fe:ca:fe:22:0a |
| - Node 3          | 10.20.202.3    | ca:fe:ca:fe:23:0a |

The DHCP configuration uses fixed addresses. So, the network interfaces of the
nodes always get these IP addresses based on their MAC addresses.

## DHCP Configuration

The configuration of the ISC DHCP servers is described in the following
subsections. First, the configuration of the interfaces used by the servers is
shown. Then, the main configuration file is described. It contains the subnets
that specify the configuration of the nodes in the network.

### Interfaces

The network interfaces the DHCP server uses and listens on are configured as
follows in the file `/etc/default/isc-dhcp-server`:

```
INTERFACESv4="eth0 eth1"
INTERFACESv6=""
```

The server uses its interfaces `eth0` and `eth1` for DHCPv4. No interface is
used for DHCPv6. So, the servers do not provide DHCP-based configuration of
IPv6 addresses.

### Subnets

The subnets of the DHCP servers are configured in the file
`/etc/dhcp/dhcpd.conf`. The following listing shows the file for Site 1:

```
default-lease-time 86400;
max-lease-time 86400;

subnet 10.20.1.0 netmask 255.255.255.0 {

        option routers                  10.20.1.1;
        option domain-name-servers      10.20.1.1;
        option ntp-servers              10.20.1.1;
        option domain-name              "s1.network.lan";

        host node2 {
                hardware ethernet ca:fe:ca:fe:12:01;
                fixed-address 10.20.1.2;
        host node3 {
                hardware ethernet ca:fe:ca:fe:13:01;
                fixed-address 10.20.1.3;
        }
}
subnet 10.20.201.0 netmask 255.255.255.0 {

        option routers                  10.20.201.1;
        option domain-name-servers      10.20.201.1;
        option ntp-servers              10.20.201.1;
        option domain-name              "s1-mgmt.network.lan";

        host node2 {
                hardware ethernet ca:fe:ca:fe:12:0a;
                fixed-address 10.20.201.2;
        host node3 {
                hardware ethernet ca:fe:ca:fe:13:0a;
                fixed-address 10.20.201.3;
        }
}
```

The next listing shows the file for Site 2:

```
default-lease-time 86400;
max-lease-time 86400;

subnet 10.20.2.0 netmask 255.255.255.0 {

        option routers                  10.20.2.1;
        option domain-name-servers      10.20.2.1;
        option ntp-servers              10.20.2.1;
        option domain-name              "s2.network.lan";

        host node2  {
                hardware ethernet ca:fe:ca:fe:22:01;
                fixed-address 10.20.2.2;
        host node3  {
                hardware ethernet ca:fe:ca:fe:23:01;
                fixed-address 10.20.2.3;
        }
}
subnet 10.20.202.0 netmask 255.255.255.0 {

        option routers                  10.20.202.1;
        option domain-name-servers      10.20.202.1;
        option ntp-servers              10.20.202.1;
        option domain-name              "s2-mgmt.network.lan";

        host node2 {
                hardware ethernet ca:fe:ca:fe:22:0a;
                fixed-address 10.20.202.2;
        host node3 {
                hardware ethernet ca:fe:ca:fe:23:0a;
                fixed-address 10.20.202.3;
        }
}
```

The options `default-lease-time` and `max-lease-time` set the default and
maximum lease times to 86400 seconds for all subnets. The two `subnet` blocks
configure the two subnets. The first subnet contains the configuration of the
regular network interfaces. The second subnet contains the configuration of the
management interfaces. The subnet line specifies the IP address range with the
IP address and netmask. The options `routers`, `domain-name-servers`,
`ntp-servers` and `domain-name` set the routers, DNS servers, NTP servers and
domain names used in the subnet. The two `host` blocks contain the
configuration of the nodes `node2` and `node3` in the subnet. The settings
`hardware ethernet` and `fixed-address` specify the MAC address of the node and
the fixed IP address allocated to the node.

TODO: mention IPs, domain names, MACs? Move this under each config file?

## Ansible

### Role

```
roles/dhcpd/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── dhcp.conf.j2
    └── isc-dhcp-server.j2
```

#### Handlers

handlers/main.yml:

```yaml
---
# handlers for dhcpd

- name: Restart dhcpd
  become: true
  ansible.builtin.service:
    name: isc-dhcp-server
    state: restarted
```

#### Tasks

tasks/main.yml:

```yaml
---
# these tasks setup dhcpd with static host configuration

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure dhcpd is installed
  become: true
  ansible.builtin.apt:
    name: isc-dhcp-server
    state: present
- name: Create dhcpd configuration from template
  become: true
  ansible.builtin.template:
    src: dhcp.conf.j2
    dest: "/etc/dhcp/dhcpd.conf"
    owner: root
    group: root
    mode: '0644'
  notify:
    - Restart dhcpd
- name: Create isc-dhcp-server defaults from template
  become: true
  ansible.builtin.template:
    src: isc-dhcp-server.j2
    dest: "/etc/default/isc-dhcp-server"
    owner: root
    group: root
    mode: '0644'
    backup: true
  notify:
    - Restart dhcpd
```

#### Templates

templates/dhcp.conf.j2:

```jinja
default-lease-time 86400;
max-lease-time 86400;

{% for subnet in subnets %}
subnet {{ subnet.ip }} netmask {{ subnet.netmask }} {

        option routers                  {{ subnet.routers }};
        option domain-name-servers      {{ subnet.dns_servers }};
        option ntp-servers              {{ subnet.ntp_servers }};
        option domain-name              "{{ subnet.domain_name }}";

{% for host in subnet.hosts %}
        host {{ host.name }} {
                hardware ethernet {{ host.mac }};
                fixed-address {{ host.ip }};
        }
{% endfor %}
}
{% endfor %}
```

templates/isc-dhcp-server.j2:

```jinja
INTERFACESv4="{{ dhcpd_interfaces }}"
INTERFACESv6=""
```

### Playbook

```yaml
---
- name: Configure dhcp servers
  hosts: dhcp_servers

  roles:
    - dhcpd
```

### Configuration

hosts:

```ini
[dhcp_servers]
node1

```

host_vars/node1:

```yaml
# dhcp server configuration
dhcpd_interfaces: "eth0 eth1"
```

group_vars/dhcp_servers:

```yaml
---
# dhcp server configuration

subnets:
  # site 1 network
  - ip: 10.20.1.0
    netmask: 255.255.255.0
    routers: 10.20.1.1
    dns_servers: 10.20.1.1
    ntp_servers: 10.20.1.1
    domain_name: s1.network.lan
    hosts:
      # - name: node1 # static
      #   mac: ca:fe:ca:fe:11:01
      #   ip: 10.20.1.1
      - name: node2
        mac: ca:fe:ca:fe:12:01
        ip: 10.20.1.2
      - name: node3
        mac: ca:fe:ca:fe:13:01
        ip: 10.20.1.3
  # site 1 management network
  - ip: 10.20.201.0
    netmask: 255.255.255.0
    routers: 10.20.201.1
    dns_servers: 10.20.201.1
    ntp_servers: 10.20.201.1
    domain_name: s1-mgmt.network.lan
    hosts:
      # hosts
      # - name: node1 # static
      #   mac: ca:fe:ca:fe:11:0a
      #   ip: 10.20.201.1
      - name: node2
        mac: ca:fe:ca:fe:12:0a
        ip: 10.20.201.2
      - name: node3
        mac: ca:fe:ca:fe:13:0a
        ip: 10.20.201.3
```

```yaml
---
# dhcp server configuration

subnets:
  # site 2 network
  - ip: 10.20.2.0
    netmask: 255.255.255.0
    routers: 10.20.2.1
    dns_servers: 10.20.2.1
    ntp_servers: 10.20.2.1
    domain_name: s2.network.lan
    hosts:
      # - name: node1 # static
      #   mac: ca:fe:ca:fe:21:01
      #   ip: 10.20.2.1
      - name: node2
        mac: ca:fe:ca:fe:22:01
        ip: 10.20.2.2
      - name: node3
        mac: ca:fe:ca:fe:23:01
        ip: 10.20.2.3
  # site 2 management network
  - ip: 10.20.202.0
    netmask: 255.255.255.0
    routers: 10.20.202.1
    dns_servers: 10.20.202.1
    ntp_servers: 10.20.202.1
    domain_name: s2-mgmt.network.lan
    hosts:
      # hosts
      # - name: node1 # static
      #   mac: ca:fe:ca:fe:21:0a
      #   ip: 10.20.202.1
      - name: node2
        mac: ca:fe:ca:fe:22:0a
        ip: 10.20.202.2
      - name: node3
        mac: ca:fe:ca:fe:23:0a
        ip: 10.20.202.3
```

### Deployment

```console
$ # site 1
$ ansible-playbook -i site1/hosts dhcpd.yml
$ # site 2
$ ansible-playbook -i site2/hosts dhcpd.yml
```

## Conclusion
