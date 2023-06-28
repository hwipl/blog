# DNS with Bind and Ansible

DNS server for small networks with Bind 9, configured with Ansible.

## Overview

Assume scenario: one or more isolated networks, each with its own DNS server.
DNS server responsible for internal DNS names, forwards other requests to other
(external) DNS servers.

DNS Serer: Bind 9

DNS installed and configured with Ansible. Zone files configured automatically.
Support for subdomain aliases.

Limitations: only A records for now.

## DNS Configuration

Bind 9

Listen addresses

Forwarders

Access Control

Zones

## Ansible

Roles and Playbooks
