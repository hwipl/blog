# TC-BPF from C Code with libbpf

This document describes how you can attach and detach an eBPF program for TC on
a network interface from C code, without using the `tc` tool, with libbpf. It
is an update for a previous document [1] that was started before libbpf had
support for attaching and detaching TC-BPF programs. It shows a custom
implementation that is similar to what current libbpf versions offer. So,
please refer to [1] for more details such as how TC, Netlink and BPF interact.

[1]: https://github.com/hwipl/blog/tree/main/posts/tc-bpf_from_c
