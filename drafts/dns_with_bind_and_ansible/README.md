# DNS with Bind and Ansible

DNS server for small networks with Bind 9, configured with Ansible.

## Overview

Assume scenario: one or more isolated networks, each with its own DNS server.
DNS server responsible for internal DNS names, forwards other requests to other
(external) DNS servers.

OS: Ubuntu 22.04 LTS

DNS Server: ISC Bind 9

DNS installed and configured with Ansible. Zone files configured automatically.
Support for subdomain aliases.

Limitations: only A records for now. only IPv4 for now.

## DNS Configuration

Bind 9

Listen addresses

Forwarders

Access Control

Zones

/etc/bind/named.conf.options:

```
acl goodclients {
        localhost;
        localnets;
        10.20.0.0/16;
};

options {
        directory "/var/cache/bind";

        recursion yes;
        allow-query { goodclients; };

        forwarders {
                10.1.1.1;
                10.2.2.2;
        };
        forward only;
        max-ncache-ttl 1;

        dnssec-validation auto;

        auth-nxdomain no;
        listen-on {
                127.0.0.1;
                10.20.1.1;
        };
        listen-on-v6 {
                ::1;
        };
};
```

## Ansible

Roles and Playbooks
