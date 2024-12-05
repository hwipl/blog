# Firewall with nftables and Ansible

This document describes how you can deploy Linux firewalls with nftables and
Ansible. It shows the nftables configuration as well as the Ansible role,
playbook and configuration that are used to install and configure each firewall
automatically.

Note: you need to be careful to not lock yourself out of remote hosts or
interfere with running services when you make changes to the firewall
configuration of hosts.

## Overview

The firewalls in this document use nftables and run on Ubuntu 22.04 LTS.
Nftables is installed and configured automatically with Ansible. The firewall
configuration assumes the network in the following figure:

```
                             Other Networks
                   _________________|_________________
                  |                                   |
..................|...................................|..................
: Site 1          |                 :                 |          Site 2 :
: 10.20.1.0/24    |                 :                 |    10.20.2.0/24 :
:                 |                 :                 |                 :
:          +----[ext0]---+          :          +----[ext0]---+          :
:          |             |          :          |             |          :
:          | Node 1      |          :          | Node 1      |          :
:          | Router      |          :          | Router      |          :
:          | NAT         |          :          | NAT         |          :
:          | Filtering   |          :          | Filtering   |          :
:          | 10.20.1.1   |          :          | 10.20.2.1   |          :
:          |             |          :          |             |          :
:          +----[int0]---+          :          +----[int0]---+          :
:          _______|_______          :          _______|_______          :
:         |               |         :         |               |         :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:  | Node 2      | | Node 3      |  :  | Node 2      | | Node 3      |  :
:  | Client      | | Client      |  :  | Client      | | Client      |  :
:  | Filtering   | | Filtering   |  :  | Filtering   | | Filtering   |  :
:  | 10.20.1.2   | | 10.20.1.3   |  :  | 10.20.2.2   | | 10.20.2.3   |  :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:...................................:...................................:
                        Network: 10.20.0.0/16
```

The example network consists of the two sites `Site 1` and `Site 2`. The sites
use private IP addresses: `10.20.1.0/24` in Site 1, `10.20.2.0/24` in Site 2.
Each site contains three nodes. `Node 1` is a router with two network
interfaces. The network interface `int0` links the node to the other nodes in
the same site. The interface `ext0` links the node to other networks. The node
interconnects the site with the other networks including the other site. It
performs Network Address Translation (NAT) to enable access from the nodes in
the site with their private IP addresses to other networks. It filters packets
to protect the nodes in the site from unwanted traffic from other networks. The
node performs the NAT and packet filtering with a nftables firewall. It only
allows traffic that was initiated by nodes inside the site and blocks other
traffic. `Node 2` and `Node 3` are client nodes. Each runs a nftables firewall
that filters packets to protect the node from unwanted traffic from both other
networks as well as other nodes inside the same site: it allows only traffic
that was initiated by the node itself and blocks other traffic.

## Firewall Configuration

The configuration of the firewalls is described in this section. The firewalls
in the two sites are configured in the file `/etc/nftables.conf`. The
configuration on the routers (Node 1) differs from the configuration on the
client nodes (Node 2 and Node 3).

### Router Nodes

The configuration of the router nodes is shown in the following listing:

```
#!/usr/sbin/nft -f

table inet fw_router
delete table inet fw_router
table inet fw_router {
    chain input_internal {
        # filter rules for incoming traffic on internal network interface

        # allow incoming ICMPv4 traffic
        ip protocol icmp accept comment "allow icmp"

        # allow incoming ICMPv6 traffic
        meta l4proto ipv6-icmp accept comment "allow icmp v6"

        # allow incoming SSH connections
        tcp dport ssh accept comment "allow sshd"
    }

    chain input {
        # filter rules for incoming traffic, drop everything by default
        type filter hook input priority 0; policy drop;

        # allow incoming traffic that was initiated by this node
        ct state {established, related} accept comment "allow tracked connections"

        # allow incoming traffic on loopback interface
        iif lo accept comment "allow from loopback"

        # check incoming traffic on internal network interface
        iifname "int0" jump input_internal
    }

    chain forward {
        # filter rules for forwarding traffic, drop everything by default
        type filter hook forward priority 0; policy drop;

        # allow traffic that was initiated from the internal network
        ct state {established, related} accept comment "allow tracked connections"

        # allow traffic from internal network
        iifname "int0" accept comment "allow from internal interface"
    }

    chain postrouting {
        # NAT rules for outgoing traffic
        type nat hook postrouting priority srcnat; policy accept;

        # NAT outgoing traffic on external network interface
        oifname "ext0" counter masquerade comment "masquerade outgoing"
    }
}
```

The configuration is in the [table][tables] `fw_router` that is added to any
pre-existing firewall configuration. The name of the table should differ from
the names that are usually used by software like, e.g., [docker][docker] or
[libvirt][libvirt] so (re)starting the firewall configuration does not remove
existing firewall [rules][rules]. Note that the rules themselves still can
interfere with each-other. For example, packets that are accepted by existing
firewall rules can still be dropped in this table.

The table contains the rules for NAT in the [chain][chains] `postrouting`. It
contains the NAT rules for outgoing traffic: its type is `nat` and it is
attached to the hook `postrouting`. It contains a single rule that changes
(`masquerade`) the source address of outgoing traffic on the external network
interface (`oifname "ext0"`) to match the address of that interface.

The table contains the rules for filtering in the three chains `input`,
`input_internal` and `forward`. The chain `input` contains the rules for
incoming traffic that is addressed to the node itself: its type is `filter` and
it is attached to the hook `input`. By default this chain drops all traffic
that goes through this chain and is not explicitly accepted by a rule in the
chain (`policy drop`). It contains the following rules:

1. The first rule allows all traffic that is already tracked by connection
   tracking (`ct`) and belongs to or is related to an existing connection (`ct
   state {established, related}`).
2. The second rule allows all incoming traffic on the loopback interface (`iif
   lo`).
3. The third rule checks incoming traffic on the internal network interface
   (`iifname "int0"`) with the rules in the additional chain `input_internal`.

The chain `input_internal` contains the following rules for incoming traffic on
the internal network interface:

1. The first rule allows incoming ICMPv4 traffic (`ip protocol icmp`)
2. The second rule allows incoming ICMPv6 traffic (`meta l4proto ipv6-icmp`)
3. The third rule allows incoming SSH connections (`tcp dport ssh`)

The chain `forward` contains the rules for traffic that is forwarded by this
node: its type is `filter` and it is attached to the hook `forward`. By default
this chain drops all traffic that goes through it without being accepted by a
rule (`policy drop`). It contains the following rules:

1. The first rule allows all traffic that is already tracked by connection
   tracking (`ct`) and belongs to or is related to an existing connection (`ct
   state {established, related}`).
2. The second rule allows all traffic coming from the internal network
   interface (`iifname "int0"`) to be forwarded anywhere.

### Client Nodes

The configuration of the client nodes is shown in the following listing:

```
#!/usr/bin/nft -f

table inet fw_client
delete table inet fw_client
table inet fw_client {
    chain input {
        # filter rules for incoming traffic, drop everything by default
        type filter hook input priority filter; policy drop;

        # allow incoming traffic that was initiated by this node
        ct state {established, related} accept comment "allow tracked connections"

        # allow incoming traffic on loopback interface
        iif lo accept comment "allow from loopback"

        # allow incoming ICMPv4 traffic
        ip protocol icmp accept comment "allow icmp"

        # allow incoming ICMPv6 traffic
        meta l4proto ipv6-icmp accept comment "allow icmp v6"

        # allow incoming SSH connections
        tcp dport ssh accept comment "allow sshd"
    }
}
```

Similar to the router configuration above, the configuration is in a
[table][tables] called `fw_client` that is added to the existing firewall
configuration. The name of the table should not interfere with existing tables
so (re)starting this firewall configuration does not delete any existing
[rules][rules].

The table contains the rules for filtering in the chain `input`. It contains
the rules for incoming traffic that is addressed to the node itself: the
[chain][chains] is of type `filter` and attached to the hook `input`. By
default this chain drops all traffic that goes through it and is not explicitly
accepted by a rule (`policy drop`). It contains the following rules:

1. The first rule allows all traffic that is already tracked by connection
   tracking (`ct`) and belongs to or is related to an existing connection (`ct
   state {established, related}`).
2. The second rule allows all incoming traffic on the loopback interface (`iif
   lo`).
3. The third rule allows incoming ICMPv4 traffic (`ip protocol icmp`)
4. The forth rule allows incoming ICMPv6 traffic (`meta l4proto ipv6-icmp`)
5. The fifth rule allows incoming SSH connections (`tcp dport ssh`)

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
    ├── nftables-client.conf.j2
    └── nftables-router.conf.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and two template files. The tasks can use the templates
`nftables-client.conf.j2` and `nftables-router.conf.j2` to create the
configuration file for the firewalls. Using other templates, e.g., in the
Ansible inventory is also possible.

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

```yaml
---
# these tasks setup nftables

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure nftables is installed
  become: true
  ansible.builtin.apt:
    name: nftables
    state: present
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

The templates for the firewall configuration file are defined as [Jinja2
template][jinja2]. They are for the configuration of the router and client
nodes. Additionally, the tasks above allow the specification of alternative
templates for individual nodes in the configuration (see the Configuration
section below).

##### Router Nodes

The template for the router configuration is defined as follows in the file
`roles/nftables/templates/nftables-router.conf.j2`:

```jinja
#!/usr/sbin/nft -f

table inet {{ nftables_table }}
delete table inet {{ nftables_table }}
table inet {{ nftables_table }} {
    chain input_internal {
        # filter rules for incoming traffic on internal network interface

        # allow incoming ICMPv4 traffic
        ip protocol icmp accept comment "allow icmp"

        # allow incoming ICMPv6 traffic
        meta l4proto ipv6-icmp accept comment "allow icmp v6"

        # allow incoming SSH connections
        tcp dport ssh accept comment "allow sshd"
    }

    chain input {
        # filter rules for incoming traffic, drop everything by default
        type filter hook input priority 0; policy drop;

        # allow incoming traffic that was initiated by this node
        ct state {established, related} accept comment "allow tracked connections"

        # allow incoming traffic on loopback interface
        iif lo accept comment "allow from loopback"

        # check incoming traffic on internal network interface
        iifname {{ nftables_int_ifs }} jump input_internal
    }

    chain forward {
        # filter rules for forwarding traffic, drop everything by default
        type filter hook forward priority 0; policy drop;

        # allow traffic that was initiated from the internal network
        ct state {established, related} accept comment "allow tracked connections"

        # allow traffic from internal network
        iifname {{ nftables_int_ifs }} accept comment "allow from internal interface"
    }

    chain postrouting {
        # NAT rules for outgoing traffic
        type nat hook postrouting priority srcnat; policy accept;

        # NAT outgoing traffic on external network interface
        oifname {{ nftables_ext_ifs }} counter masquerade comment "masquerade outgoing"
    }
}
```

The template reflects the configuration of the router nodes shown in the
Firewall Configuration section above with parts dynamically generated based on
the Ansible configuration: The name of the table is taken from the Ansible
variable `nftables_table`. The names of the internal network interfaces are
taken from the variable `nftables_int_ifs`. The names of the external network
interfaces are taken from the variable `nftables_ext_ifs`. Both variables can
be set to the name of a single interface like `ext0` or to multiple interface
names like `{int0, int1}`.

##### Client Nodes

The template for the client configuration is defined as follows in the file
`roles/nftables/templates/nftables-client.conf.j2`:

```jinja
#!/usr/bin/nft -f

table inet {{ nftables_table }}
delete table inet {{ nftables_table }}
table inet {{ nftables_table }} {
    chain input {
        # filter rules for incoming traffic, drop everything by default
        type filter hook input priority filter; policy drop;

        # allow incoming traffic that was initiated by this node
        ct state {established, related} accept comment "allow tracked connections"

        # allow incoming traffic on loopback interface
        iif lo accept comment "allow from loopback"

        # allow incoming ICMPv4 traffic
        ip protocol icmp accept comment "allow icmp"

        # allow incoming ICMPv6 traffic
        meta l4proto ipv6-icmp accept comment "allow icmp v6"

        # allow incoming SSH connections
        tcp dport ssh accept comment "allow sshd"
    }
}
```

The template reflects the configuration of the client nodes shown in the
Firewall Configuration section above with parts dynamically generated based on
the Ansible configuration: The name of the table is taken from the Ansible
variable `nftables_table`.

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

The configuration is derived from host and group files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use. Each site uses a different inventory and, thus, different host and group
files to allow for site-specific configurations as shown below.

#### Hosts

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[nftables_hosts]
node1
node2
node3
```

The hosts file of each site defines the group `nftables_hosts` and assigns
Nodes 1, 2 and 3 (`node1`, `node2` and `node3`) to it. Thus, the three nodes
are defined as firewall hosts for the playbook.

The host-specific configuration of the firewall on Node 1 is in the `host_vars`
of `node1` in each site. The configuration for Site 1 and Site 2 is defined as
follows in the files `site1/host_vars/node1` and `site2/host_vars/node1`:

```yaml
---
# nftables configuration
nftables_conf: "nftables-router.conf.j2"
nftables_table: "fw_router"
nftables_int_ifs: "int0"
nftables_ext_ifs: "ext0"
```

Both files set the firewall configuration of Node 1 to the router configuration
described in the Firewall Configuration section above.

The variable `nftables_conf` sets the template of the configuration file to
`nftables-router.conf.j2`. The variable `nftables_table` sets the name of the
table to `fw_router`. The variable `nftables_int_ifs` sets the internal network
interfaces to the interface `int0`. The variable `nftables_ext_ifs` sets the
external network interfaces to the interface `ext0`.

As mentioned earlier, the tasks, or more specifically the `nftables_conf`
variable, allow the specification of alternative firewall configurations other
than the two templates in the Ansible role for individual nodes. For example,
a different template could be set for a node in its `host_vars` as follows:

```yaml
---
# nftables configuration
nftables_conf: "{{ inventory_dir }}/nftables/nftables-other.conf.j2"
```

Here, the nftables configuration file is set to the template file
`nftables-other.conf.j2` in the subdirectory `nftables` of the Ansible
inventory (`{{ inventory_dir }}`): In Site 1, this template file is
`site1/nftables/nftables-other.conf.j2` and in Site 2 it is
`site2/nftables/nftables-other.conf.j2`.

#### Groups

The site-specific configuration of the firewall on Node 2 and 3 in Site 1 and 2
is defined as follows in the in the file `site1/group_vars/nftables_hosts` and
`site2/group_vars/nftables_hosts`:

```yaml
---
# nftables configuration
nftables_conf: "nftables-client.conf.j2"
nftables_table: "fw_client"
```

Both files set the firewall configuration of Node 2 and 3 in both sites to the
client node configuration described in the Firewall Configuration section
above.

The variable `nftables_conf` sets the template of the configuration file to
`nftables-client.conf.j2`. The variable `nftables_table` sets the name of the
table to `fw_client`.

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

This document describes how you can automatically install and configure
nftables firewalls on Ubuntu 22.04 LTS with Ansible. After presenting an
example network and firewall configuration, an Ansible Role is shown that
contains the installation and configuration tasks as well as templates. The
tasks use the templates together with Ansible variables to create the nftables
configuration. Finally, an Ansible Playbook is used with the `ansible-playbook`
command to run the tasks and, thus, install and configure all firewalls. All
sections contain configuration or code examples you can use as a basis for your
own setup. Also, you can find links to the code and configuration examples in
the appendix below.

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/nftables/roles/nftables)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/nftables/nftables.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/nftables/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/nftables/examples/site2)

[tables]: https://wiki.nftables.org/wiki-nftables/index.php/Configuring_tables
[chains]: https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains
[rules]: https://wiki.nftables.org/wiki-nftables/index.php/Simple_rule_management
[docker]: https://www.docker.com
[libvirt]: https://libvirt.org
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
