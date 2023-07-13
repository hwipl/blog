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

### Listen Addresses

The DNS servers listen on IP addresses to receive queries from DNS clients.

Each DNS server listens on two IPv4 addresses: the loopback address `127.0.0.1`
and the IP address of the server node's network interface (`10.20.1.1` in Site
1 and `10.20.2.1` in Site 2).  Listening on the loopback interface allows local
processes on the server node itself to query the DNS server. Listening on the
node's network interface allows other nodes in the network to query the DNS
server.

Each DNS server only listens on one IPv6 address: the loopback address `::1`.
Thus, the server only receives IPv6 queries from processes on the server node
itself and not from any other nodes in the network.

Note: Alternatively, each DNS server can receive DNS queries on all the
server's IPv4 addresses with the IPv4 listen address `any`. To allow receiving
IPv6 queries from other nodes in the network, the server's IPv6 addresses or
`any` can be added to IPv6 listen addresses.

### Forwarders

The DNS servers are recursive forwarders. They forward queries, except for
local domain names (see Zones) or cached records, to the other DNS servers
`10.1.1.1` and `10.2.2.2`.

### Access Control

The clients that are allowed to query the DNS servers are restricted with
access control lists.

Each DNS server accepts queries only from the configured `good clients`
addresses `localhost`, `localnets`, `10.20.0.0/16` and ignores other queries.
The special address `localhost` is the DNS server itself. The special address
`localnets` are the IP addresses of the networks the server is connected to.
The IPv4 prefix `10.20.0.0/16` specifies the IPv4 addresses in the respective
range.

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

### Zones

The DNS servers are responsible for local domain names and answer queries of
these names directly without forwarding them to other servers. These domain
names are configured in the zone files.

The local domain is `network.lan` with additional subdomains `s1.network.lan`,
`site1.network.lan`, `s2.network.lan`, and `site2.network.lan`. The subdomain
`s1.network.lan` is just an alias for `site1.network.lan` and contains the same
domain names.  Accordingly, `s2.network.lan` is an alias for
`site2.network.lan`.

The aliasing is realized with domain-specific `$INCLUDE` statements in the zone
file of the domain `network.lan`. The domain names are included from other
files. For the subdomains `site1.network.lan` and `s1.network.lan`, the same
file is included. Accordingly, `site2.network.lan` and `s1.network.lan` point
to the same file.

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

```
roles/bind/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── db-actual.j2
    ├── db-includes.j2
    ├── named.conf.local.j2
    └── named.conf.options.j2
```

roles/bind/handlers/main.yml:

```yaml
---
# handlers for bind

- name: Restart bind9
  become: true
  ansible.builtin.service:
    name: bind9
    state: restarted
```

roles/bind/tasks/main.yml:

```yaml
---
# these tasks install and configure bind9 name servers

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure bind9 is installed
  become: true
  ansible.builtin.apt:
    name: bind9
    state: present
- name: Create named.conf.options from template
  become: true
  ansible.builtin.template:
    src: named.conf.options.j2
    dest: "/etc/bind/named.conf.options"
    owner: root
    group: bind
    mode: '0644'
    backup: true
  notify:
    - Restart bind9
- name: Create named.conf.local from template
  become: true
  ansible.builtin.template:
    src: named.conf.local.j2
    dest: "/etc/bind/named.conf.local"
    owner: root
    group: bind
    mode: '0644'
  notify:
    - Restart bind9
- name: Create db files with includes from template
  become: true
  ansible.builtin.template:
    src: db-includes.j2
    dest: "/etc/bind/db.{{ item.name }}"
    owner: root
    group: bind
    mode: '0644'
  loop: "{{ zones }}"
  notify:
    - Restart bind9
- name: Create actual db files from template
  become: true
  ansible.builtin.template:
    src: db-actual.j2
    dest: "/etc/bind/{{ item.1.file }}"
    owner: root
    group: bind
    mode: '0644'
  loop: "{{ zones | subelements('dbs') }}"
  notify:
    - Restart bind9
```

roles/bind/templates/named.conf.options.j2:

```jinja
acl goodclients {
{% for client in goodclients %}
	{{ client }};
{% endfor %}
};

options {
	directory "/var/cache/bind";

	recursion yes;
	allow-query { goodclients; };

	forwarders {
{% for forwarder in forwarders %}
		{{ forwarder }};
{% endfor %}
	};
	forward only;
	max-ncache-ttl 1;

	dnssec-validation auto;

	auth-nxdomain no;
	listen-on {
{% for listen in bind_listen_on %}
		{{ listen }};
{% endfor %}
	};
	listen-on-v6 {
{% for listen in bind_listen_on_v6 %}
		{{ listen }};
{% endfor %}
	};
};
```

roles/bind/templates/named.conf.local.j2:

```jinja
{% for zone in zones %}
# {{ zone.description }}
zone "{{ zone.name }}" {
    type master;
    file "/etc/bind/db.{{ zone.name }}";
};
{% endfor %}
```

roles/bind/templates/db-includes.j2:

```jinja
$TTL    604800
@       IN      SOA     {{ item.name }}. {{ item.contact }} (
                              {{ item.serial}}         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      ns.{{ item.name }}.
@       IN      A       {{ item.nameserver_ip }}
@       IN      AAAA    ::1
ns      IN      A       {{ item.nameserver_ip }}

{% for include in item.includes %}
$INCLUDE "/etc/bind/{{ include.file }}" {{ include.name }}
{% endfor %}
```

roles/bind/templates/db-actual.j2:

```jinja
{% for a in item.1.a_records %}
{{ a.name }}	IN	A	{{ a.ip }}
{% endfor %}
```

bind.yml:

```yaml
---
- name: Configure dns servers
  hosts: dns_servers

  roles:
    - bind
```

hosts:

```ini
[dns_servers]
node1
```

host configuration on DNS server, host_vars/node1:

```yaml
---
# dns server configuration
bind_listen_on:
  - "127.0.0.1"
  - "10.20.1.1"
bind_listen_on_v6:
  - "::1"
```

group_vars/dns_servers:

```yaml
---
# dns server configuration

forwarders:
  - 10.1.1.1
  - 10.2.2.2

goodclients:
  - localhost
  - localnets
  - 10.20.0.0/16

zones:
  - name: network.lan
    description: network zone
    nameserver_ip: 10.20.1.1
    contact: root.network.lan.
    serial: 1
    includes:   # references to actual db files, allows aliases
      - name: site1.network.lan.
        file: db.network.lan-site1
      - name: s1.network.lan.   # alias for site1
        file: db.network.lan-site1
      - name: site2.network.lan.
        file: db.network.lan-site2
      - name: s2.network.lan.   # alias for site2
        file: db.network.lan-site2
      - name: network.lan.
        file: db.network.lan-all
    dbs:    # actual db files that contain host names
      - file: db.network.lan-site1
        a_records:
          - name: node1
            ip: 10.20.1.1
          - name: node2
            ip: 10.20.1.2
          - name: node3
            ip: 10.20.1.3
          - name: service1
            ip: 10.20.1.1
          - name: service2
            ip: 10.20.1.1
      - file: db.network.lan-site2
        a_records:
          - name: node1
            ip: 10.20.2.1
          - name: node2
            ip: 10.20.2.2
          - name: node3
            ip: 10.20.2.3
          - name: service1
            ip: 10.20.2.1
          - name: service2
            ip: 10.20.2.1
      - file: db.network.lan-all
        a_records:
          - name: service1
            ip: 10.20.1.1
          - name: service1
            ip: 10.20.2.1
          - name: service2
            ip: 10.20.1.1
          - name: service2
            ip: 10.20.2.1
```

Deployment:

```console
$ # site 1
$ ansible-playbook -i site1/hosts bind.yml
$ # site 2
$ ansible-playbook -i site2/hosts bind.yml
```
