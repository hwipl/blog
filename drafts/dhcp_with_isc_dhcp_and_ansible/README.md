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

The options `default-lease-time` and `max-lease-time` set the default and
maximum lease times to 86400 seconds for all subnets. The two `subnet` blocks
configure the two subnets. The `subnet` line specifies the IP address range
with the IP address and netmask.

The first subnet with IP `10.20.1.0` and netmask `255.255.255.0` contains the
configuration of the regular network interfaces. The second subnet with IP
`10.20.201.0` and netmask `255.255.255.0` contains the configuration of the
management interfaces.

The options `routers`, `domain-name-servers` and `ntp-servers` set the routers,
DNS servers, and NTP servers for each subnet. In the regular subnet they are
all set to IP `10.20.1.1`, in the management subnet to `10.20.201.1`. The
option `domain-name` specifies the domain names used in the subnets. In the
regular subnet it is set to `s1.network.lan`, in the management subnet to
`s1-mgmt.network.lan`.

The two `host` blocks contain the configuration of the nodes `node2` and
`node3` in the subnets. The settings `hardware ethernet` and `fixed-address`
specify the MAC address of the node and the fixed IP address allocated to the
node: for the regular interface of Node 2 they are set to IP `10.20.1.2` and
MAC `ca:fe:ca:fe:12:01`, for the management interface to IP `10.20.201.2` and
MAC `ca:fe:ca:fe:12:0a`; for Node 3 they are set to IP `10.20.1.3` and MAC
`ca:fe:ca:fe:13:01` as well as IP `10.20.201.3` and MAC `ca:fe:ca:fe:13:0a`.

The next listing shows the file `/etc/dhcp/dhcpd.conf` for Site 2:

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

This file differs from the previous one in the IP and MAC addresses. The
subnets use IPs `10.20.2.0` and `10.20.202.0`. Routers, DNS servers and NTP
servers are set to `10.20.2.1` and `10.20.202.1`. The domain names are
`s2.network.lan` and `s2-mgmt.network.lan`. The addresses of Node 2 are IP
`10.20.2.2` and MAC `ca:fe:ca:fe:22:01` as well as IP `10.20.202.2` and MAC
`ca:fe:ca:fe:22:0a`. For Node 3, they are IP `10.20.2.3` and MAC
`ca:fe:ca:fe:23:01` as well as IP `10.20.202.3` and MAC `ca:fe:ca:fe:23:0a`.

## Ansible

Ansible allows for automatic installation and configuration of the DHCP
servers. Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist
of tasks, templates and handlers. Tasks are the individual installation and
configuration steps. They use the [templates][templates] to generate
configuration files and trigger events that are handled by the
[handlers][handlers].

A role and a playbook are used to deploy the DHCP servers. The playbook assigns
the role to all nodes of a group defined in the Ansible [inventory][inventory].
The configuration of each DHCP server is derived from host and group variables
in the inventory. Each site uses a different inventory to allow for
site-specific configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

### Role

The Ansible role is called `dhcpd` and structured as shown in the listing
below:

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

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and two files for templates. The tasks use the template `dhcp.conf.j2` to
create the main configuration file including the subnets. The template
`isc-dhcp-server.j2` is used for the configuration of the network interfaces of
the DHCP servers.

#### Handlers

A handler is defined as follows in the file `roles/dhcpd/handlers/main.yml`:

```yaml
---
# handlers for dhcpd

- name: Restart dhcpd
  become: true
  ansible.builtin.service:
    name: isc-dhcp-server
    state: restarted
```

The handler is called `Restart dhcpd` and restarts the DHCP server with the
[service module][service] when it is triggered. It requires root privileges to
manipulate the state of the system services, so [become][become] is set to
`true` for [privilege escalation][privilege]. To restart the DHCP server, it
sets the system service `isc-dhcp-server` to state `restarted`.

#### Tasks

The tasks are defined as follows in the file `roles/dhcpd/tasks/main.yml`:

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

These tasks install the ISC DHCP server with the [apt module][apt], configure
it with templates and the [template module][template] and trigger the restart
event with [notify][notify] if changes are actually made by the tasks:

All tasks need root privileges to manipulate the system configuration. So,
[become][become] is set to `true`. The file owner of all configuration files is
set to `root`, group is set to `root` and privileges are set to `644`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs the ISC DHCP server with `apt` if it is not already
   installed.
3. The third task creates or updates the main configuration of the DHCP server
   in file `/etc/dhcp/dhcpd.conf` from the template `dhcp.conf.j2`. If the file
   is changed, the restart event is triggered.
4. The fourth task creates or updates the configuration of the network
   interfaces in the file `/etc/default/isc-dhcp-server` from the template
   `isc-dhcp-server.j2`. If a different version already exists, the task
   creates a backup of the existing file. If the file is changed, the restart
   event is triggered.

#### Templates

All templates in the role are defined as [Jinja2 templates][jinja2].

The template for the file `dhcp.conf` is defined as follows in the file
`roles/dhcpd/templates/dhcp.conf.j2`:

```jinja
default-lease-time {{ dhcpd_default_lease_time }};
max-lease-time {{ dhcpd_max_lease_time }};

{% for subnet in dhcpd_subnets %}
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

The template reflects the DHCP server and subnets configuration shown in the
DHCP configuration section above with parts dynamically generated based on the
Ansible configuration:

- The default and maximum lease times are set from the variables
  `dhcpd_default_lease_time` and `dhcpd_max_lease_time`.
- For each subnet configured in the Ansible list `dhcpd_subnets`, the template
  creates a `subnet` block. The IP address and subnet mask are taken from the
  subnet variables `ip` and `netmask`. The options for routers, DNS servers,
  NTP servers and domain names are taken from the subnet variables `routers`,
  `dns_servers`, `ntp servers` and `domain_name`.
- For each host in the Ansible list `hosts` within the subnet, a `host` block
  is created inside the `subnet` block. The name of the host is taken from the
  host variable `name`. The MAC and IP addresses are taken from the host
  variables `mac` and `ip`.

TODO: use variables for lease times?

The template for the file `/etc/default/isc-dhcp-server` is defined as follows
in the file `roles/dhcpd/templates/isc-dhcp-server.j2`:

```jinja
INTERFACESv4="{{ dhcpd_interfaces_v4 }}"
INTERFACESv6="{{ dhcpd_interfaces_v6 }}"
```

The template reflects the interface configuration shown in the DHCP
configuration section above with parts dynamically generated based on the
Ansible configuration: The network interfaces for DHCPv4 are taken from the
Ansible variable `dhcpd_interfaces_v4`. For DHCPv6, they are specified in the
variable `dhcpd_interfaces_v6`.

TODO: add variable for DHCPv6 interfaces?
TODO: change order of templates?

### Playbook

The playbook is defined as follows in the file `dhcpd.yml`:

```yaml
---
- name: Configure dhcp servers
  hosts: dhcp_servers

  roles:
    - dhcpd
```

The playbook assigns the role `dhcpd` that is described above to all hosts in
the group `dhcp_servers`. On execution, this playbook runs all the tasks of the
role on all the hosts in the group to install and configure the DHCP servers.

### Configuration

The configuration is derived from host and group files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use. Each site uses a different inventory and, thus, different host and group
files to allow for site-specific configurations as shown below.

#### Hosts

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[dhcp_servers]
node1
```

The hosts file of each site defines the group `dhcp_servers` and assigns the
node `node1` to it. Thus, `node1` is defined as DHCP server for the playbook.

The host-specific configuration of the DHCP servers is in the `host_vars` of
`node1` in each site. The configuration for the DHCP server in Site 1 and Site
2 is defined as follows in the files `site1/host_vars/node1` and
`site2/host_vars/node1`:

```yaml
# dhcp server configuration
dhcpd_interfaces_v4: "eth0 eth1"
dhcpd_interfaces_v6: ""
dhcpd_default_lease_time: 86400
dhcpd_max_lease_time: 86400
```

Both files contain the following configuration as described in the DHCP
configuration section above: The network interfaces for DHCPv4 are set to
`eth0` and `eth1`. No DHCPv6 interfaces are set. The default and max lease
times are both set to `86400`.

#### Groups

The site-specific DHCP configuration of Site 1 is defined as follows in the
file `site1/group_vars/dhcp_servers`:

```yaml
---
# dhcp server configuration

dhcpd_subnets:
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

The site-specific DHCP configuration of Site 2 is defined as follows in the
file `site2/group_vars/dhcp_servers`:

```yaml
---
# dhcp server configuration

dhcpd_subnets:
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

Both files set the configuration for the subnets of Site 1 and Site 2 as
described in the DHCP configuration section above. In each site, the list
`subnets` contains both subnets. Each subnet is configured with the following
variables:

- IP and netmask are defined in the variables `ip` and `netmask`
- Routers, DNS servers and NTP servers are defined in variables `routers`,
  `dns_servers` and `ntp_servers`
- Domain name is set in `domain_name`
- Hosts are configured in the list `hosts` and each entry consists of the
  host's settings for the name, MAC address and IP address in the variables
  `name`, `mac` and `ip`.

### Deployment

The DHCP servers can be installed and configured with the Ansible role,
configuration and playbook described above. You can use the following
[ansible-playbook][ansible-playbook] commands:

```console
$ # site 1
$ ansible-playbook -i site1/hosts dhcpd.yml
$ # site 2
$ ansible-playbook -i site2/hosts dhcpd.yml
```

Both `ansible-playbook` commands run the playbook `dhcpd.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
first command installs and configures all DHCP servers in Site 1 and the second
command in Site 2. After successful execution of the commands above, the DHCP
servers should be configured and running.

## Conclusion

This document describes how you can automatically install and configure ISC
DHCP servers on Ubuntu 22.04 LTS with Ansible. After presenting an example
network and DHCP configuration, an Ansible Role is shown that contains the
installation and configuration tasks as well as templates. The tasks use the
templates together with Ansible host and group variables to create the ISC DHCP
configuration. Finally, an Ansible Playbook is used with the `ansible-playbook`
command to run the tasks and, thus, install and configure all servers. All
sections contain configuration or code examples you can use as a basis for your
own setup. Also, you can find links to the code and configuration examples in
the appendix below.

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/dhcpd/roles/dhcpd)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/dhcpd/dhcpd.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/dhcpd/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/dhcpd/examples/site2)

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
