# Network Configuration with Netplan and Ansible

This document describes how you can configure Linux network settings with
Netplan and Ansible. It shows the Netplan configuration as well as the Ansible
role, playbook and configuration that are used to configure each host
automatically.

## Overview

## Netplan Configuration

## Ansible

### Role

#### Handlers

```yaml
---
# netplan handlers

- name: Apply netplan configuration
  become: true
  ansible.builtin.command: netplan apply
```

#### Tasks

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

```jinja
network:
  {{ network | to_nice_yaml(indent=2,sort_keys=false) | indent(width=2) | trim }}
```

### Playbook

```yaml
---
- name: Configure network using netplan
  hosts: netplan_hosts

  roles:
    - netplan
```

### Configuration

### Deployment

## Conclusion

## Appendix: Code
