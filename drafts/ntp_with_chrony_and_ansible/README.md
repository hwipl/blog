# NTP with Chrony and Ansible

This document describes how you can deploy chrony NTP servers with Ansible. It
shows the NTP configuration of chrony as well as the Ansible role, playbook and
configuration that are used to install and configure each NTP server
automatically.

## Overview

The NTP servers in this document run the chrony NTP Server on Ubuntu 22.04 LTS.
Chrony is installed and configured automatically with Ansible. The NTP
configuration assumes the network in the following figure:

```
                        +-----------------------+
                        |    NTP Server Pools   |
                        +-----------------------+
                        | 0.ubuntu.pool.ntp.org |
                        | 1.ubuntu.pool.ntp.org |
                        | 2.ubuntu.pool.ntp.org |
                        | 3.ubuntu.pool.ntp.org |
                        +-----------------------+
                   _________________|_________________
                  |                                   |
..................|...................................|..................
: Site 1          |                 :                 |          Site 2 :
: 10.20.1.0/24    |                 :                 |    10.20.2.0/24 :
:                 |                 :                 |                 :
:          +-------------+          :          +-------------+          :
:          | Node 1      |          :          | Node 1      |          :
:          | NTP Server  |          :          | NTP Server  |          :
:          | 10.20.1.1   |          :          | 10.20.2.1   |          :
:          +-------------+          :          +-------------+          :
:          _______|_______          :          _______|_______          :
:         |               |         :         |               |         :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:  | Node 2      | | Node 3      |  :  | Node 2      | | Node 3      |  :
:  | NTP Client  | | NTP Client  |  :  | NTP Client  | | NTP Client  |  :
:  | 10.20.1.2   | | 10.20.1.3   |  :  | 10.20.2.2   | | 10.20.2.3   |  :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:...................................:...................................:
                        Network: 10.20.0.0/16
```

The example network consists of the two sites `Site 1` and `Site 2`. Each site
contains three nodes. `Node 1` runs the NTP server. `Node 2` and `Node 3` are
NTP clients. The NTP clients in a site use the NTP server in the same site.
But in case of a server failure, clients could also use the server in the other
site as fallback. So, both servers must allow access from clients in both
sites. The NTP servers use the Ubuntu default NTP server pools
`0.ubuntu.pool.ntp.org`, `1.ubuntu.pool.ntp.org`, `2.ubuntu.pool.ntp.org` and
`3.ubuntu.pool.ntp.org` as external time sources.

## NTP Configuration

The configuration of the chrony NTP servers is described in this section. The
servers are configured with the following directives in the file
`/etc/chrony/chrony.conf`:

```
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst

# Allow access to this ntp server from the following hosts/networks
allow 10.20.1.0/24
allow 10.20.2.0/24
```

The [pool][pool] directives configure the ubuntu NTP server pools
`0.ubuntu.pool.ntp.org`, `1.ubuntu.pool.ntp.org`, `2.ubuntu.pool.ntp.org` and
`3.ubuntu.pool.ntp.org` as time sources for the NTP servers. The [allow][allow]
directives set the IP subnets from which NTP clients are allowed to access the
NTP servers to `10.20.1.0/24` and `10.20.2.0/24`.

## Ansible

Ansible allows for automatic installation and configuration of the NTP
servers. Ansible uses [roles][roles] and [playbooks][playbooks]. Roles consist
of tasks, templates and handlers. Tasks are the individual installation and
configuration steps. They use the [templates][templates] to generate
configuration files and trigger events that are handled by the
[handlers][handlers].

A role and a playbook are used to deploy the NTP servers. The playbook assigns
the role to all nodes of a group defined in the Ansible [inventory][inventory].
The configuration of each NTP server is derived from host and group variables
in the inventory. Each site uses a different inventory to allow for
site-specific configurations. The deployment is finally performed with the
[ansible-playbook][ansible-playbook] command. The role, playbook, configuration
and deployment are shown in the following subsections. Additionally, you can
find links to the code and configuration examples in the appendix at the end of
this document.

### Role

The Ansible role is called `chronyd` and structured as shown in the listing
below:

```
roles/chronyd/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── chrony.conf.j2
```

The role consists of one `main.yml` file for handlers, one `main.yml` file for
tasks and one template file. The tasks use the template `chrony.conf.j2` to
create the NTP configuration.

#### Handlers

A handler is defined as follows in the file `roles/chronyd/handlers/main.yml`:

```yaml
---
# handlers for chronyd

- name: Restart chrony
  become: true
  ansible.builtin.service:
    name: chrony
    state: restarted
```

The handler is called `Restart chrony` and restarts the NTP server with the
[service module][service] when it is triggered. It requires root privileges to
manipulate the state of the system services, so [become][become] is set to
`true` for [privilege escalation][privilege]. To restart the NTP server, it
sets the system service `chrony` to state `restarted`.

#### Tasks

The tasks are defined as follows in the file `roles/chronyd/tasks/main.yml`:

```yaml
---
# these tasks install and configure chrony ntp servers

- name: Update apt cache if older than 3600 seconds
  become: true
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
- name: Ensure chrony is installed
  become: true
  ansible.builtin.apt:
    name: chrony
    state: present
- name: Create chrony configuration from template
  become: true
  ansible.builtin.template:
    src: chrony.conf.j2
    dest: "/etc/chrony/chrony.conf"
    owner: root
    group: root
    mode: '0644'
  notify:
    - Restart chrony
```

These tasks install the chrony NTP server with the [apt module][apt], configure
it with the template and the [template module][template] and trigger the
restart event with [notify][notify] if the configuration is actually changed:

All tasks need root privileges to manipulate the system configuration. So,
[become][become] is set to `true`. The file owner of the configuration file is
set to `root`, group is set to `root` and privileges are set to `644`.

1. The first task updates the `apt` cache if it is older than one hour to make
   sure the following installation task can run with up-to-date package
   sources.
2. The second task installs the chrony NTP server with `apt` if it is not
   already installed.
3. The third task creates or updates the configuration of the NTP server in
   file `/etc/chrony/chrony.conf` from the template `chrony.conf.j2`. If the
   file is changed, the restart event is triggered.

#### Templates

The template for the file `/etc/chrony/chrony.conf` is a [Jinja2
template][jinja2]. It is defined as follows in the file
`roles/chronyd/templates/chrony.conf.j2`:

```jinja
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst

# Allow access to this ntp server from the following hosts/networks
{% for allow in chronyd_allows %}
allow {{ allow }}
{% endfor %}
```

The template reflects the chrony configuration shown in the NTP configuration
section above with parts dynamically generated based on the Ansible
configuration: The allowed IP address ranges are taken from the Ansible list
variable `chronyd_allows`.

### Playbook

The playbook is defined as follows in the file `chronyd.yml`:

```yaml
---
- name: Configure ntp servers
  hosts: ntp_servers

  roles:
    - chronyd
```

The playbook assigns the role `chronyd` that is described above to all hosts in
the group `ntp_servers`. On execution, this playbook runs all the tasks of the
role on all the hosts in the group to install and configure the NTP servers.

### Configuration

The configuration is derived from host and group files in the Ansible
[inventory][inventory]. They contain the variables that the tasks of the role
use. Each site uses a different inventory and, thus, different host and group
files to allow for site-specific configurations as shown below.

#### Hosts

The Ansible hosts are defined as follows in the files `site1/hosts` and
`site2/hosts`:

```ini
[ntp_servers]
node1
```

The hosts file of each site defines the group `ntp_servers` and assigns the
node `node1` to it. Thus, `node1` is defined as NTP server for the playbook.
There is no other host-specific configuration of the NTP servers in the
`host_vars` of `node1` in each site.

#### Groups

The site-specific NTP configuration of Site 1 and Site 2 is defined as follows
in the files `site1/group_vars/ntp_servers` and `site2/group_vars/ntp_servers`:

```yaml
---
# ntp server configuration

chronyd_allows:
  - 10.20.1.0/24
  - 10.20.2.0/24
```

Both files set the configuration of the allowed IP address ranges in Site 1 and
Site 2 as described in the NTP configuration section above. The allowed address
ranges are configured in the Ansible list variable `chronyd_allows`.

### Deployment

The NTP servers can be installed and configured with the Ansible role,
configuration and playbook described above. You can use the following
[ansible-playbook][ansible-playbook] commands:

```console
$ # site 1
$ ansible-playbook -i site1/hosts chronyd.yml
$ # site 2
$ ansible-playbook -i site2/hosts chronyd.yml
```

Both `ansible-playbook` commands run the playbook `chronyd.yml` with the
site-specific hosts files specified with the command line argument `-i`. The
first command installs and configures all NTP servers in Site 1 and the second
command in Site 2. After successful execution of the commands above, the NTP
servers should be configured and running.

## Conclusion

This document describes how you can automatically install and configure chrony
NTP servers on Ubuntu 22.04 LTS with Ansible. After presenting an example
network and NTP configuration, an Ansible Role is shown that contains the
installation and configuration tasks as well as a template. The tasks use the
template together with Ansible variables to create the chrony NTP
configuration. Finally, an Ansible Playbook is used with the `ansible-playbook`
command to run the tasks and, thus, install and configure all servers. All
sections contain configuration or code examples you can use as a basis for your
own setup. Also, you can find links to the code and configuration examples in
the appendix below.

## Appendix: Code

You can find the Ansible role and playbook as well as the configuration of both
sites as shown in this document at the following links:

- [Ansible Role](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/chronyd/roles/chronyd)
- [Ansible Playbook](https://github.com/hwipl/ansible-playbooks/blob/main/ubuntu/chronyd/chronyd.yml)
- [Site 1 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/chronyd/examples/site1)
- [Site 2 Configuration](https://github.com/hwipl/ansible-playbooks/tree/main/ubuntu/chronyd/examples/site2)

[pool]: https://chrony-project.org/doc/4.2/chrony.conf.html#pool
[allow]: https://chrony-project.org/doc/4.2/chrony.conf.html#allow
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
