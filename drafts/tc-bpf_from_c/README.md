# TC-BPF from C code

This document describes how a eBPF program for TC can be attached and detached
from C code, without using the `tc` tool.

Note: this document was started before libbpf supported TC program loading.
Maybe a, probably much shorter, libbpf version of this will come in the future.
Still, this document provides information on how tc, netlink, and bpf interact
and can be used from a C program.

## Overview

Attaching BPF programs consists of the following steps:

* Load bpf program
* Add qdisc on network interface
* Add tc filter with bpf program on network interface

Detaching BPF programs can be achieved with the following step:

* Remove qdisc on network interface

## Loading the BPF Program

As a first step, the bpf program needs to be loaded into the kernel. For
example, this can be achieved with libbpf's function `bpf_prog_load_xattr()`.
The load function requires the specification of the bpf program and the program
type. In this case, the program type is `BPF_PROG_TYPE_SCHED_CLS`. Loading the
bpf kernel returns a file descriptor that can be used to reference the loaded
program in the following steps.

## Adding the QDISC

The next step is adding the traffic control (TC) queueing discipline (QDISC).
TC configuration relies on a Netlink interface. So, a netlink socket has to be
created and an appropriate netlink message has to be sent to the kernel over
the socket.

The netlink socket is a routing (rtnetlink) socket with the family
`NETLINK_ROUTE`.

```c
/* create socket address */
struct sockaddr_nl sa;
memset(&sa, 0, sizeof(sa));
sa.nl_family = AF_NETLINK;

/* create and bind socket */
int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
if (fd < 0) {
	return fd;
}
if (bind(fd, (struct sockaddr *) &sa, sizeof(sa))) {
	return -1;
}
```

The netlink message consists of a header and an embedded TC message with a TC
kind attribute.

```
+--------------------------------------------------+
|                                   Netlink Header |
| type: RTM_RTM_NEWQDISC                           |
| flags: NLM_F_REQUEST | NLM_F_CREATE              |
+--------------------------------------------------+
|                                       TC Message |
| family: AF_UNSPEC                                |
| ifindex: if_index                                |
| handle: TC_H_MAKE(TC_H_CLSACT, 0)                |
| parent: TC_H_CLSACT                              |
| info: TC_H_MAKE(0, htons(ETH_P_ALL))             |
+--------------------------------------------------+
|                                   Kind Attribute |
| type: TCA_KIND                                   |
| data: "clsact"                                   |
+--------------------------------------------------+
```

```c
/* create request message */
struct {
	struct nlmsghdr hdr;
	struct tcmsg tcm;
	char attrbuf[512];
} req;
memset(&req, 0, sizeof(req));
```

The header specifies the message type `RTM_NEWQDISC` and the flags
`NLM_F_REQUEST | NLM_F_CREATE` that indicate a request to create a new QDISC.

```c
/* fill header */
req.hdr.nlmsg_len = NLMSG_LENGTH(sizeof(req.tcm));
req.hdr.nlmsg_pid = 0;
req.hdr.nlmsg_seq = 1;
req.hdr.nlmsg_type = RTM_NEWQDISC;
req.hdr.nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE;
```

The TC message specifies the TC family `AF_UNSPEC`, the index of the network
interface where the QDISC should be created, the TC handle
`TC_H_MAKE(TC_H_CLSACT, 0)` and the TC parent `TC_H_CLSACT`.

```c
/* fill tc message */
req.tcm.tcm_family = AF_UNSPEC;
req.tcm.tcm_ifindex = if_nametoindex(if_name);
req.tcm.tcm_handle = TC_H_MAKE(TC_H_CLSACT, 0);
req.tcm.tcm_parent = TC_H_CLSACT;
```

The kind attribute is a netlink routing attribute of type `TCA_KIND` and
contains the kind `clsact` as a string.

```c
/* add kind attribute */
const char *kind = "clsact";
struct rtattr *kind_rta;
kind_rta = (struct rtattr *)(((char *) &req) +
			     NLMSG_ALIGN(req.hdr.nlmsg_len));
kind_rta->rta_type = TCA_KIND;
kind_rta->rta_len = RTA_LENGTH(strnlen(kind, 6));
memcpy(RTA_DATA(kind_rta), kind, strnlen(kind, 6));

/* update message length */
req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) + kind_rta->rta_len;
```

## Adding the TC Filter

With the QDISC configured in the kernel, the TC Filter can be added. This also
requires communication with the kernel over the netlink routing socket.

This time the netlink message consists of a header and an embedded TC message
with a kind and an options attribute. The options attribute contains a bpf
file descriptor attribute, a bpf name attribute, and a bpf flags attribute.

```
+--------------------------------------------------+
|                                   Netlink Header |
| type: RTM_NEWTFILTER                             |
| flags: NLM_F_REQUEST | NLM_F_CREATE              |
+--------------------------------------------------+
|                                       TC Message |
| family: AF_UNSPEC                                |
| ifindex: if_index                                |
| handle: 0                                        |
| parent: TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS) |
| info: TC_H_MAKE(0, htons(ETH_P_ALL))             |
+--------------------------------------------------+
|                                   Kind Attribute |
| type: TCA_KIND                                   |
| data: "bpf"                                      |
+--------------------------------------------------+
|                                Options Attribute |
| type: TCA_OPTIONS                                |
+--------------------------------------------------+
|                    BPF File Descriptor Attribute |
| type: TCA_BPF_FD                                 |
| data: bpf_fd                                     |
+--------------------------------------------------+
|                               BPF Name Attribute |
| type: TCA_BPF_NAME                               |
| data: name                                       |
+--------------------------------------------------+
|                              BPF Flags Attribute |
| type: TCA_BPF_FLAGS                              |
| data: TCA_BPF_FLAG_ACT_DIRECT                    |
+--------------------------------------------------+
```

```c
/* create request message */
struct {
	struct nlmsghdr hdr;
	struct tcmsg tcm;
	char attrbuf[512];
} req;
memset(&req, 0, sizeof(req));
```

The header specifies the message type `RTM_NEWTFILTER` and the flags
`NLM_F_REQUEST | NLM_F_CREATE` that indicate a request to create a new TC
Filter.

```c
/* fill header */
req.hdr.nlmsg_len = NLMSG_LENGTH(sizeof(req.tcm));
req.hdr.nlmsg_pid = 0;
req.hdr.nlmsg_seq = 1;
req.hdr.nlmsg_type = RTM_NEWTFILTER;
req.hdr.nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE;
```

The TC message specifies the TC familiy `AF_UNSPEC`, the index of the network
interface where the filter should be added, the TC handle `0`, the TC parent
`TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS)` and the TC info `TC_H_MAKE(0,
htons(ETH_P_ALL))`.

```c
/* fill tc message */
req.tcm.tcm_family = AF_UNSPEC;
req.tcm.tcm_ifindex = if_nametoindex(if_name);
req.tcm.tcm_handle = 0;
req.tcm.tcm_parent = TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS);
req.tcm.tcm_info = TC_H_MAKE(0, htons(ETH_P_ALL));
```

The kind attribute is a netlink routing attribute of type `TCA_KIND` and
contains the kind `bpf` as a string.

```c
/* add kind attribute */
const char *kind = "bpf";
struct rtattr *kind_rta;
kind_rta = (struct rtattr *)(((char *) &req) +
			     NLMSG_ALIGN(req.hdr.nlmsg_len));
kind_rta->rta_type = TCA_KIND;
kind_rta->rta_len = RTA_LENGTH(strnlen(kind, 3) + 1);
memcpy(RTA_DATA(kind_rta), kind, strnlen(kind, 3) + 1);

/* update message length */
req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) + kind_rta->rta_len;
```

The options attribute is netlink routing attribute of type `TCA_OPTIONS` and
contains the other bpf attributes.

```c
/* add options attribute */
struct rtattr *options_rta;
options_rta = (struct rtattr *)(((char *) &req) +
				NLMSG_ALIGN(req.hdr.nlmsg_len));
options_rta->rta_type = TCA_OPTIONS;
options_rta->rta_len = RTA_LENGTH(0);
```

The bpf file descriptor attribute is a netlink routing attribute of type
`TCA_BPF_FD` and contains the file descriptor of the bpf program that was
returned by loading the bpf program into the kernel, as described above.

```c
/* add bpf fd attribute */
struct rtattr *fd_rta = RTA_DATA(options_rta);
fd_rta->rta_type = TCA_BPF_FD;
fd_rta->rta_len = RTA_LENGTH(sizeof(int));
memcpy(RTA_DATA(fd_rta), &bpf_fd, sizeof(int));

/* update options length */
options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
	fd_rta->rta_len;
```

The bpf name descriptor attribute is a netlink routing attribute of type
`TCA_BPF_NAME` and contains the name of the section within the loaded bpf
program, that identifies the packet handling function, as a string, e.g.,
`accept-all`.

```c
/* add bpf name attribute */
const char *name = "accept-all";
struct rtattr *name_rta;
name_rta = (struct rtattr *)(((char *) options_rta) +
			     RTA_ALIGN(options_rta->rta_len));
name_rta->rta_type = TCA_BPF_NAME;
name_rta->rta_len = RTA_LENGTH(strnlen(name, 10) + 1);
memcpy(RTA_DATA(name_rta), name, strnlen(name, 10 + 1));

/* update options length */
options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
	name_rta->rta_len;
```

TODO: add section to document about bpf file creation? reference to section
name in bpf name attribute?

The bpf flags attribute is a netlink routing attribute of type `TCA_BPF_FLAGS`
and contains the flag `TCA_BPF_FLAG_ACT_DIRECT` as an unsigned 32 bit integer.

```c
/* add bpf flags */
__u32 flags = TCA_BPF_FLAG_ACT_DIRECT;
struct rtattr *flags_rta;
flags_rta = (struct rtattr *)(((char *) options_rta) +
			      RTA_ALIGN(options_rta->rta_len));
flags_rta->rta_type = TCA_BPF_FLAGS;
flags_rta->rta_len = RTA_LENGTH(sizeof(flags));
memcpy(RTA_DATA(flags_rta), &flags, sizeof(flags));

/* update options length */
options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
	flags_rta->rta_len;

/* update message length */
req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) +
	options_rta->rta_len;
```

## Removing the QDISC

The previous steps can be undone by deleting the QDISC on the network
interface. Again, this requires communication with the kernel over the netlink
routing socket. The netlink message consists of a header and an embedded TC
message with a TC kind attribute. The header specifies the message type
`RTM_DELQDISC` and the flag `NLM_F_REQUEST` that indicate a request to delete a
QDISC. The TC message specifies the TC family `AF_UNSPEC`, the index of the
network interface where the QDISC should be deleted, the TC handle
`TC_H_MAKE(TC_H_CLSACT, 0)` and the TC parent `TC_H_CLSACT`. The kind attribute
is a netlink routing attribute of type `TCA_KIND` and contains the kind
`clsact` as string.

## Code Snippets

The sections below contain C code snippets that show the implementation of
tc-bpf attaching and detaching.

### Attaching a BPF Program on a Network Interface

Include headers:

```c
/* bpf */
#include <bpf/libbpf.h>

/* netlink imports */
#include <asm/types.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>

/* if_nametoindex() */
#include <net/if.h>

/* TC_H_* */
#include <linux/pkt_sched.h>

/* TCA_BPF_* */
#include <linux/pkt_cls.h>

/* ETH_P_ALL */
#include <linux/if_ether.h>

/* htons() */
#include <arpa/inet.h>
```

Function for loading the bpf program:

```c
/* load bpf program from file and return program fd */
int load_bpf(const char* file) {
	struct bpf_prog_load_attr prog_load_attr = {
		.prog_type      = BPF_PROG_TYPE_SCHED_CLS,
		.file		= file,
	};
	struct bpf_object *obj;
	int prog_fd;

	if (bpf_prog_load_xattr(&prog_load_attr, &obj, &prog_fd)) {
		return -1;
	}

	return prog_fd;
}
```

Function for creating a netlink socket:

```c
/* create netlink socket and return socket fd */
int create_socket() {
	/* create socket address */
	struct sockaddr_nl sa;
	memset(&sa, 0, sizeof(sa));
	sa.nl_family = AF_NETLINK;

	/* create and bind socket */
	int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
	if (fd < 0) {
		return fd;
	}
	if (bind(fd, (struct sockaddr *) &sa, sizeof(sa))) {
		return -1;
	}
	return fd;
}
```

Function for adding a qdisc on a network interface using the netlink socket:

```c
/* add qdisc with netlink request */
int send_request_qdisc(int fd, const char *if_name) {
	/* create socket address */
	struct sockaddr_nl sa;
	memset(&sa, 0, sizeof(sa));
	sa.nl_family = AF_NETLINK;

	/* create request message */
	struct {
		struct nlmsghdr hdr;
		struct tcmsg tcm;
		char attrbuf[512];
	} req;
	memset(&req, 0, sizeof(req));

	/* fill header */
	req.hdr.nlmsg_len = NLMSG_LENGTH(sizeof(req.tcm));
	req.hdr.nlmsg_pid = 0;
	req.hdr.nlmsg_seq = 1;
	req.hdr.nlmsg_type = RTM_NEWQDISC;
	req.hdr.nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE;

	/* fill tc message */
	req.tcm.tcm_family = AF_UNSPEC;
	req.tcm.tcm_ifindex = if_nametoindex(if_name);
	req.tcm.tcm_handle = TC_H_MAKE(TC_H_CLSACT, 0);
	req.tcm.tcm_parent = TC_H_CLSACT;

	/* add kind attribute */
	const char *kind = "clsact";
	struct rtattr *kind_rta;
	kind_rta = (struct rtattr *)(((char *) &req) +
				     NLMSG_ALIGN(req.hdr.nlmsg_len));
	kind_rta->rta_type = TCA_KIND;
	kind_rta->rta_len = RTA_LENGTH(strnlen(kind, 6));
	memcpy(RTA_DATA(kind_rta), kind, strnlen(kind, 6));

	/* update message length */
	req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) + kind_rta->rta_len;

	/* send request */
	struct iovec iov = { &req, req.hdr.nlmsg_len };
	struct msghdr msg = { &sa, sizeof(sa), &iov, 1, NULL, 0, 0 };
	sendmsg(fd, &msg, 0);

	return 0;
}
```

Function for adding a tc filter for a bpf program on a network interface using
the netlink socket:

```c
/* add tc filter with netlink request */
int send_request_filter(int fd, const char *if_name, int bpf_fd) {
	/* create socket address */
	struct sockaddr_nl sa;
	memset(&sa, 0, sizeof(sa));
	sa.nl_family = AF_NETLINK;

	/* create request message */
	struct {
		struct nlmsghdr hdr;
		struct tcmsg tcm;
		char attrbuf[512];
	} req;
	memset(&req, 0, sizeof(req));

	/* fill header */
	req.hdr.nlmsg_len = NLMSG_LENGTH(sizeof(req.tcm));
	req.hdr.nlmsg_pid = 0;
	req.hdr.nlmsg_seq = 1;
	req.hdr.nlmsg_type = RTM_NEWTFILTER;
	req.hdr.nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE;

	/* fill tc message */
	req.tcm.tcm_family = AF_UNSPEC;
	req.tcm.tcm_ifindex = if_nametoindex(if_name);
	req.tcm.tcm_handle = 0;
	req.tcm.tcm_parent = TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS);
	req.tcm.tcm_info = TC_H_MAKE(0, htons(ETH_P_ALL));

	/* add kind attribute */
	const char *kind = "bpf";
	struct rtattr *kind_rta;
	kind_rta = (struct rtattr *)(((char *) &req) +
				     NLMSG_ALIGN(req.hdr.nlmsg_len));
	kind_rta->rta_type = TCA_KIND;
	kind_rta->rta_len = RTA_LENGTH(strnlen(kind, 3) + 1);
	memcpy(RTA_DATA(kind_rta), kind, strnlen(kind, 3) + 1);

	/* update message length */
	req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) + kind_rta->rta_len;

	/* add options attribute */
	struct rtattr *options_rta;
	options_rta = (struct rtattr *)(((char *) &req) +
					NLMSG_ALIGN(req.hdr.nlmsg_len));
	options_rta->rta_type = TCA_OPTIONS;
	options_rta->rta_len = RTA_LENGTH(0);

	/* add bpf fd attribute */
	struct rtattr *fd_rta = RTA_DATA(options_rta);
	fd_rta->rta_type = TCA_BPF_FD;
	fd_rta->rta_len = RTA_LENGTH(sizeof(int));
	memcpy(RTA_DATA(fd_rta), &bpf_fd, sizeof(int));

	/* update options length */
	options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
		fd_rta->rta_len;

	/* add bpf name attribute */
	const char *name = "accept-all";
	struct rtattr *name_rta;
	name_rta = (struct rtattr *)(((char *) options_rta) +
				     RTA_ALIGN(options_rta->rta_len));
	name_rta->rta_type = TCA_BPF_NAME;
	name_rta->rta_len = RTA_LENGTH(strnlen(name, 10) + 1);
	memcpy(RTA_DATA(name_rta), name, strnlen(name, 10 + 1));

	/* update options length */
	options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
		name_rta->rta_len;

	/* add bpf flags */
	__u32 flags = TCA_BPF_FLAG_ACT_DIRECT;
	struct rtattr *flags_rta;
	flags_rta = (struct rtattr *)(((char *) options_rta) +
				      RTA_ALIGN(options_rta->rta_len));
	flags_rta->rta_type = TCA_BPF_FLAGS;
	flags_rta->rta_len = RTA_LENGTH(sizeof(flags));
	memcpy(RTA_DATA(flags_rta), &flags, sizeof(flags));

	/* update options length */
	options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
		flags_rta->rta_len;

	/* update message length */
	req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) +
		options_rta->rta_len;

	/* send request */
	struct iovec iov = { &req, req.hdr.nlmsg_len };
	struct msghdr msg = { &sa, sizeof(sa), &iov, 1, NULL, 0, 0 };
	sendmsg(fd, &msg, 0);

	return 0;
}
```

Main function that combines the previous functions:

```c
int main(int argc, char **argv) {
	/* handle command line arguments */
	if (argc < 3) {
		return -1;
	}
	const char *bpf_file = argv[1];
	const char *if_name = argv[2];

	/* load bpf program */
	int prog_fd = load_bpf(bpf_file);
	if (prog_fd < 0) {
		printf("Error loading bpf program\n");
		return -1;
	}

	/* create netlink socket */
	int nl_fd = create_socket();
	if (nl_fd < 0) {
		printf("Error creating netlink socket\n");
		return -1;
	}

	/* add qdisc */
	send_request_qdisc(nl_fd, if_name);

	/* add tc filter */
	send_request_filter(nl_fd, if_name, prog_fd);

	return 0;
}
```

### Detaching BPF Programs on a Network Interface

Include headers:

```c
/* netlink imports */
#include <asm/types.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>

/* memset */
#include <string.h>

/* printf */
#include <stdio.h>

/* if_nametoindex() */
#include <net/if.h>

/* TC_H_* */
#include <linux/pkt_sched.h>
```

Function for creating a netlink socket:

```c
/* create netlink socket and return socket fd */
int create_socket() {
	/* create socket address */
	struct sockaddr_nl sa;
	memset(&sa, 0, sizeof(sa));
	sa.nl_family = AF_NETLINK;

	/* create and bind socket */
	int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
	if (fd < 0) {
		return fd;
	}
	if (bind(fd, (struct sockaddr *) &sa, sizeof(sa))) {
		return -1;
	}
	return fd;
}
```

Function for removing the qdisc on a network interface using the netlink
socket:

```c
/* send netlink request */
int send_request(int fd, const char *if_name) {
	/* create socket address */
	struct sockaddr_nl sa;
	memset(&sa, 0, sizeof(sa));
	sa.nl_family = AF_NETLINK;

	/* create request message */
	struct {
		struct nlmsghdr hdr;
		struct tcmsg tcm;
		char attrbuf[512];
	} req;
	memset(&req, 0, sizeof(req));

	/* fill header */
	req.hdr.nlmsg_len = NLMSG_LENGTH(sizeof(req.tcm));
	req.hdr.nlmsg_pid = 0;
	req.hdr.nlmsg_seq = 1;
	req.hdr.nlmsg_type = RTM_DELQDISC;
	req.hdr.nlmsg_flags = NLM_F_REQUEST;

	/* fill tc message */
	req.tcm.tcm_family = AF_UNSPEC;
	req.tcm.tcm_ifindex = if_nametoindex(if_name);
	req.tcm.tcm_handle = TC_H_MAKE(TC_H_CLSACT, 0);
	req.tcm.tcm_parent = TC_H_CLSACT;

	/* add kind attribute */
	const char *kind = "clsact";
	struct rtattr *kind_rta;
	kind_rta = (struct rtattr *)(((char *) &req) +
				     NLMSG_ALIGN(req.hdr.nlmsg_len));
	kind_rta->rta_type = TCA_KIND;
	kind_rta->rta_len = RTA_LENGTH(strnlen(kind, 6));
	memcpy(RTA_DATA(kind_rta), kind, strnlen(kind, 6));

	/* update message length */
	req.hdr.nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) + kind_rta->rta_len;

	/* send request */
	struct iovec iov = { &req, req.hdr.nlmsg_len };
	struct msghdr msg = { &sa, sizeof(sa), &iov, 1, NULL, 0, 0 };
	sendmsg(fd, &msg, 0);

	return 0;
}
```

Main function that combines the previous functions:

```c
int main(int argc, char **argv) {
	if (argc < 2) {
		return -1;
	}
	int fd = create_socket();
	send_request(fd, argv[1]);
}
```
