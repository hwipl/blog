# DHCP with ISC DHCP and Ansible

This document describes how you can deploy ISC DHCP servers with Ansible. It
shows the DHCP configuration of ISC DHCP as well as the Ansible role, playbook
and configuration that are used to install and configure each DHCP server
automatically.

## Overview

```
.........................................................................
: Site 1                            :                            Site 2 :
: 10.20.1.0/24                      :                      10.20.2.0/24 :
: 10.20.201.0/24                    :                    10.20.202.0/24 :
:                                   :                                   :
:          +-------------+          :          +-------------+          :
:          | Node 1      |          :          | Node 1      |          :
:          | DHCP Server |          :          | DHCP Server |          :
:          | 10.20.1.1   |          :          | 10.20.2.1   |          :
:          | 10.20.201.1 |          :          | 10.20.202.1 |          :
:          +-------------+          :          +-------------+          :
:          _______|_______          :          _______|_______          :
:         |               |         :         |               |         :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:  | Node 2      | | Node 3      |  :  | Node 2      | | Node 3      |  :
:  | DHCP Client | | DHCP Client |  :  | DHCP Client | | DHCP Client |  :
:  | 10.20.1.2   | | 10.20.1.3   |  :  | 10.20.2.2   | | 10.20.2.3   |  :
:  | 10.20.201.2 | | 10.20.201.3 |  :  | 10.20.202.2 | | 10.20.202.3 |  :
:  +-------------+ +-------------+  :  +-------------+ +-------------+  :
:...................................:...................................:
                        Network: 10.20.0.0/16
```

| Entity      | IP Address     |  MAC Address      |
|-------------|----------------|-------------------|
| Network     | 10.20.0.0/16   |                   |
| Site 1      | 10.20.1.0/24   |                   |
| - Node 1    | 10.20.1.1      | ca:fe:ca:fe:11:01 |
| - Node 2    | 10.20.1.2      | ca:fe:ca:fe:12:01 |
| - Node 3    | 10.20.1.3      | ca:fe:ca:fe:13:01 |
| Site 2      | 10.20.2.0/24   |                   |
| - Node 1    | 10.20.2.1      | ca:fe:ca:fe:21:01 |
| - Node 2    | 10.20.2.2      | ca:fe:ca:fe:22:01 |
| - Node 3    | 10.20.2.3      | ca:fe:ca:fe:23:01 |
| Site 1 Mgmt | 10.20.201.0/24 |                   |
| - Node 1    | 10.20.201.1    | ca:fe:ca:fe:11:0a |
| - Node 2    | 10.20.201.2    | ca:fe:ca:fe:12:0a |
| - Node 3    | 10.20.201.3    | ca:fe:ca:fe:13:0a |
| Site 2 Mgmt | 10.20.202.0/24 |                   |
| - Node 1    | 10.20.202.1    | ca:fe:ca:fe:21:0a |
| - Node 2    | 10.20.202.2    | ca:fe:ca:fe:22:0a |
| - Node 3    | 10.20.202.3    | ca:fe:ca:fe:23:0a |

## DHCP Configuration

## Ansible

## Conclusion
