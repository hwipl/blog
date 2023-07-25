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

The configuration of Site 1 contains five `A` records: the three nodes `node1`
to `node3` and the two services `service1` and `service2`. The `A` records of
the nodes resolve to the addresses of the respective nodes in the Site 1
network. The `A` records of the services both resolve to the address of
`node1`. The configuration of Site 2 only differs in the addresses of the
nodes: the addresses of the respective nodes inside the Site 2 network.

In addition to the two site-specific subdomains above, there are domain names
in the domain `network.lan`: the two services `service1` and `service2`. Like
the services in the two sites, they resolve to the address of `node1`. But the
DNS servers in Site 1 and Site 2 use different configurations: in Site 1 the
domain names resolve to the node in Site 1 and in Site 2 to the node in Site 2.
This way, clients in one site can use these domain names to access the services
in their respective site without having to know in which site they are.

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

/etc/bind/db.network.lan-all in Site 1:

```
service1    IN    A    10.20.1.1
service2    IN    A    10.20.1.1
```

/etc/bind/db.network.lan-all in Site 2:

```
service1    IN    A    10.20.2.1
service2    IN    A    10.20.2.1
```

## Ansible

Ansible allows for automatic installation and configuration of the DNS servers.
Ansible uses roles and playbooks. Roles consist of tasks, templates and
handlers. Tasks are the individual installation and configuration steps. They
use the templates to generate configuration files and trigger events that are
handled by the handlers.

Roles and Playbooks

### Role

The Ansible role is called `bind` and structured as shown in the listing below:

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

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and four files for templates. The tasks use `db-includes.j2` to create
the base zone file that includes the other files. The other files are created
with `db-actual.j2`. The template `named.conf.local.j2` is used for the DNS
server configuration for the zone and `named.conf.options.j2` is used for the
DNS server options.

#### Handlers

A handler is defined as follows in the file `roles/bind/handlers/main.yml`:

```yaml
---
# handlers for bind

- name: Restart bind9
  become: true
  ansible.builtin.service:
    name: bind9
    state: restarted
```

The handler is called `Restart bind9` and restarts the DNS server when it is
triggered. It requires root privileges to manipulate the state of the system
services, so `become` is set to `true`. To restart the DNS server, it sets the
system service `bind9` to state `restarted`.

#### Tasks

The tasks are defined as follows in the file `roles/bind/tasks/main.yml`:

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

These tasks install the bind9 DNS server, configure it with templates and
trigger the restart event if changes are actually made by the tasks:

All tasks need root privileges to manipulate the system configuration. So,
`become` is set to `true`.  All configuration files are stored in the directory
`/etc/bind`. File owner is set to `root`, group is set to `bind` and privileges
are set to `644`.

The first task updates the `apt` cache if it is older than one hour to make
sure the following installation task can run with up-to-date package sources.

The second task installs the bind9 DNS server with `apt` if it is not already
installed.

The third task creates or updates the DNS server options file
`named.conf.options` from the template `named.conf.options.j2`. If a different
version already exists, the task creates a backup of the existing file. If the
file is changed, the restart event is triggered.

The following three tasks create the local DNS zones configuration and trigger
the restart event if any files changed.

The fourth task creates or updates the local zone configuration file
`named.conf.local` from the template `named.conf.local.j2`.

The fifth task creates or updates the base zone files that include the actual
domain name files.  For each zone, a base file is created using the template
`db-includes.j2`. The file name consists of the prefix `db.` and the name of
the zone as defined in the configuration.

The sixth task creates or updates the files that contain the actual domain
names. For each such file in the configuration, a file is created using the
template `db-actual.j2`. The file name is also defined in the configuration.

#### Templates

The template for the file `named.conf.options` is defined as follows in the
file `roles/bind/templates/named.conf.options.j2`:

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

The template reflects the DNS server options shown in the DNS configuration
section above with parts dynamically generated based on the Ansible
configuration:

For each client configured in the list of `good clients`, the template creates
an entry in the access control list (ACL) `good clients`. This ACL is used in
`allow-query` in the `options` to allow these clients to query the DNS server.

For each DNS server configured in the list of forwarders, the template creates
an entry in the `forwarders`.

For each IPv4 address configured in the list `bind listen on`, a listen address
is created in `listen-on`. For each IPv6 address configured in the list `bind
listen on v6`, a listen address is created in `listen-on-v6`.

The template for the file `named.conf.local` is defined as follows in the file
`roles/bind/templates/named.conf.local.j2`:

```jinja
{% for zone in zones %}
# {{ zone.description }}
zone "{{ zone.name }}" {
    type master;
    file "/etc/bind/db.{{ zone.name }}";
};
{% endfor %}
```

The template reflects the DNS zone configuration shown in the DNS configuration
section above with parts dynamically generated based on the Ansible
configuration:

For each zone configured in the list `zones`, a zone entry is generated in the
file. The entry is started with a comment that is filled with the zone
`description`.  The name is taken from the zone `name`. The name is also used
as a suffix of the name of the referenced `file` where the content of the zone
is configured.

The template for the files referenced in `named.conf.local` is defined as
follows in the file `roles/bind/templates/db-includes.j2`:

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

The template creates the base file for the content of a zone. It creates the
following records:

- `SOA` record with the zone `name`, `contact` and current `serial` number.
- `NS` with the concatenation of `ns.` with the zone `name`
- `A` record for the zone name with the IP address of the DNS server
- `AAAA` record for the zone name with the IPv6 loopback address
- `A` record for the name `ns` with the IP address of the DNS server

For each `file` in the list `includes`, the template creates an `$INCLUDE`
entry with the `file` and DNS `name`.

The template for the files referenced by the base files above is defined as
follows in the file `roles/bind/templates/db-actual.j2`:

```jinja
{% for a in item.1.a_records %}
{{ a.name }}	IN	A	{{ a.ip }}
{% endfor %}
```

The template creates an `A` record entry for each `A` record in the list of `A`
records. Each entry maps the `name` of the record to the address `ip`.

### Configuration and Playbook

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

### Deployment

Deployment:

```console
$ # site 1
$ ansible-playbook -i site1/hosts bind.yml
$ # site 2
$ ansible-playbook -i site2/hosts bind.yml
```
