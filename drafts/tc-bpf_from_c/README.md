# TC-BPF from C code

TODO: add overview/metadata table? Release date, version/last modification,
keywords? Add glue text connecting sections? Add changelog at the end of the
document?

TODO: simplify code snippets in sections? Add comments for variables, e.g.,
`file // filename of bpf program`?

TODO: create svg figures instead of ascii art?

This document describes how you can attach and detach an eBPF program for TC on
a network interface from C code, without using the `tc` tool. It shows an
alternative to running the following `tc` commands:

```sh
# attach bpf program to network interface:
# - add qdisc on interface
# - add filter to qdisc on interface for direction ingress or egress with
#   bpf program and section
tc qdisc add dev "$INTERFACE" clsact
tc filter add dev "$INTERFACE" "$DIRECTION" bpf \
      direct-action obj "$BPF_PROGRAM" sec "$BPF_SECTION"
```

```sh
# detach bpf program from network interface:
# - remove qdisc on interface
tc qdisc del dev "$INTERFACE" clsact
```

Note: creating this document started before libbpf supported TC program
loading. Maybe a, probably much shorter, libbpf version of this will come in
the future.  Still, this document provides information on how tc, netlink, and
bpf interact and can be used from a C program.

## Overview

The following sections describe how to create a BPF program for TC and the
steps for attaching and detaching such a TC-BPF program.

```
+-----------------------+
|       C Program       |
|                       |
+----------[ Netlink ]--+
   |            |
   | (1)        | (2), (3), (4)
   |            |
+-----+    +---------+
| BPF |    | Traffic |
|     |    | Control |
+-----+    +---------+
```


Attaching BPF programs to a network interface consists of the following steps:

1. Load bpf program
2. Add qdisc on network interface
3. Add tc filter with bpf program on network interface

You can detach BPF programs from a network interface with the following step:

4. Remove qdisc on network interface

Steps 2,3 and 4 require communication with TC inside the kernel. So, there is
an additional section describing the TC Netlink interface before them. Also,
you can find complete code snippets at the end of this document.

## Creating a BPF Program

Attaching and using a BPF program as described in the sections below requires a
BPF program that is compatible with TC.

```
+---------+           +---------+
|         |    skb    |         |
| Traffic |---------->| BPF     |
| Control |  verdict  | Program |
|         |<----------|         |
+---------+           +---------+
```

When passing packets to the BPF program, TC identifies the corresponding
functions within the program by the section name in the compiled ELF file. The
functions should accept a linux socket buffer (SKB) and return an action like
`OK`, `SHOT`, `REDIRECT` as a verdict.

```c
/* accept all packets */
SEC("accept_all")
int _accept_all(struct __sk_buff *skb)
{
	return TC_ACT_OK;
}
```

## Loading the BPF Program

The first step of attaching the BPF program is loading it into the kernel. For
example, you can achieve this with libbpf's function `bpf_prog_load_xattr()`.
The load function requires the specification of the bpf program and the program
type. In this case, the program type is `BPF_PROG_TYPE_SCHED_CLS`. Loading the
bpf into the kernel returns a file descriptor that you can use to reference the
loaded program in the following steps.

```c
struct bpf_prog_load_attr prog_load_attr = {
	.prog_type      = BPF_PROG_TYPE_SCHED_CLS,
	.file		= file, // filename of the bpf program
};
struct bpf_object *obj;
int prog_fd;
bpf_prog_load_xattr(&prog_load_attr, &obj, &prog_fd);
```

## Communicating with TC via Netlink

Traffic Control (TC) inside the kernel and, thus, the TC configuration in the
following sections relies on a Netlink interface. So, you have to create a
respective Netlink socket and you have to send appropriate Netlink messages to
the kernel over the socket.

The netlink socket is a routing (rtnetlink) socket with the family
`NETLINK_ROUTE`.

```c
/* create socket address */
struct sockaddr_nl sa;
memset(&sa, 0, sizeof(sa));
sa.nl_family = AF_NETLINK;

/* create and bind socket */
int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
bind(fd, (struct sockaddr *) &sa, sizeof(sa));
```

The netlink messages consist of a header (`struct nlmsghdr`), a TC message
(`struct tcmsg`) and one or more netlink routing attributes (`struct rtattr`).

```c
char msg_buf[512] = { 0 }; // buffer for netlink message
struct nlmsghdr *hdr = (struct nlmsghdr *) msg_buf; // netlink message header
struct tcmsg *tcm = (struct tcmsg *) (msg_buf + sizeof(*hdr)); // tc message
char *attr_buf = msg_buf + NLMSG_ALIGN(NLMSG_LENGTH(sizeof(struct tcmsg))); // routing attributes
struct rtattr *attr = (struct rtattr *) attr_buf; // a routing attribute
```

## Adding the QDISC

The next step after loading the BPF program into the kernel is adding the
traffic control (TC) queueing discipline (QDISC). This requires communication
with the kernel over the netlink routing socket as mentioned above.

The netlink message consists of a header and an embedded TC message with a TC
kind attribute.

```
+---------------------------------------------------+
|                                    Netlink Header |
| type:    RTM_NEWQDISC                             |
| flags:   NLM_F_REQUEST | NLM_F_CREATE             |
+---------------------------------------------------+
|                                        TC Message |
| family:  AF_UNSPEC                                |
| ifindex: if_nametoindex(if_name)                  |
| handle:  TC_H_MAKE(TC_H_CLSACT, 0)                |
| parent:  TC_H_CLSACT                              |
+---------------------------------------------------+
|                                    Kind Attribute |
| type:    TCA_KIND                                 |
| data:    "clsact"                                 |
+---------------------------------------------------+
```

The header specifies the message type `RTM_NEWQDISC` and the flags
`NLM_F_REQUEST | NLM_F_CREATE` that indicate a request to create a new QDISC.

```c
/* netlink message header */
hdr->nlmsg_len = NLMSG_ALIGN(NLMSG_LENGTH(sizeof(struct tcmsg))) + RTA_LENGTH(strlen("clsact"));
hdr->nlmsg_pid = 0;
hdr->nlmsg_seq = 1;
hdr->nlmsg_type = RTM_NEWQDISC;
hdr->nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE;
```

The TC message specifies the TC family `AF_UNSPEC`, the index of the network
interface where the QDISC should be created, the TC handle
`TC_H_MAKE(TC_H_CLSACT, 0)` and the TC parent `TC_H_CLSACT`.

```c
/* tc message */
tcm->tcm_family = AF_UNSPEC;
tcm->tcm_ifindex = if_nametoindex(if_name);
tcm->tcm_handle = TC_H_MAKE(TC_H_CLSACT, 0);
tcm->tcm_parent = TC_H_CLSACT;
tcm->tcm_info = 0;
```

The kind attribute is a netlink routing attribute of type `TCA_KIND` and
contains the kind `clsact` as a string.

```c
/* kind attribute */
struct rtattr *kind_rta = (struct rtattr *) attr_buf;
kind_rta->rta_type = TCA_KIND;
kind_rta->rta_len = RTA_LENGTH(strlen("clsact"));
memcpy(RTA_DATA(kind_rta), "clsact", strlen("clsact"));
```

## Adding the TC Filter

With the QDISC configured in the kernel, you can add the TC Filter. This also
requires communication with the kernel over the netlink routing socket.

This time the netlink message consists of a header and an embedded TC message
with a kind and an options attribute. The options attribute contains a bpf
file descriptor attribute, a bpf name attribute, and a bpf flags attribute.

```
+---------------------------------------------------+
|                                    Netlink Header |
| type:    RTM_NEWTFILTER                           |
| flags:   NLM_F_REQUEST | NLM_F_CREAT              |
+---------------------------------------------------+
|                                        TC Message |
| family:  AF_UNSPEC                                |
| ifindex: if_nametoindex(if_name)                  |
| handle:  0                                        |
| parent:  TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS) |
| info:    TC_H_MAKE(0, htons(ETH_P_ALL))           |
+---------------------------------------------------+
|                                    Kind Attribute |
| type:    TCA_KIND                                 |
| data:    "bpf"                                    |
+---------------------------------------------------+
|                                 Options Attribute |
| type:    TCA_OPTIONS                              |
+---------------------------------------------------+
|                     BPF File Descriptor Attribute |
| type:    TCA_BPF_FD                               |
| data:    bpf_fd                                   |
+---------------------------------------------------+
|                                BPF Name Attribute |
| type:    TCA_BPF_NAME                             |
| data:    name                                     |
+---------------------------------------------------+
|                               BPF Flags Attribute |
| type:    TCA_BPF_FLAGS                            |
| data:    TCA_BPF_FLAG_ACT_DIRECT                  |
+---------------------------------------------------+
```

The header specifies the message type `RTM_NEWTFILTER` and the flags
`NLM_F_REQUEST | NLM_F_CREATE` that indicate a request to create a new TC
Filter.

```c
/* netlink message header */
hdr->nlmsg_len = NLMSG_LENGTH(sizeof(req.tcm));
hdr->nlmsg_len = NLMSG_ALIGN(NLMSG_LENGTH(sizeof(struct tcmsg))) + RTA_LENGTH(strlen("bpf") + 1);
hdr->nlmsg_pid = 0;
hdr->nlmsg_seq = 1;
hdr->nlmsg_type = RTM_NEWTFILTER;
hdr->nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE;
```

The TC message specifies the TC familiy `AF_UNSPEC`, the index of the network
interface where the filter should be added, the TC handle `0`, the TC parent
`TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS)` for ingress or
`TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_EGRESS)` for egress, and the TC info
`TC_H_MAKE(0, htons(ETH_P_ALL))`.

```c
/* tc message */
tcm->tcm_family = AF_UNSPEC;
tcm->tcm_ifindex = if_nametoindex(if_name);
tcm->tcm_handle = 0;
tcm->tcm_parent = TC_H_MAKE(TC_H_CLSACT, TC_H_MIN_INGRESS);
tcm->tcm_info = TC_H_MAKE(0, htons(ETH_P_ALL));
```

The kind attribute is a netlink routing attribute of type `TCA_KIND` and
contains the kind `bpf` as a string.

```c
/* kind attribute */
struct rtattr *kind_rta = (struct rtattr *) attr_buf;
kind_rta->rta_type = TCA_KIND;
kind_rta->rta_len = RTA_LENGTH(strnlen("bpf") + 1); // check offset
memcpy(RTA_DATA(kind_rta), "bpf", strlen("bpf") + 1);
```

The options attribute is netlink routing attribute of type `TCA_OPTIONS` and
contains the other bpf attributes.

```c
/* add options attribute */
struct rtattr *options_rta = (struct rtattr *)(attr_buf + RTA_ALIGN(kind_rta->rta_len));
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
struct rtattr *name_rta;
name_rta = (struct rtattr *)(((char *) options_rta) +
			     RTA_ALIGN(options_rta->rta_len));
name_rta->rta_type = TCA_BPF_NAME;
name_rta->rta_len = RTA_LENGTH(strlen("accept-all") + 1);
memcpy(RTA_DATA(name_rta), "accept-all", strlen("accept-all") + 1);

/* update options length */
options_rta->rta_len = RTA_ALIGN(options_rta->rta_len) +
	name_rta->rta_len;
```

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
hdr->nlmsg_len = NLMSG_ALIGN(req.hdr.nlmsg_len) +
	options_rta->rta_len;
```

## Removing the QDISC

You can undo the previous steps by deleting the QDISC on the network interface.
Again, this requires communication with the kernel over the netlink routing
socket.

The netlink message consists of a header and an embedded TC message with a TC
kind attribute.

```
+---------------------------------------------------+
|                                    Netlink Header |
| type:    RTM_DELQDISC                             |
| flags:   NLM_F_REQUEST                            |
+---------------------------------------------------+
|                                        TC Message |
| family:  AF_UNSPEC                                |
| ifindex: if_nametoindex(if_name)                  |
| handle:  TC_H_MAKE(TC_H_CLSACT, 0)                |
| parent:  TC_H_CLSACT                              |
+---------------------------------------------------+
|                                    Kind Attribute |
| type:    TCA_KIND                                 |
| data:    "clsact"                                 |
+---------------------------------------------------+
```

The header specifies the message type `RTM_DELQDISC` and the flag
`NLM_F_REQUEST` that indicate a request to delete a QDISC.

```c
/* netlink message header */
hdr->nlmsg_len = NLMSG_ALIGN(NLMSG_LENGTH(sizeof(struct tcmsg))) + RTA_LENGTH(strlen("clsact"));
hdr->nlmsg_pid = 0;
hdr->nlmsg_seq = 1;
hdr->nlmsg_type = RTM_DELQDISC;
hdr->nlmsg_flags = NLM_F_REQUEST;
```

The TC message specifies the TC family `AF_UNSPEC`, the index of the network
interface where the QDISC should be deleted, the TC handle
`TC_H_MAKE(TC_H_CLSACT, 0)` and the TC parent `TC_H_CLSACT`.

```c
/* tc message */
tcm->tcm_family = AF_UNSPEC;
tcm->tcm_ifindex = if_nametoindex(if_name);
tcm->tcm_handle = TC_H_MAKE(TC_H_CLSACT, 0);
tcm->tcm_parent = TC_H_CLSACT;
tcm->tcm_info = 0;
```

The kind attribute is a netlink routing attribute of type `TCA_KIND` and
contains the kind `clsact` as string.

```c
/* kind attribute */
struct rtattr *kind_rta = (struct rtattr *) attr_buf;
kind_rta->rta_type = TCA_KIND;
kind_rta->rta_len = RTA_LENGTH(strlen("clsact"));
memcpy(RTA_DATA(kind_rta), "clsact", strlen("clsact"));
```

## Conclusion?

TODO: add a conclusion?

## Code Snippets

The sections below contain C code snippets that show a dummy BPF program and a
complete implementation of tc-bpf attaching and detaching as described above.

### Dummy BPF Program

Dummy BPF program for TC that accepts all packets:

```c
/* bpf */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

/* tc */
#include <linux/pkt_cls.h>

/* set license to gpl */
char _license[] SEC("license") = "GPL";

/* accept all packets */
SEC("accept_all")
int _accept_all(struct __sk_buff *skb)
{
	return TC_ACT_OK;
}
```

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
