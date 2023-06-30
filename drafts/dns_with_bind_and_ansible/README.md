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

/etc/bind/named.conf.local:

```
# network zone
zone "network.lan" {
    type master;
    file "/etc/bind/db.network.lan";
};
```

/etc/bind/db.network.lan:

```
$TTL    604800
@       IN      SOA     network.lan. root.network.lan. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns.network.lan.
@       IN      A       10.20.1.1
@       IN      AAAA    ::1
ns      IN      A       10.20.1.1

$INCLUDE "/etc/bind/db.network.lan-site1" site1.network.lan.
$INCLUDE "/etc/bind/db.network.lan-site1" s1.network.lan.
$INCLUDE "/etc/bind/db.network.lan-site2" site2.network.lan.
$INCLUDE "/etc/bind/db.network.lan-site2" s2.network.lan.
$INCLUDE "/etc/bind/db.network.lan-all" network.lan.
```

/etc/bind/db.network.lan-site1:

```
node1       IN    A    10.20.1.1
node2       IN    A    10.20.1.2
node3       IN    A    10.20.1.3

service1    IN    A    10.20.1.1
service2    IN    A    10.20.1.1
```

/etc/bind/db.network.lan-site2:

```
node1       IN    A    10.20.2.1
node2       IN    A    10.20.2.2
node3       IN    A    10.20.2.3

service1    IN    A    10.20.2.1
service2    IN    A    10.20.2.1
```

/etc/bind/db.network.lan-all:

```
service1    IN    A    10.20.1.1
service1    IN    A    10.20.2.1
service2    IN    A    10.20.1.1
service2    IN    A    10.20.2.1
```

## Ansible

Roles and Playbooks
