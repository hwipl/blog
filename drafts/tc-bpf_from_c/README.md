# TC-BPF from C code

## code snippets

### loading a bpf program on a network interface

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
