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
examined in this document. `Node 1` connects the site to other networks, e.g.,
the Internet. `Site 2` is, except for addresses, identical to `Site 1`. To save
space, it is not examined further in this document. Also, the three local
networks just affect the routing configuration and are not discussed in more
detail. The IP addresses in `Site 1` and `Site 2` are `10.20.1.0/24` and
`10.20.2.0/24`. The IP addresses in the three local networks are
`10.21.0.0/16`, `10.22.0.0/16` and `10.23.0.0/16`.

The settings of the network devices on all nodes are shown in the following
table:

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

On Node 1, the device `ext0` with the MAC address `ca:fe:ca:fe:11:01` is not a
member of a software bridge. The device `int0` with the MAC address
`ca:fe:ca:fe:11:03` is the only member of the software bridge `int-br0`. The
MAC address of the software bridge (device `int-br0`) is the same as its member
device. On Node 2, the device `int0` with the MAC address `ca:fe:ca:fe:12:01`
is the only member of the software bridge `int-br0`. The MAC address of the
software bridge (device `int-br0`) is the same as its member device. On Node 3,
the device `int0` with the MAC address `ca:fe:ca:fe:13:01` is not a member of a
software bridge. The MTU of all network devices on all nodes is set to `1500`.

The IPv4 configuration of the network devices on all nodes is shown in the
following table:

| Node   | Device   | IPv4 Address | IPv4 Routes           |
|--------|----------|--------------|-----------------------|
| Node 1 | ext0     | DHCP         | DHCP                  |
|        | int0     | None         | None                  |
|        | int-br0  | 10.20.1.1/24 | None                  |
|        |          |              |                       |
| Node 2 | int0     | None         | None                  |
|        | int-br0  | 10.20.1.2/24 | default via 10.20.1.1 |
|        |          |              |                       |
| Node 3 | int0     | 10.20.1.3/24 | default via 10.20.1.1 |

On Node 1, the device `ext0` retrieves its IPv4 address and routes via DHCP. No
addresses and no routes are configured on device `int0`, because it's just a
member of the bridge `int-br0`. The bridge device `int-br0` has the address
`10.20.1.1/24` and no additional routes. On Node 2, there is also no IP address
or route on device `int0`, because it's a member of the bridge `int-br0`. The
bridge device `int-br0` has IP address `10.20.1.2/24` and the default route is
set to `10.20.1.1` (i.e. Node 1). On Node 3, device `int0` has IP address
`10.20.1.3/24` and the default route is also set to `10.20.1.1` (Node 1).

## Netplan Configuration

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
`netplan-network.yaml.j2` to create the configuration file for netplan.

#### Handlers

A handler is defined as follows in the file `roles/netplan/handlers/main.yml`:

```yaml
---
# netplan handlers

- name: Apply netplan configuration
  become: true
  ansible.builtin.command: netplan apply
```

The handler is called `Apply netplan configuration` and applies the network
configuration with the [command module][command] when it is triggered. It
requires root privileges to manipulate the system configuration, so
[become][become] is set to `true` for [privilege escalation][privilege]. It
calls the command `netplan apply` to apply the network configuration.

#### Tasks

The tasks are defined as follows in the file `roles/netplan/tasks/main.yml`:

```yaml
---
# these tasks setup a network configuration with netplan

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure netplan is installed
  become: true
  ansible.builtin.apt:
    name: netplan.io
    state: present
- name: Create netplan configuration from template
  become: true
  ansible.builtin.template:
    src: netplan-network.yaml.j2
    dest: "/etc/netplan/{{ netplan_config_file }}"
    owner: root
    group: root
    mode: '0644'
  notify:
    - Apply netplan configuration
```

These tasks make sure netplan is installed with the [apt module][apt],
configure it with the template and the [template module][template] and trigger
the event to apply the network configuration with [notify][notify] if the
configuration file is changed. All tasks need root privileges to manipulate the
system configuration. So, [become][become] is set to `true`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs netplan with `apt` if it is not already installed.
   In Ubuntu, it should already be installed by default.
3. The third task creates or updates the network configuration in the directory
   `/etc/netplan` from the template `netplan-network.yaml.j2`. The file name is
   read from the Ansible variable `netplan_config_file`. The file owner and
   group are set to `root`. File permissions are set to `644`. If the
   configuration changed, the task triggers the event to apply the network
   configuration.

#### Templates

The template for the network configuration file is defined as [Jinja2
template][jinja2]. It is defined as follows in the file
`roles/nftables/templates/netplan-network.yaml.j2`:

```jinja
network:
  {{ network | to_nice_yaml(indent=2,sort_keys=false) | indent(width=2) | trim }}
```

The template creates a netplan configuration as shown in the Network
Configuration section above from the Ansible configuration. To this end, it
converts the network configuration in the Ansible variable `network` to a yaml
file that netplan can use. So, the content of the `network` variable must be a
valid netplan configuration (see Configuration below).

### Playbook

The playbook is defined as follows in the file `netplan.yml`:

```yaml
---
- name: Configure network using netplan
  hosts: netplan_hosts

  roles:
    - netplan
```

The playbook assigns the role `netplan` that is described above to all hosts in
the group `netplan_hosts`. On execution, this playbook runs all the tasks of
the role on all the hosts in the group to configure the network settings.

### Configuration

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[netplan_hosts]
node1
node2
node3
```

Node 1:

The host-specific configuration of Node 1 is in the `host_vars` of `node1` in
each site. The configuration for Site 1 defined as follows in the file
`site1/host_vars/node1`:

```yaml
# netplan network configuration
netplan_config_file: 90-node1-site1-network.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ext0:
      match:
        macaddress: "ca:fe:ca:fe:11:01"
      set-name: ext0
      dhcp4: true
    mgmt0:
      match:
        macaddress: "ca:fe:ca:fe:11:02"
      set-name: mgmt0
      dhcp4: false
    int0:
      match:
        macaddress: "ca:fe:ca:fe:11:03"
      set-name: int0
      dhcp4: false
    int1:
      match:
        macaddress: "ca:fe:ca:fe:11:04"
      set-name: int1
      dhcp4: false
  bridges:
    mgmt-br0:
      interfaces:
        - mgmt0
      parameters:
        forward-delay: 15
        stp: false
      macaddress: "ca:fe:ca:fe:11:02"
      mtu: 1500
      dhcp4: false
      addresses:
        - 10.20.201.1/24
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
        # other cluster traffic
        - to: 10.20.0.0/16
          via: 10.20.1.201
        # other user traffic
        - to: 10.21.0.0/16
          via: 10.20.1.201
        - to: 10.22.0.0/16
          via: 10.20.1.201
        - to: 10.23.0.0/16
          via: 10.20.1.201
```

Node 2:

The host-specific configuration of Node 2 is in the `host_vars` of `node2` in
each site. The configuration for Site 1 defined as follows in the file
`site1/host_vars/node2`:

```yaml
# netplan network configuration
netplan_config_file: 90-node2-site1-network.yaml
network:
    bridges:
        int-br0:
            interfaces:
            - int0
            parameters:
                forward-delay: 15
                stp: false
            macaddress: ca:fe:ca:fe:12:01
            mtu: 1500
            addresses:
            - 10.20.1.2/24
            routes:
              - to: default
                via: 10.20.1.1
              # vpn traffic
              - to: 10.20.21.0/24
                via: 10.20.1.1
              # other cluster traffic
              - to: 10.20.0.0/16
                via: 10.20.1.201
              # other user traffic
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
                - s1.cluster.lan
                - cluster.lan
        mgmt-br0:
            interfaces:
            - mgmt0
            parameters:
                forward-delay: 15
                stp: false
            macaddress: ca:fe:ca:fe:12:02
            mtu: 1500
            addresses:
            - 10.20.201.2/24
            nameservers:
                addresses:
                - 10.20.201.10
                search:
                - maas
    ethernets:
        int0:
            match:
                macaddress: ca:fe:ca:fe:12:01
            set-name: int0
            mtu: 1500
        mgmt0:
            match:
                macaddress: ca:fe:ca:fe:12:02
            set-name: mgmt0
            mtu: 1500
    version: 2
```

Node 3:

The host-specific configuration of Node 3 is in the `host_vars` of `node3` in
each site. The configuration for Site 1 is defined as follows in the file
`site1/host_vars/node3`:

```yaml
# netplan network configuration
netplan_config_file: 90-node3-site1-network.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # interface connected to management network
    mgmt0:
      match:
        macaddress: "ca:fe:ca:fe:13:01"
      set-name: mgmt0
      dhcp4: false
      addresses:
        - 10.20.201.3/24
    int0:
      match:
        macaddress: "ca:fe:ca:fe:13:02"
      set-name: int0
      dhcp4: false
      addresses:
        - 10.20.1.3/24
      nameservers:
        search:
          - s1.cluster.lan
          - cluster.lan
        addresses:
          - 10.20.1.1
      routes:
        - to: default
          via: 10.20.1.1
        # vpn traffic
        - to: 10.20.21.0/24
          via: 10.20.1.1
        # other cluster traffic
        - to: 10.20.0.0/16
          via: 10.20.1.201
        # other user traffic
        - to: 10.21.0.0/16
          via: 10.20.1.201
        - to: 10.22.0.0/16
          via: 10.20.1.201
        - to: 10.23.0.0/16
          via: 10.20.1.201
```

### Deployment

The network configuration can be deployed with the Ansible role, configuration
and playbook described above. You can use the following
[ansible-playbook][ansible-playbook] commands:

```console
$ # site 1
$ ansible-playbook -i site1/hosts netplan.yml
$ # site 2
$ ansible-playbook -i site2/hosts netplan.yml
```

Both `ansible-playbook` commands run the playbook `netplan.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
first command configures all hosts in Site 1 and the second command in Site 2.
After successful execution of the commands above, the network settings should
be configured and active.

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

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/netplan/roles/netplan)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/netplan/netplan.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/netplan/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/netplan/examples/site2)

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
