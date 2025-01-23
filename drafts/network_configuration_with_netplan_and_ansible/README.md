# Network Configuration with Netplan and Ansible

This document describes how you can configure Linux network settings with
Netplan and Ansible. It shows the Netplan configuration as well as the Ansible
role, playbook and configuration that are used to configure each host
automatically.

## Overview

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

#### Tasks

The tasks are defined as follows in the file `roles/netplan/tasks/main.yml`:

```yaml
---
# these tasks setup a network configuration with netplan

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

#### Templates

The template is defined as follows in the file
`roles/nftables/templates/netplan-network.yaml.j2`:

```jinja
network:
  {{ network | to_nice_yaml(indent=2,sort_keys=false) | indent(width=2) | trim }}
```

### Playbook

The playbook is defined as follows in the file `netplan.yml`:

```yaml
---
- name: Configure network using netplan
  hosts: netplan_hosts

  roles:
    - netplan
```

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

```console
$ # site 1
$ ansible-playbook -i site1/hosts netplan.yml
$ # site 2
$ ansible-playbook -i site2/hosts netplan.yml
```

## Conclusion

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
