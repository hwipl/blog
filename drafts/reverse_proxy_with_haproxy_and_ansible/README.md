# Reverse Proxy with HAProxy and Ansible

This document describes how you can deploy HAProxy reverse proxy servers with
Ansible. It shows the reverse proxy configuration as well as the Ansible role,
playbook and configuration that are used to install and configure each HAProxy
server automatically.

## Overview

The reverse proxy servers in this document run HAProxy on Ubuntu 22.04 LTS.
HAProxy is installed and configured automatically with Ansible. The HAPorxy
configuration in this document assumes the network in the following figure:

```
                        Other Networks
                     _________|_________
                    |                   |
....................|...................|....................
: Site 1            |         :         |            Site 2 :
: 10.20.1.0/24      |         :         |      10.20.2.0/24 :
:                   |         :         |                   :
:             +-----------+   :   +-----------+             :
:             | Node 1    |   :   | Node 1    |             :
:             | HAProxy   |   :   | HAProxy   |             :
:             | 10.20.1.1 |   :   | 10.20.2.1 |             :
:             +-----------+   :   +-----------+             :
:                   |         :         |                   :
:                   |         :         |                   :
:  +------------+   |         :         |   +------------+  :
:  | Node 11    |   |         :         |   | Node 11    |  :
:  | Service B  |___|         :         |___| Service B  |  :
:  | 10.20.1.11 |   |         :         |   | 10.20.2.11 |  :
:  +------------+   |         :         |   +------------+  :
:  +------------+   |         :         |   +------------+  :
:  | Node 13    |   |         :         |   | Node 13    |  :
:  | Service A  |___|         :         |___| Service A  |  :
:  | 10.20.1.13 |   |         :         |   | 10.20.2.13 |  :
:  +------------+   |         :         |   +------------+  :
:  +------------+   |         :         |   +------------+  :
:  | Node 14    |   |         :         |   | Node 14    |  :
:  | Service C  |___|         :         |___| Service C  |  :
:  | 10.20.1.14 |   |         :         |   | 10.20.2.14 |  :
:  +------------+   |         :         |   +------------+  :
:  +------------+   |         :         |   +------------+  :
:  | Node 21    |   |         :         |   | Node 21    |  :
:  | Service B  |___|         :         |___| Service B  |  :
:  | 10.20.1.21 |             :             | 10.20.2.21 |  :
:  +------------+             :             +------------+  :
:.............................:.............................:
                    Network: 10.20.0.0/16
```

The example network consists of the two sites `Site 1` and `Site 2`. Each site
contains five nodes. `Node 1` runs the HAProxy server and is connected to the
other nodes in the site as well as other networks. `Node 11`, `Node 13`, `Node
14` and `Node 21` run services `A`, `B` and `C`. Service `A` and `C` are each
only provided a single node (`Node 13` and `Node 14`). Service `B` is provided
by the two nodes `Node 11` and `Node 21`. In each site, the HAProxy server
enables access from other networks to these services.

| Service | Port   | Type         | Server           |
|---------|--------|--------------|------------------|
| A       | :8443  | Passthrough  | 10.20.1.13:8443  |
| B       | :32196 | Termination  | 10.20.1.11:32196 |
|         |        |              | 10.20.1.21:32196 |
| C       | :3000  | Termination  | 10.20.1.14:3000  |

## Reverse Proxy Configuration

HAProxy in Site 1 is configured as follows in the file
`/etc/haproxy/haproxy.cfg`:

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

# frontends
frontend A
	bind *:8443
	option tcplog
	mode tcp
	default_backend A-servers
frontend B
	bind *:32196 ssl crt /etc/haproxy/site1-haproxy.pem
	option tcplog
	mode tcp
	default_backend B-servers
frontend C
	bind *:3000 ssl crt /etc/haproxy/site1-haproxy.pem
	option tcplog
	mode tcp
	default_backend C-servers

# backends
backend A-servers
	mode tcp
	balance roundrobin
	option ssl-hello-chk
	server a1.s1.network.lan 10.20.1.13:8443 check
backend B-servers
	mode tcp
	balance roundrobin
	server b1.s1.network.lan 10.20.1.11:32196 check
	server b2.s1.network.lan 10.20.1.21:32196 check
backend C-servers
	mode tcp
	balance roundrobin
	server c1.s1.network.lan 10.20.1.14:3000 check
```

HAProxy in Site 2 is configured as follows in the file
`/etc/haproxy/haproxy.cfg`:

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

# frontends
frontend A
	bind *:8443
	option tcplog
	mode tcp
	default_backend A-servers
frontend B
	bind *:32196 ssl crt /etc/haproxy/site2-haproxy.pem
	option tcplog
	mode tcp
	default_backend B-servers
frontend C
	bind *:3000 ssl crt /etc/haproxy/site2-haproxy.pem
	option tcplog
	mode tcp
	default_backend C-servers

# backends
backend A-servers
	mode tcp
	balance roundrobin
	option ssl-hello-chk
	server a1.s2.network.lan 10.20.2.13:8443 check
backend B-servers
	mode tcp
	balance roundrobin
	server b1.s2.network.lan 10.20.2.11:32196 check
	server b2.s2.network.lan 10.20.2.21:32196 check
backend C-servers
	mode tcp
	balance roundrobin
	server c1.s2.network.lan 10.20.2.14:3000 check
```

## Ansible

Ansible allows for automatic installation and configuration of the revers proxy
servers. Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist
of tasks, templates and handlers. Tasks are the individual installation and
configuration steps. They use the [templates][templates] to generate
configuration files and trigger events that are handled by the
[handlers][handlers].

A role and a playbook are used to deploy the HAProxy servers. The playbook
assigns the role to all nodes of a group defined in the Ansible
[inventory][inventory]. The configuration of each HAProxy server is derived
from variables in the inventory. Each site uses a different inventory to allow
for site-specific configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections.  Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

### Role

The Ansible role is called `haproxy` and structured as shown in the listing
below:

```
roles/haproxy/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── haproxy.cfg.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and one template file. The tasks use the template `haproxy.cfg.j2` to
create the configuration file of the reverse proxy servers.

#### Handlers

A handler is defined as follows in the file `roles/haproxy/handlers/main.yml`:

```yaml
---
# handlers for haproxy

- name: Restart haproxy
  become: true
  ansible.builtin.service:
    name: haproxy
    state: restarted
```

The handler is called `Restart haproxy` and restarts the HAProxy server with
the [service module][service] when it is triggered. It requires root privileges
to manipulate the state of the system services, so [become][become] is set to
`true` for [privilege escalation][privilege]. To restart the HAProxy server, it
sets the system service `haproxy` to state `restarted`.

#### Tasks

The tasks are defined as follows in the file `roles/haproxy/tasks/main.yml`:

```yaml
---
# these tasks setup haproxy

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure haproxy is installed
  become: true
  ansible.builtin.apt:
    name: haproxy
    state: present
- name: Copy certificates
  become: true
  ansible.builtin.copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: root
    group: root
    mode: '0600'
  loop: "{{ haproxy_certificates }}"
- name: Create haproxy configuration from template
  become: true
  ansible.builtin.template:
    src: haproxy.cfg.j2
    dest: "/etc/haproxy/haproxy.cfg"
    owner: root
    group: root
    mode: '0644'
  notify:
    - Restart haproxy
```

These tasks install the reverse proxy server with the [apt module][apt],
configure it with the template and the [template module][template] and trigger
the restart event with [notify][notify] if the config file is changed:

All tasks need root privileges to manipulate the system configuration. So,
[become][become] is set to `true`. The file owner and group of all files is set
to `root`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs HAProxy with `apt` if it is not already installed.
3. The third task copies all certificates defined in the Ansible list variable
   `haproxy_certificates` from their source file defined in `src` to their
   destination file defined in `dest`. File permissions are set to `600`.
4. The fourth task creates or updates the configuration of HAProxy in file
   `/etc/haproxy/haproxy.cfg` from the template `haproxy.cfg.j2`. File
   permissions are set to `644`. If the file is changed, the restart event is
   triggered.

#### Templates

The template for the configuration file `/etc/haproxy/haproxy.cfg` is defined
as [Jinja2 template][jinja2] as follows in the file
`roles/haproxy/templates/proxy.cfg.j2`:

```jinja
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

# frontends
{% for frontend in haproxy_frontends %}
frontend {{ frontend.name }}
{% for entry in frontend.config %}
	{{ entry }}
{% endfor %}
{% endfor %}

# backends
{% for backend in haproxy_backends %}
backend {{ backend.name }}
{% for entry in backend.config %}
	{{ entry }}
{% endfor %}
{% endfor %}
```

The template reflects the HAProxy configuration shown in the Reverse Proxy
Configuration section above with parts dynamically generated based on the
Ansible configuration:

- For each frontend in the Ansible list `frontends`, the template creates a
  `frontend` block. The name of a block is taken from the frontend variable
  `name`. All entries of a frontend configuration are taken from the frontend
  list variable `config`.
- Like for the frontends, the template creates a `backend` block for each
  backend in the Ansible list `backends`. The name and configuration entries of
  a block are taken from the backend variables `name` and `config`.

### Playbook

The playbook is defined as follows in the file `haproxy.yml`:

```yaml
---
- name: Configure haproxy
  hosts: haproxy_servers

  roles:
    - haproxy
```

The playbook assigns the role `haproxy` that is described above to all hosts in
the group `haproxy_servers`. On execution, this playbook runs all the tasks of
the role on all the hosts in the group to install and configure the HAProxy
servers.

### Configuration

The configuration is derived from host and group files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use. Each site uses a different inventory and, thus, different host and group
files to allow for site-specific configurations as shown below.

#### Hosts

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[haproxy_servers]
node1
```

The hosts file of each site defines the group `haproxy_servers` and assigns the
node `node1` to it. Thus, `node1` is defined as HAProxy server for the
playbook. There is no other host-specific configuration of the HAProxy servers
in the `host_vars` of `node1` in each site.

#### Groups

The site-specific HAProxy configuration of Site 1 is defined as follows in the
file `site1/group_vars/haproxy_servers`:

```yaml
---
# haproxy server configuration

# a haproxy certificate is a pem file that contains both the certificate and
# private key, e.g., created with:
# cat cert_file key_file | tee haproxy_cert.pem
haproxy_certificates:
- src: "{{ inventory_dir }}/certs/site1-haproxy.pem"
  dest: /etc/haproxy/site1-haproxy.pem

# haproxy frontend configurations
haproxy_frontends:
- name: A
  config:
  - bind *:8443
  - option tcplog
  - mode tcp
  - default_backend A-servers
- name: B
  config:
  - bind *:32196 ssl crt /etc/haproxy/site1-haproxy.pem
  - option tcplog
  - mode tcp
  - default_backend B-servers
- name: C
  config:
  - bind *:3000 ssl crt /etc/haproxy/site1-haproxy.pem
  - option tcplog
  - mode tcp
  - default_backend C-servers

# haproxy backend configurations
haproxy_backends:
- name: A-servers
  config:
  - mode tcp
  - balance roundrobin
  - option ssl-hello-chk
  - server a1.s1.network.lan 10.20.1.13:8443 check
- name: B-servers
  config:
  - mode tcp
  - balance roundrobin
  - server b1.s1.network.lan 10.20.1.11:32196 check
  - server b2.s1.network.lan 10.20.1.21:32196 check
- name: C-servers
  config:
  - mode tcp
  - balance roundrobin
  - server c1.s1.network.lan 10.20.1.14:3000 check
```

The site-specific HAProxy configuration of Site 2 is defined as follows in the
file `site2/group_vars/haproxy_servers`:

```yaml
---
# haproxy server configuration

# a haproxy certificate is a pem file that contains both the certificate and
# private key, e.g., created with:
# cat cert_file key_file | tee haproxy_cert.pem
haproxy_certificates:
- src: "{{ inventory_dir }}/certs/site2-haproxy.pem"
  dest: /etc/haproxy/site2-haproxy.pem

# haproxy frontend configurations
haproxy_frontends:
- name: A
  config:
  - bind *:8443
  - option tcplog
  - mode tcp
  - default_backend A-servers
- name: B
  config:
  - bind *:32196 ssl crt /etc/haproxy/site2-haproxy.pem
  - option tcplog
  - mode tcp
  - default_backend B-servers
- name: C
  config:
  - bind *:3000 ssl crt /etc/haproxy/site2-haproxy.pem
  - option tcplog
  - mode tcp
  - default_backend C-servers

# haproxy backend configurations
haproxy_backends:
- name: A-servers
  config:
  - mode tcp
  - balance roundrobin
  - option ssl-hello-chk
  - server a1.s2.network.lan 10.20.2.13:8443 check
- name: B-servers
  config:
  - mode tcp
  - balance roundrobin
  - server b1.s2.network.lan 10.20.2.11:32196 check
  - server b2.s2.network.lan 10.20.2.21:32196 check
- name: C-servers
  config:
  - mode tcp
  - balance roundrobin
  - server c1.s2.network.lan 10.20.2.14:3000 check
```

Both files set the configuration of HAProxy in Site 1 and Site 2 as described
in the Reverse Proxy Configuration section above. The frontends are configured
in the Ansible list variable `haproxy_frontends`. The corresponding backends
are configured in the Ansible list variable `haproxy_backends`. Each entry is
identified by its name in the variable `name` and contains its configuration in
the list variable `config`.

### Deployment

The reverse proxy servers can be installed and configured with the Ansible
role, configuration and playbook described above. You can use the following
[ansible-playbook][ansible-playbook] commands:

```console
$ # site 1
$ ansible-playbook -i site1/hosts haproxy.yml
$ # site 2
$ ansible-playbook -i site2/hosts haproxy.yml
```

Both `ansible-playbook` commands run the playbook `haproxy.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
first command installs and configures all HAProxy servers in Site 1 and the
second command in Site 2. After successful execution of the commands above, the
reverse proxy servers should be configured and running.

## Conclusion

This document describes how you can automatically install and configure HAProxy
servers on Ubuntu 22.04 LTS with Ansible. After presenting an example network
and HAProxy configuration, an Ansible Role is shown that contains the
installation and configuration tasks as well as a template. The tasks use the
template together with Ansible variables to create the HAProxy configuration.
Finally, an Ansible Playbook is used with the `ansible-playbook` command to run
the tasks and, thus, install and configure all reverse proxy servers. All
sections contain configuration or code examples you can use as a basis for your
own setup. Also, you can find links to the code and configuration examples in
the appendix below.

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/haproxy/roles/haproxy)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/haproxy/haproxy.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/haproxy/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/haproxy/examples/site2)

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
