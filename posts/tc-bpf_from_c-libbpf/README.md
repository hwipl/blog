# TC-BPF from C Code with libbpf

This document describes how you can attach and detach an eBPF program for TC on
a network interface from C code, without using the `tc` tool, with libbpf. It
is an update for a [previous
document](https://github.com/hwipl/blog/tree/main/posts/tc-bpf_from_c) that was
started before libbpf had support for attaching and detaching TC-BPF programs.
The previous document shows a custom implementation that is similar to what
current libbpf versions offer. So, please refer to that document for more
details such as how TC, Netlink and BPF interact, or how to build and load a
TC-BPF program into the kernel.

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

After running the code above, you can check if attaching was successful, e.g.,
with the command line tools `tc` and `bpftool`.

With `tc` you can display TC QDISCs and TC Filters on your network interface.
The following example shows the respective commands and their output for the
attachment point `BPF_TC_INGRESS` and the network interface `eth0`. You can see
that the `_accept_all` program was attached successfully:

```console
$ tc qdisc show dev eth0
qdisc mq 0: root
qdisc fq_codel 0: parent :2 limit 10240p flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
qdisc fq_codel 0: parent :1 limit 10240p flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
qdisc clsact ffff: parent ffff:fff1
$ tc filter show dev eth0 ingress
filter protocol all pref 49152 bpf chain 0
filter protocol all pref 49152 bpf chain 0 handle 0x1 _accept_all:[33] direct-action not_in_hw id 33 tag a04f5eef06a7f555
```

With `bpftool` you can display BPF program information. The following example
shows the command for inspecting network related BPF program attachments and
its output. Again, you can see that the TC-BPF program `_accept_all` was
attached successfully on the network interface `eth0` for incoming packets:

```console
$ sudo bpftool net
xdp:

tc:
eth0(2) clsact/ingress _accept_all:[33] id 33

flow_dissector:

```

## Detaching

Detaching the TC-BPF program on a network interface with libbpf only requires a
single step: destroying the BPF TC hook.

In order to destroy the hook, you need to use a `struct bpf_tc_hook` again and
set its members `ifindex` and `attach_point`. The member `ifindex` specifies
the ID of the network interface you want to detach the BPF program from. The
member `attach_point` can be used in two different ways. If you specify
`BPF_TC_EGRESS` or `BPF_TC_INGRESS`, only the respective TC Filter for outgoing
or incoming packets will be removed. The underlying TC QDISC will remain on the
network interface. If you specify `BPF_TC_EGRESS | BPF_TC_INGRESS`, both the TC
Filter and the underlying TC QDISC will be removed from the network interface.
A call to the function `bpf_tc_hook_destroy()` finally destroys the hook:

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

This removes the TC QDISC and the TC Filter rule for the BPF program you
created in the previous section. This also removes the loaded BPF program.

After running the code above, you can check if detaching was successful, e.g.,
with the command line tools `tc` and `bpftool`, as described in the previous
section.

With `tc` you can see that the program is not attached to `eth0` any more:

```console
$ tc qdisc show dev eth0
qdisc mq 0: root
qdisc fq_codel 0: parent :2 limit 10240p flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
qdisc fq_codel 0: parent :1 limit 10240p flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
$ tc filter show dev eth0 ingress
```

With `bpftool` you can also see that there is no more TC-BPF program loaded and
attached to a network interface:

```console
$ sudo bpftool net
xdp:

tc:

flow_dissector:

```

## Conclusion

This document describes how you can attach and detach TC-BPF programs on
network interfaces with libbpf in C code without using the `tc` tool. Attaching
and detaching TC-BPF programs requires only a few steps compared to a manual
implementation: in order to attach a program, you create a TC BPF hook on the
network interface for incoming or outgoing packets; to detach the program
again, you destroy the TC BPF hook. All sections contain code examples you can
use for your own implementation. Additionally, you can find a complete
implementation in the appendix below.

## Appendix: Code

The sections below contain C code of attaching and detaching programs, build
instructions and usage examples. You can also find the source code, including a
dummy TC-BPF program, here:

- [Dummy BPF Program](https://github.com/hwipl/snippets-c/blob/main/bpf/tc-accept.c)
- [TC-BPF Attaching](https://github.com/hwipl/snippets-c/blob/main/bpf/tc-attach2.c)
- [TC-BPF Detaching](https://github.com/hwipl/snippets-c/blob/main/bpf/tc-detach2.c)

Note: for more information on the dummy TC-BPF program, how to build it and how
to load it, please see the [previous
document](https://github.com/hwipl/blog/tree/main/posts/tc-bpf_from_c) or the
[documentation](https://github.com/hwipl/snippets-c/blob/main/bpf/README.md) of
the source code.

### Attaching a BPF Program on a Network Interface

Include headers:

```c
/* bpf */
#include <bpf/libbpf.h>

/* if_nametoindex() */
#include <net/if.h>
```

Function for loading the BPF program in the file identified by the filename in
`file` and returning the file descriptor of the loaded program:

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

You can, for example, save the code in `tc-attach2.c` and then build it as
`tc-attach2` with clang:

```console
$ clang tc-attach2.c -o tc-attach2 -l bpf
```

You can then attach the BPF program `tc-accept.o`, for example, to
the network interface `eth0` with the following command:

```console
$ sudo ./tc-attach2 tc-accept.o eth0
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

You can, for example, save the code in `tc-detach2.c` and then build it as
`tc-detach2` with clang:

```console
$ clang tc-detach2.c -o tc-detach2
```

You can then detach the previously attached BPF program, for example, on
network interface `eth0` with the following command:

```console
$ sudo ./tc-detach2 eth0
```
