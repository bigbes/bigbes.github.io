+++
title = "eBPF for SSL packet inspections"
tags  = "ebpf"
date  = 2024-07-07T11:30:00+03:00
draft = false

description = "..."
+++

{{ .Page.TableOfContents }}

As a retrospection of task in a process, i've had a task to intercept http
traffic sent by different processes with encryption using openssl. Main points
of that task (related to eBPF) were:

* eBPF uprobes for SSL_* functions (to obtain data that was sent/recived)
* eBPF kprobes for tcp_sendmsg/tcp_recvmsg (to obtain src/dst hosts/ports)
* a lot of tracepoints (primarily to map fd)

## Setting development environment

I'm using WSL2 with arch on windows platform, so we're having env problems :D
First take was clion - but, sadly, I've failed to set up make projets >_<

Second take was with vim - `github/copilot.vim`+`ycm-core/YouCompleteMe` works
just fine (except `.h` files were interpreted as `c++` files and large
`vmlinux.h` processing was exceptionally slow, freezing vim for forever).
As a possible workaround `vmlinux_part.h` was created, with a lot of parts
to be deleted. Not convenient :(

Third take was with Sublime as editor - `c lsp plugin` failed to use clangd
(some extended problems with host-vm :) ), but LSP-copilot works. 50/50
usability!

## File structure

```console
$ tree
├── TODO.md
├── include
│   ├── bpf_endian.h
│   ├── bpf_helper_defs.h
│   ├── bpf_helpers.h
│   ├── bpf_tracing.h
│   ├── counter.h
│   ├── directions.h
│   ├── helpers.h
│   ├── logger.h
│   ├── pid_checker.h
│   ├── structures.h
│   └── vmlinux.h
├── kprobe.c
├── makefile
├── module.c
├── nodemon.json
├── openssl-args.c
├── openssl-stats.c
├── openssl-store.c
├── openssl.c
├── prepare-env.py
└── tracepoint.c
```

so what's in include directories?

* `vmlinux.h` - contains all linux kernel definitions (types, structures, enums, etc)\
  generated using `bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h`
* ebpf helpers: (`bpf_*.h` - https://github.com/libbpf/libbpf/blob/v0.6.1/src/bpf_tracing.h)
* self-written helpers ...


and what about source file structure?

* `module.c` - main file point, all defines and includes of all functions
  ```c
  //go:build ignore

  #define PID_CHECK_STATE 0
  #define DEBUG_PRINT 1
  #define OPENSSL_VERSION 0x30300000

  #include "openssl.c"
  #include "tracepoint.c"
  #include "kprobe.c"

  static char _license[] SEC("license") = "Dual BSD/GPL";
  ```
  it's used for convenience of building of single module using `bpf2go`
* `tracepoint.c`/`openssl.c`/`kprobe.c` - files with all ebpf entrypoints.
* `openssl-*.c` files are used to store helpers.

## eBPF primer for uprobes/kprobes/tracepoints

just a note: difference between `k(u)probe` and `k(u)retprobe` is a time when
callback is called. `*probe` is called on entering function and `*retprobe` is
called on function exit. Input arguments can be accessed only in `*probe`,
output argement can be accessed only in `*retprobe`.

Naming in `SEC` macro is vital, so "thinkaboutitman"!

### defining eBPF entrypoints and other functions

* for eBPF we're using `SEC()` macro:
    `SEC("u[ret]probe/<name>")`\
    `SEC("k[ret]probe/<name>")`\
    `SEC("tracepoint/syscalls/sys_[enter/exit]_<name>")`\
    are examples
* for non-ebpf functions we're using `static INLINE_FUNC` macro to embed
  all functions (since ebpf can't make calls to other functions we must inline
  it's code inside [exception are tail call which must be called explicitly])

Examples:

```c
static INLINE_FUNC
direction_t get_direction(u32 fd)
{
    /* ... */
}

SEC("uprobe/SSL_read")
int uprobe_ssl_read(struct pt_regs *ctx)
{
    /* ... */
}

SEC("tracepoint/syscalls/sys_enter_accept4")
void sys_enter_accept4(struct sys_enter_accept4_ctx *ctx)
{
    /* ... */
}
```

### accessing return variables and call arguments (uprobes/kprobes)

Functions call arguments are accessed using `PT_REGS_PARM<pos>` and `PT_REGS_RC`.
As a convience helpers we're using macroses `BPF_UPROBE(<name>, <args...>)`,
`BPF_URETPROBE(<name>, <rv>)`, and `BPF_KPROBE`/`BPF_KRETPROBE`

Examples:

```c
SEC("uprobe/SSL_read")
int BPF_UPROBE(uprobe_ssl_read, void *ssl, void *buffer, int num)
{
    /* ... */
}

SEC("uretprobe/SSL_read")
int BPF_URETPROBE(uretprobe_ssl_read, int rv)
{
    /* ... */
}
```

### defining tracepoints

Tracepoint can be different types, for example `syscalls`, `oom`, `tcp`, `net`.
But for now we're investigating only `syscalls`.

To understand arguments of tracepoint - we need to cat sys path to investigate
"what's inside":

```c

struct sys_common_ctx {
	u16 common_type;
	u8  common_flags;
	u8  common_preempt_count;
	s32 common_pid;
	s32 __syscall_nr;
};

// cat /sys/kernel/tracing/events/syscalls/sys_enter_connect/format
//
// name: sys_enter_connect
// ID: 2407
// format:
//         field:unsigned short common_type;       		offset:0;       size:2; signed:0;
//         field:unsigned char common_flags;       		offset:2;       size:1; signed:0;
//         field:unsigned char common_preempt_count;    offset:3;       size:1; signed:0;
//         field:int common_pid;              			offset:4;       size:4; signed:1;
//         field:int __syscall_nr; 						offset:8;       size:4; signed:1;
//
//         field:int fd;  							 	offset:16;      size:8; signed:0;
//         field:struct sockaddr * uservaddr;      		offset:24;      size:8; signed:0;
//         field:int addrlen;      						offset:32;      size:8; signed:0;
//
//

struct sys_enter_connect_ctx {
	struct sys_common_ctx ctx;
	u64 				  fd;
	struct sockaddr		 *uservaddr;
	u64 				  uservaddr_len;
};

SEC("tracepoint/<type>/sys_enter_connect")
void sys_enter_connect(struct sys_enter_connect_ctx *ctx)
{
    /* ... */
}

// cat /sys/kernel/tracing/events/syscalls/sys_exit_connect/format
//
// name: sys_exit_connect
// ID: 2406
// format:
//         field:unsigned short common_type;       		offset:0;       size:2; signed:0;
//         field:unsigned char common_flags;       		offset:2;       size:1; signed:0;
//         field:unsigned char common_preempt_count;    offset:3;       size:1; signed:0;
//         field:int common_pid;   						offset:4;       size:4; signed:1;
//         field:int __syscall_nr; 						offset:8;       size:4; signed:1;
//
//         field:long ret; 								offset:16;      size:8; signed:1;

struct sys_exit_connect_ctx {
	struct sys_common_ctx ctx;
	s64 				  ret;
};

SEC("tracepoint/syscalls/sys_exit_connect")
void sys_exit_connect(struct sys_exit_connect_ctx *ctx)
{
    /* ... */
}
```

### defining and accessing maps

TODO

### ringbuf as special type of map

TODO

### using and accessing from go code

TODO

## eBPF for SSL packet interception flow

TODO

## links

* [learning eBPF book](https://cilium.isovalent.com/hubfs/Learning-eBPF%20-%20Full%20book.pdf)
* [golang ebpf lib](https://github.com/cilium/ebpf)
* [features by kernel versions](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)
* https://docs.kernel.org/6.6/bpf/index.html
* https://prototype-kernel.readthedocs.io/en/latest/bpf/index.html
