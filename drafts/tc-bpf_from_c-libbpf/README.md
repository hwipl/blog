# TC-BPF from C Code with libbpf

This document describes how you can attach and detach an eBPF program for TC on
a network interface from C code, without using the `tc` tool, with libbpf. It
is an update for a previous document [1] that was started before libbpf had
support for attaching and detaching TC-BPF programs. It shows a custom
implementation that is similar to what current libbpf versions offer. So,
please refer to [1] for more details such as how TC, Netlink and BPF interact.

[1]: https://github.com/hwipl/blog/tree/main/posts/tc-bpf_from_c

## Attaching

Attaching a TC-BPF program on a network interface with libbpf consists of two
steps: (1) creating a BPF TC hook and then (2) attaching the program using this
hook and additional BPF TC options.

In order to create the hook, you need to use a `struct bpf_tc_hook` and set its
members `ifindex` and `attach_point`. The member `ifindex` specifies the ID of
the network interface you want to attach the BPF program on. The member
`attach_point` specifies if the program should be used for outgoing
(`BPF_TC_EGRESS`) or incoming (`BPF_TC_INGRESS`) packets on the network
interface. A call to the function `bpf_tc_hook_create()` finally creates the
hook:

```c
// create bpf hook
struct bpf_tc_hook hook;
memset(&hook, 0, sizeof(hook));
hook.sz			= sizeof(struct bpf_tc_hook);
hook.ifindex		= if_nametoindex(if_name);
hook.attach_point	= BPF_TC_INGRESS; // BPF_TC_EGRESS, BPF_TC_CUSTOM
bpf_tc_hook_create(&hook);
```

This creates a TC QDISC on the network interface.

In order to attach the program using the hook created above, you need to
provide additional BPF TC options. For this, you can use a `struct bpf_tc_opts`
and set its member `prog_fd`. The member `prog_fd` identifies the loaded BPF
program via its file descriptor. A call to the function `bpf_tc_attach` with
the hook and the options as parameters then attaches the BPF program to the
network interface:

```c
// attach bpf program
struct bpf_tc_opts opts;
memset(&opts, 0, sizeof(opts));
opts.sz		= sizeof(struct bpf_tc_opts);
opts.prog_fd	= prog_fd;
bpf_tc_attach(&hook, &opts);
```

This creates a TC Filter rule for the BPF program in the TC QDISC you created
in the previous step.

TODO: add `tc` and `bpftool` commands to check if attaching was successful?

## Detaching

Destroy hook:

```c
// destroy bpf hook
struct bpf_tc_hook hook;
memset(&hook, 0, sizeof(hook));
hook.sz			= sizeof(struct bpf_tc_hook);
hook.ifindex		= if_nametoindex(if_name);
// specify BPF_TC_INGRESS | BPF_TC_EGRESS to delete the qdisc;
// specifying only BPF_TC_INGRESS or only BPF_TC_EGRESS
// deletes the respective filter only
hook.attach_point	= BPF_TC_INGRESS | BPF_TC_EGRESS;
bpf_tc_hook_destroy(&hook);
```

## Appendix: Code

- [Dummy BPF Program](https://github.com/hwipl/snippets-c/blob/main/bpf/tc-accept.c)
- [TC-BPF Attaching](https://github.com/hwipl/snippets-c/blob/main/bpf/tc-attach2.c)
- [TC-BPF Detaching](https://github.com/hwipl/snippets-c/blob/main/bpf/tc-detach2.c)

### Attaching a BPF Program on a Network Interface

Include headers:

```c
/* bpf */
#include <bpf/libbpf.h>

/* if_nametoindex() */
#include <net/if.h>
```

Function for loading the BPF program in the file identified by the filename in
`file` and returning the file descriptor of the loaded porgram:

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

Function for attaching the BPF program with its file descriptor `prog_fd` on
the interface with the name `if_name`:

```c
/* attach bpf program in prog_fd to network interface identified by if_name */
int attach_bpf(const char *if_name, int prog_fd) {
	int rc;

	// create bpf hook
	struct bpf_tc_hook hook;
	memset(&hook, 0, sizeof(hook));
	hook.sz			= sizeof(struct bpf_tc_hook);
	hook.ifindex		= if_nametoindex(if_name);
	hook.attach_point	= BPF_TC_INGRESS; // BPF_TC_EGRESS, BPF_TC_CUSTOM
	rc = bpf_tc_hook_create(&hook);
	if (rc) {
		printf("Error creating tc hook\n");
		return rc;
	}

	// attach bpf program
	struct bpf_tc_opts opts;
	memset(&opts, 0, sizeof(opts));
	opts.sz		= sizeof(struct bpf_tc_opts);
	opts.prog_fd	= prog_fd;
	rc = bpf_tc_attach(&hook, &opts);
	if (rc) {
		printf("Error during tc attach\n");
		return rc;
	}

	return 0;
}
```

Main function that combines the previous functions and reads the filename of
the BPF program from its first command line argument and the name of the
network interface from the second command line argument:

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

	/* attach bpf program */
	if (attach_bpf(if_name, prog_fd)) {
		printf("Error attaching bpf program\n");
		return -1;
	}

	return 0;
}
```

### Detaching BPF Programs on a Network Interface

Include headers:

```c
/* bpf */
#include <bpf/libbpf.h>

/* if_nametoindex() */
#include <net/if.h>
```

Function for detaching the BPF program on the interface with the name
`if_name`:

```c
/* detach bpf program from network interface identified by if_name */
int detach_bpf(const char *if_name) {
	int rc;

	// destroy bpf hook
	struct bpf_tc_hook hook;
	memset(&hook, 0, sizeof(hook));
	hook.sz			= sizeof(struct bpf_tc_hook);
	hook.ifindex		= if_nametoindex(if_name);
	// specify BPF_TC_INGRESS | BPF_TC_EGRESS to delete the qdisc;
	// specifying only BPF_TC_INGRESS or only BPF_TC_EGRESS
	// deletes the respective filter only
	hook.attach_point	= BPF_TC_INGRESS | BPF_TC_EGRESS;
	rc = bpf_tc_hook_destroy(&hook);
	if (rc) {
		printf("Error destroying tc hook\n");
		return rc;
	}

	return 0;
}
```

Main function that uses the previous function and reads the name of the network
interface from the first command line argument:

```c
int main(int argc, char **argv) {
	/* handle command line arguments */
	if (argc < 2) {
		return -1;
	}
	const char *if_name = argv[1];

	/* detach bpf program */
	if (detach_bpf(if_name)) {
		printf("Error detaching bpf program\n");
		return -1;
	}

	return 0;
}
```
