# Linux BPF / eBPF Cheatsheet

A complete reference for Berkeley Packet Filter (classic BPF) and extended BPF (eBPF) — the Linux kernel's programmable in-kernel virtual machine for networking, observability, security, and tracing.

---

## Overview

| Feature | cBPF (classic) | eBPF (extended) |
|---------|---------------|-----------------|
| Introduced | Linux 2.5 (1997 BSD) | Linux 3.18 (2014) |
| Registers | 2 (A, X) | 11 (r0–r10) |
| Register width | 32-bit | 64-bit |
| Map support | No | Yes |
| Kernel helpers | No | Yes (200+) |
| JIT compiled | Yes (limited) | Yes (all arches) |
| Use cases | `tcpdump`, `seccomp` | Everything |
| Verifier | Basic | Full safety verifier |

---

## eBPF Architecture

```
User Space                        Kernel Space
─────────────────────────────     ─────────────────────────────────────
  Source (.c)                       Verifier
      │                                │
  Clang/LLVM                       JIT Compiler
      │                                │
  BPF bytecode ──── bpf() syscall ──► BPF VM / Maps
      │                                │
  Loader (libbpf,                  Hook Points:
  bpftool, etc.)                     kprobes / uprobes
                                      tracepoints
                                      XDP / TC / socket
                                      cgroups / LSM
```

---

## Hook Points (Program Types)

### Networking

| Type | Attach point | Use case |
|------|-------------|---------|
| `BPF_PROG_TYPE_XDP` | Network driver (earliest RX) | DDoS mitigation, load balancing |
| `BPF_PROG_TYPE_SCHED_CLS` | TC ingress/egress | Traffic shaping, filtering |
| `BPF_PROG_TYPE_SCHED_ACT` | TC action | Packet modification |
| `BPF_PROG_TYPE_SOCKET_FILTER` | Socket | Packet filtering (tcpdump) |
| `BPF_PROG_TYPE_SOCK_OPS` | TCP socket events | TCP tuning per-connection |
| `BPF_PROG_TYPE_SK_SKB` | Sockmap | Kernel-space socket proxying |
| `BPF_PROG_TYPE_SK_MSG` | Sockmap msg | Message-level socket redirection |
| `BPF_PROG_TYPE_SK_LOOKUP` | Socket lookup | Custom socket selection |
| `BPF_PROG_TYPE_LWT_*` | Lightweight tunnel | Encapsulation |

### Tracing & Observability

| Type | Attach point | Use case |
|------|-------------|---------|
| `BPF_PROG_TYPE_KPROBE` | Any kernel function | Dynamic kernel tracing |
| `BPF_PROG_TYPE_TRACEPOINT` | Static tracepoints | Stable kernel event hooks |
| `BPF_PROG_TYPE_PERF_EVENT` | Perf events | CPU profiling, PMU events |
| `BPF_PROG_TYPE_RAW_TRACEPOINT` | Raw tracepoints | Lower-overhead tracing |
| `BPF_PROG_TYPE_FENTRY` / `FEXIT` | BTF-based fentry/fexit | Fast function entry/exit hooks |
| `BPF_PROG_TYPE_TRACING` | BTF tracing | kfunc, iter, raw_tp |

### Security

| Type | Attach point | Use case |
|------|-------------|---------|
| `BPF_PROG_TYPE_LSM` | LSM hooks | Security policy enforcement |
| `BPF_PROG_TYPE_SECCOMP` | Seccomp | Syscall filtering |
| `BPF_PROG_TYPE_CGROUP_SKB` | Cgroup network | Per-cgroup packet filtering |
| `BPF_PROG_TYPE_CGROUP_SOCK` | Cgroup socket | Socket policy per-cgroup |
| `BPF_PROG_TYPE_CGROUP_DEVICE` | Cgroup device | Device access control |

---

## XDP Return Codes

| Code | Value | Action |
|------|-------|--------|
| `XDP_DROP` | 1 | Drop packet silently |
| `XDP_PASS` | 2 | Pass to normal network stack |
| `XDP_TX` | 3 | Transmit back out same interface |
| `XDP_REDIRECT` | 4 | Redirect to another interface/CPU/socket |
| `XDP_ABORTED` | 0 | Drop + trace (error path) |

### XDP Attachment Modes

| Mode | Flag | Description |
|------|------|-------------|
| Generic (skb) | `XDP_FLAGS_SKB_MODE` | Slowest — works on any driver |
| Native | `XDP_FLAGS_DRV_MODE` | Driver-level — fast |
| Offloaded | `XDP_FLAGS_HW_MODE` | Runs on NIC hardware |

---

## TC (Traffic Control) Return Codes

| Code | Action |
|------|--------|
| `TC_ACT_OK` | Pass packet |
| `TC_ACT_SHOT` | Drop packet |
| `TC_ACT_REDIRECT` | Redirect packet |
| `TC_ACT_PIPE` | Pass to next action |
| `TC_ACT_UNSPEC` | Use default action |

---

## BPF Map Types

Maps are the primary mechanism for sharing data between BPF programs and user space.

| Map Type | Description | Key | Value |
|----------|-------------|-----|-------|
| `BPF_MAP_TYPE_HASH` | Hash table | Any | Any |
| `BPF_MAP_TYPE_ARRAY` | Fixed-size array | `u32` index | Any |
| `BPF_MAP_TYPE_PERCPU_HASH` | Per-CPU hash table | Any | Any |
| `BPF_MAP_TYPE_PERCPU_ARRAY` | Per-CPU array | `u32` index | Any |
| `BPF_MAP_TYPE_LRU_HASH` | LRU hash (auto-evict) | Any | Any |
| `BPF_MAP_TYPE_LPM_TRIE` | Longest prefix match | IP prefix | Any |
| `BPF_MAP_TYPE_RINGBUF` | Ring buffer (preferred for events) | — | Any |
| `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | Perf ring buffer (legacy) | CPU index | fd |
| `BPF_MAP_TYPE_PROG_ARRAY` | Array of BPF programs | `u32` index | prog fd |
| `BPF_MAP_TYPE_STACK_TRACE` | Kernel stack traces | Stack id | Addrs |
| `BPF_MAP_TYPE_CGROUP_ARRAY` | Cgroup fds | `u32` index | cgroup fd |
| `BPF_MAP_TYPE_SOCKMAP` | Socket map | `u32` index | socket fd |
| `BPF_MAP_TYPE_SOCKHASH` | Socket hash map | Any | socket fd |
| `BPF_MAP_TYPE_DEVMAP` | Device redirect map | `u32` index | ifindex |
| `BPF_MAP_TYPE_CPUMAP` | CPU redirect map | CPU id | config |
| `BPF_MAP_TYPE_XSKMAP` | AF_XDP socket map | `u32` index | xsk fd |
| `BPF_MAP_TYPE_QUEUE` | FIFO queue | — | Any |
| `BPF_MAP_TYPE_STACK` | LIFO stack | — | Any |
| `BPF_MAP_TYPE_BLOOM_FILTER` | Probabilistic membership | — | Any |
| `BPF_MAP_TYPE_INODE_STORAGE` | Per-inode local storage | inode | Any |
| `BPF_MAP_TYPE_TASK_STORAGE` | Per-task local storage | task | Any |
| `BPF_MAP_TYPE_SK_STORAGE` | Per-socket local storage | socket | Any |

### Map Operations (user space — libbpf)

```c
// Create
int fd = bpf_map_create(BPF_MAP_TYPE_HASH, "mymap",
                        sizeof(key), sizeof(value), max_entries, NULL);

// Lookup
bpf_map_lookup_elem(fd, &key, &value);

// Insert / Update
bpf_map_update_elem(fd, &key, &value, BPF_ANY);   // create or update
bpf_map_update_elem(fd, &key, &value, BPF_NOEXIST); // create only
bpf_map_update_elem(fd, &key, &value, BPF_EXIST);   // update only

// Delete
bpf_map_delete_elem(fd, &key);

// Iterate
bpf_map_get_next_key(fd, &key, &next_key);

// Batch operations (kernel 5.6+)
bpf_map_lookup_batch(fd, ...);
bpf_map_update_batch(fd, ...);
bpf_map_delete_batch(fd, ...);
```

### Map Operations (BPF program — kernel side)

```c
// Lookup (returns pointer or NULL)
value = bpf_map_lookup_elem(&my_map, &key);

// Update
bpf_map_update_elem(&my_map, &key, &value, BPF_ANY);

// Delete
bpf_map_delete_elem(&my_map, &key);

// Push/pop (queue/stack maps)
bpf_map_push_elem(&my_map, &value, BPF_ANY);
bpf_map_pop_elem(&my_map, &value);
bpf_map_peek_elem(&my_map, &value);
```

---

## BPF Program Structure (C / libbpf)

### Minimal XDP program

```c
// xdp_drop.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_prog(struct xdp_md *ctx)
{
    return XDP_DROP;
}

char LICENSE[] SEC("license") = "GPL";
```

### XDP program with map

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, __u32);     // blocked IPv4 address
    __type(value, __u64);   // packet counter
} blocklist SEC(".maps");

SEC("xdp")
int xdp_filter(struct xdp_md *ctx)
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    if (bpf_ntohs(eth->h_proto) != ETH_P_IP)
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    __u64 *count = bpf_map_lookup_elem(&blocklist, &ip->saddr);
    if (count) {
        __sync_fetch_and_add(count, 1);
        return XDP_DROP;
    }

    return XDP_PASS;
}

char LICENSE[] SEC("license") = "GPL";
```

### Kprobe / tracepoint program

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

SEC("kprobe/sys_execve")
int kprobe_execve(struct pt_regs *ctx)
{
    __u64 pid_tgid = bpf_get_current_pid_tgid();
    __u32 pid = pid_tgid >> 32;
    char comm[16];
    bpf_get_current_comm(comm, sizeof(comm));
    bpf_printk("execve called: pid=%d comm=%s\n", pid, comm);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

### Ring buffer event output

```c
struct event {
    __u32 pid;
    char  comm[16];
    char  filename[256];
};

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} events SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_openat")
int trace_openat(struct trace_event_raw_sys_enter *ctx)
{
    struct event *e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e) return 0;

    e->pid = bpf_get_current_pid_tgid() >> 32;
    bpf_get_current_comm(e->comm, sizeof(e->comm));
    bpf_probe_read_user_str(e->filename, sizeof(e->filename),
                            (void *)ctx->args[1]);

    bpf_ringbuf_submit(e, 0);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

---

## Common BPF Helper Functions

### Process / Task

```c
bpf_get_current_pid_tgid()        // returns (tgid << 32 | pid)
bpf_get_current_uid_gid()         // returns (gid << 32 | uid)
bpf_get_current_comm(buf, size)   // current process name
bpf_get_current_task()            // pointer to current task_struct
bpf_get_current_cgroup_id()       // cgroup ID of current task
```

### Time

```c
bpf_ktime_get_ns()                // monotonic clock in nanoseconds
bpf_ktime_get_boot_ns()           // boot-relative time in ns
bpf_ktime_get_tai_ns()            // TAI clock in ns
bpf_jiffies64()                   // kernel jiffies counter
```

### Memory / Probing

```c
bpf_probe_read_kernel(dst, size, src)       // safe kernel memory read
bpf_probe_read_user(dst, size, src)         // safe user memory read
bpf_probe_read_kernel_str(dst, size, src)   // kernel string read
bpf_probe_read_user_str(dst, size, src)     // user string read
```

### Output / Debugging

```c
bpf_printk(fmt, ...)              // print to /sys/kernel/debug/tracing/trace_pipe
bpf_perf_event_output(ctx, map, flags, data, size)  // perf ring output
bpf_ringbuf_reserve(map, size, flags)       // reserve ring buffer space
bpf_ringbuf_submit(data, flags)             // submit ring buffer entry
bpf_ringbuf_discard(data, flags)            // discard ring buffer entry
bpf_ringbuf_output(map, data, size, flags)  // reserve+submit in one call
```

### Networking

```c
bpf_skb_load_bytes(skb, offset, to, len)    // read packet bytes
bpf_skb_store_bytes(skb, offset, from, len, flags)
bpf_l3_csum_replace(skb, offset, from, to, flags)
bpf_l4_csum_replace(skb, offset, from, to, flags)
bpf_skb_adjust_room(skb, len_diff, mode, flags)
bpf_clone_redirect(skb, ifindex, flags)
bpf_redirect(ifindex, flags)
bpf_redirect_map(map, key, flags)
bpf_xdp_adjust_head(ctx, delta)
bpf_xdp_adjust_tail(ctx, delta)
bpf_xdp_adjust_meta(ctx, delta)
```

### Map Helpers

```c
bpf_map_lookup_elem(map, key)
bpf_map_update_elem(map, key, value, flags)
bpf_map_delete_elem(map, key)
bpf_map_push_elem(map, value, flags)
bpf_map_pop_elem(map, value)
bpf_for_each_map_elem(map, callback, ctx, flags)
```

### Tail Calls & Program Chaining

```c
bpf_tail_call(ctx, prog_array_map, index)   // jump to another BPF program
bpf_loop(nr_loops, callback, ctx, flags)    // bounded loop helper
```

### Miscellaneous

```c
bpf_get_stackid(ctx, map, flags)            // capture stack trace
bpf_get_stack(ctx, buf, size, flags)        // copy stack frames
bpf_get_numa_node_id()
bpf_get_smp_processor_id()
bpf_spin_lock(lock)
bpf_spin_unlock(lock)
bpf_timer_init(timer, map, clockid)
bpf_timer_set_callback(timer, callback)
bpf_timer_start(timer, nsecs, flags)
bpf_timer_cancel(timer)
```

---

## Compilation

### Compile BPF program with Clang

```bash
clang -O2 -g -target bpf \
    -D__TARGET_ARCH_x86 \
    -I/usr/include/x86_64-linux-gnu \
    -c xdp_prog.c -o xdp_prog.o
```

### Generate BTF (BPF Type Format) — required for CO-RE

```bash
clang -O2 -g -target bpf \
    -D__TARGET_ARCH_x86 \
    -c prog.c -o prog.o

# Inspect BTF
bpftool btf dump file prog.o
```

### Generate vmlinux.h (BTF header for CO-RE)

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

### Skeleton generation (libbpf)

```bash
bpftool gen skeleton prog.o > prog.skel.h
```

---

## bpftool Reference

`bpftool` is the primary CLI for inspecting and managing BPF programs, maps, and links.

### Programs

```bash
bpftool prog list                          # list all loaded BPF programs
bpftool prog list --json | jq .            # JSON output
bpftool prog show id <id>                  # show program details
bpftool prog dump xlated id <id>           # disassemble (human-readable)
bpftool prog dump jited id <id>            # disassemble JIT output
bpftool prog dump xlated id <id> visual    # visual CFG (requires graphviz)
bpftool prog load prog.o /sys/fs/bpf/prog  # pin program to BPF filesystem
bpftool prog pin id <id> /sys/fs/bpf/prog  # pin by ID
bpftool prog tracelog                      # tail bpf_printk output
```

### Maps

```bash
bpftool map list                           # list all BPF maps
bpftool map show id <id>                   # map details
bpftool map dump id <id>                   # dump all map contents
bpftool map lookup id <id> key hex <key>   # lookup a key
bpftool map update id <id> key hex <key> value hex <val>
bpftool map delete id <id> key hex <key>
bpftool map pin id <id> /sys/fs/bpf/mymap  # pin map to BPF filesystem
bpftool map create /sys/fs/bpf/mymap       # create pinned map
    type hash key 4 value 8 entries 1024 name mymap
bpftool map freeze id <id>                 # make map read-only
```

### Links & Attachments

```bash
bpftool link list                          # list BPF links
bpftool link show id <id>
bpftool link pin id <id> /sys/fs/bpf/link
bpftool link detach id <id>
```

### Network

```bash
bpftool net list                           # list network-attached BPF progs
bpftool net show dev eth0                  # show progs on interface
```

### BTF

```bash
bpftool btf list                           # list BTF objects
bpftool btf show id <id>
bpftool btf dump id <id>                   # dump BTF type info
bpftool btf dump file /sys/kernel/btf/vmlinux format c  # generate vmlinux.h
```

### Cgroups

```bash
bpftool cgroup list /sys/fs/cgroup/        # list cgroup-attached programs
bpftool cgroup attach /sys/fs/cgroup/ ingress id <prog_id>
bpftool cgroup detach /sys/fs/cgroup/ ingress id <prog_id>
bpftool cgroup tree                        # show full cgroup tree
```

### Feature Detection

```bash
bpftool feature probe                      # probe kernel BPF features
bpftool feature probe kernel               # kernel feature probe
bpftool feature probe dev eth0             # device XDP feature probe
```

### Skeleton & Code Generation

```bash
bpftool gen skeleton prog.o > prog.skel.h          # C skeleton
bpftool gen min_core_btf vmlinux.btf min.btf prog.o  # minimal BTF for distro
bpftool gen object prog.linked.o prog.o            # link BPF objects
```

---

## iproute2 / tc — Loading XDP and TC Programs

### XDP via ip link

```bash
# Attach XDP program (native mode)
ip link set dev eth0 xdp obj prog.o sec xdp

# Attach in generic/skb mode
ip link set dev eth0 xdpgeneric obj prog.o sec xdp

# Attach in offload mode
ip link set dev eth0 xdpoffload obj prog.o sec xdp

# Detach XDP program
ip link set dev eth0 xdp off
ip link set dev eth0 xdpgeneric off

# View attached XDP program
ip link show dev eth0
```

### TC (Traffic Control)

```bash
# Add clsact qdisc (required for BPF TC programs)
tc qdisc add dev eth0 clsact

# Attach BPF program on ingress
tc filter add dev eth0 ingress bpf obj prog.o sec tc direct-action

# Attach on egress
tc filter add dev eth0 egress bpf obj prog.o sec tc direct-action

# List filters
tc filter list dev eth0 ingress
tc filter list dev eth0 egress

# Delete filter
tc filter del dev eth0 ingress

# Delete qdisc (removes all filters)
tc qdisc del dev eth0 clsact
```

---

## BPF Filesystem (bpffs)

Pin programs and maps to the BPF virtual filesystem for persistence across program restarts.

```bash
# Mount bpffs (usually auto-mounted at /sys/fs/bpf)
mount -t bpf bpffs /sys/fs/bpf

# Pin a program
bpftool prog pin id <id> /sys/fs/bpf/my_prog

# Pin a map
bpftool map pin id <id> /sys/fs/bpf/my_map

# Load and pin in one step
bpftool prog load prog.o /sys/fs/bpf/my_prog \
    map name mymap pinned /sys/fs/bpf/my_map

# List pinned objects
ls /sys/fs/bpf/
bpftool prog list pinned /sys/fs/bpf/my_prog

# Remove pinned object (program continues running until all refs drop)
rm /sys/fs/bpf/my_prog
```

---

## Tracing Interfaces

### bpf_printk output

```bash
# View trace output (blocking)
cat /sys/kernel/debug/tracing/trace_pipe

# One-shot snapshot
cat /sys/kernel/debug/tracing/trace

# Clear trace buffer
echo > /sys/kernel/debug/tracing/trace
```

### Perf events

```bash
# Run perf with BPF
perf record -e bpf-output/no-inherit,name=evt/ \
            -e bpf_trace:* -- sleep 10
perf script
```

### tracefs

```bash
ls /sys/kernel/debug/tracing/events/     # available tracepoints
cat /sys/kernel/debug/tracing/available_filter_functions  # kprobeable functions
cat /sys/kernel/debug/tracing/kprobe_events               # active kprobes
cat /sys/kernel/debug/tracing/uprobe_events               # active uprobes
```

---

## BCC — BPF Compiler Collection

BCC provides Python and Lua frontends for writing BPF programs without a full C toolchain.

### Install

```bash
apt install bpfcc-tools linux-headers-$(uname -r)  # Debian/Ubuntu
dnf install bcc bcc-tools kernel-devel              # Fedora/RHEL
```

### Hello World (Python/BCC)

```python
from bcc import BPF

prog = r"""
#include <uapi/linux/ptrace.h>
int hello(void *ctx) {
    bpf_trace_printk("Hello, World!\n");
    return 0;
}
"""

b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello")
b.trace_print()
```

### BCC built-in tools

```bash
execsnoop-bpfcc          # trace new process executions
opensnoop-bpfcc          # trace file opens
tcpconnect-bpfcc         # trace TCP connections
tcpaccept-bpfcc          # trace TCP accepts
tcpretrans-bpfcc         # trace TCP retransmissions
biolatency-bpfcc         # block I/O latency histogram
biosnoop-bpfcc           # trace block I/O
cachestat-bpfcc          # page cache hit/miss stats
cachetop-bpfcc           # top-like page cache view
filetop-bpfcc            # top files by I/O
profile-bpfcc            # CPU profiling (flame graph data)
offcputime-bpfcc         # off-CPU time analysis
syscount-bpfcc           # count syscalls
funccount-bpfcc          # count function calls
funclatency-bpfcc        # function latency histograms
trace-bpfcc              # general-purpose tracer
argdist-bpfcc            # argument distribution
stackcount-bpfcc         # count stack traces
```

---

## bpftrace

`bpftrace` is a high-level tracing language for eBPF — like DTrace for Linux.

### Install

```bash
apt install bpftrace            # Debian/Ubuntu
dnf install bpftrace            # Fedora/RHEL
```

### Syntax

```
probe /filter/ { action }
```

### Probe Types

| Probe | Example | Description |
|-------|---------|-------------|
| `kprobe` | `kprobe:vfs_read` | Kernel function entry |
| `kretprobe` | `kretprobe:vfs_read` | Kernel function return |
| `uprobe` | `uprobe:/bin/bash:readline` | User function entry |
| `uretprobe` | `uretprobe:/bin/bash:readline` | User function return |
| `tracepoint` | `tracepoint:syscalls:sys_enter_open` | Static tracepoint |
| `rawtracepoint` | `rawtracepoint:sys_enter` | Raw tracepoint |
| `software` | `software:page-faults:1` | Software event |
| `hardware` | `hardware:cache-misses:1000000` | PMU hardware event |
| `profile` | `profile:hz:99` | Timer-based sampling |
| `interval` | `interval:s:1` | Periodic interval |
| `watchpoint` | `watchpoint:addr:len:mode` | Memory watchpoint |
| `BEGIN` | `BEGIN` | Runs at bpftrace start |
| `END` | `END` | Runs at bpftrace exit |

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `pid` | Process ID |
| `tid` | Thread ID |
| `uid` | User ID |
| `gid` | Group ID |
| `comm` | Process name |
| `nsecs` | Timestamp in nanoseconds |
| `elapsed` | Time since bpftrace start |
| `cpu` | CPU ID |
| `curtask` | Current task_struct pointer |
| `rand` | Random 32-bit integer |
| `args` | Tracepoint arguments struct |
| `retval` | Return value (kretprobe/uretprobe) |
| `func` | Current function name |
| `probe` | Full probe name |
| `arg0`–`argN` | kprobe arguments |
| `$1`–`$N` | bpftrace script positional parameters |

### Built-in Functions

```
printf(fmt, ...)         print formatted output
print(value)             print a value or map
time(fmt)                print formatted time
str(ptr [, len])         read string from pointer
buf(ptr, len)            read bytes as hex
kstack([limit])          kernel stack trace
ustack([limit])          user stack trace
ksym(addr)               resolve kernel symbol
usym(addr)               resolve user symbol
ntop([af,] addr)         format IP address
join(arr [, sep])        join string array
len(map)                 number of elements in map
count()                  count occurrences
sum(n)                   sum of values
avg(n)                   average of values
min(n)                   minimum value
max(n)                   maximum value
hist(n [, base, step])   power-of-2 histogram
lhist(n, min, max, step) linear histogram
delete(@map[key])        delete map element
clear(@map)              clear all map elements
zero(@map)               zero all map values
exit()                   exit bpftrace
system(fmt, ...)         run shell command
cat(file)                print file contents
override(retval)         override return value
signal(sig)              send signal to process
```

### bpftrace One-Liners

```bash
# Trace all new process executions
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args->filename)); }'

# Count syscalls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# File opens by process
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# Kernel function call count
bpftrace -e 'kprobe:vfs_* { @[func] = count(); }'

# CPU profiling (99Hz sampling)
bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'

# Block I/O latency histogram
bpftrace -e 'kprobe:blk_account_io_start { @start[arg0] = nsecs; }
             kprobe:blk_account_io_done
             /@start[arg0]/ { @usecs = hist((nsecs - @start[arg0]) / 1000); delete(@start[arg0]); }'

# TCP connect trace
bpftrace -e 'kprobe:tcp_connect { printf("%s -> %s\n", comm, ntop(((struct sock*)arg0)->__sk_common.skc_daddr)); }'

# Read size distribution by process
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @[comm] = hist(args->ret); }'

# On-CPU time (ms) per process in 10s
bpftrace -e 'profile:hz:99 { @[comm] = count(); } interval:s:10 { print(@); clear(@); exit(); }'

# Trace slow disk I/O (> 10ms)
bpftrace -e 'kprobe:blk_account_io_start { @start[arg0] = nsecs; }
             kprobe:blk_account_io_done
             /@start[arg0] && (nsecs - @start[arg0]) > 10000000/
             { printf("slow I/O: %dms\n", (nsecs - @start[arg0])/1000000); delete(@start[arg0]); }'

# List all available probes matching a pattern
bpftrace -l 'kprobe:vfs_*'
bpftrace -l 'tracepoint:syscalls:*open*'
```

---

## libbpf — C API

`libbpf` is the standard C library for loading and interacting with BPF programs.

### Installation

```bash
apt install libbpf-dev          # Debian/Ubuntu
dnf install libbpf-devel        # Fedora/RHEL
```

### Key API (skeleton-based — recommended)

```c
#include "prog.skel.h"          // generated by: bpftool gen skeleton prog.o

// Open, load, and attach
struct prog_bpf *skel = prog_bpf__open_and_load();
prog_bpf__attach(skel);

// Access maps
bpf_map__fd(skel->maps.my_map);

// Access programs
bpf_program__fd(skel->progs.my_prog);

// Cleanup
prog_bpf__destroy(skel);
```

### Key API (manual)

```c
#include <bpf/libbpf.h>
#include <bpf/bpf.h>

// Open object file
struct bpf_object *obj = bpf_object__open("prog.o");

// Load into kernel
bpf_object__load(obj);

// Find and attach a program
struct bpf_program *prog = bpf_object__find_program_by_name(obj, "my_prog");
struct bpf_link *link = bpf_program__attach(prog);

// Find and use a map
struct bpf_map *map = bpf_object__find_map_by_name(obj, "my_map");
int map_fd = bpf_map__fd(map);
bpf_map_lookup_elem(map_fd, &key, &value);

// Cleanup
bpf_link__destroy(link);
bpf_object__close(obj);
```

### Ring buffer polling

```c
struct ring_buffer *rb = ring_buffer__new(
    bpf_map__fd(skel->maps.events),
    handle_event,    // callback: int handle_event(void *ctx, void *data, size_t sz)
    NULL,
    NULL
);

while (true) {
    ring_buffer__poll(rb, 100 /* ms timeout */);
}

ring_buffer__free(rb);
```

---

## CO-RE — Compile Once, Run Everywhere

CO-RE allows BPF programs compiled on one kernel to run on others with different struct layouts.

### Requirements
- Kernel with BTF enabled (`CONFIG_DEBUG_INFO_BTF=y`)
- Check: `ls /sys/kernel/btf/vmlinux`
- `libbpf` ≥ 0.2
- Clang ≥ 10

### CO-RE Macros

```c
#include "vmlinux.h"             // generated BTF header (all kernel types)
#include <bpf/bpf_core_read.h>

// Safe portable kernel struct read
__u32 pid = BPF_CORE_READ(task, pid);

// Read into variable
BPF_CORE_READ_INTO(&pid, task, pid);

// String read
BPF_CORE_READ_STR_INTO(buf, task, comm);

// Field existence check
if (bpf_core_field_exists(task->thread_info)) { ... }

// Type existence check
if (bpf_core_type_exists(struct new_kernel_struct)) { ... }

// Enum value check
if (bpf_core_enum_value_exists(enum bpf_map_type, BPF_MAP_TYPE_BLOOM_FILTER)) { ... }
```

---

## Kernel Configuration

### Check BPF support

```bash
# Is BPF JIT enabled?
cat /proc/sys/net/core/bpf_jit_enable

# Enable BPF JIT
sysctl -w net.core.bpf_jit_enable=1
echo 'net.core.bpf_jit_enable=1' >> /etc/sysctl.conf

# Enable JIT hardening (security)
sysctl -w net.core.bpf_jit_harden=2

# BPF unprivileged access (0=allowed, 1=admin only, 2=disabled)
cat /proc/sys/kernel/unprivileged_bpf_disabled
sysctl -w kernel.unprivileged_bpf_disabled=1

# Check kernel config
grep BPF /boot/config-$(uname -r)
zgrep BPF /proc/config.gz

# Required kernel config options
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_DEBUG_INFO_BTF=y         # for CO-RE
CONFIG_BPF_LSM=y                # for LSM programs
CONFIG_BPF_EVENTS=y             # for tracing
CONFIG_HAVE_EBPF_JIT=y
CONFIG_CGROUP_BPF=y             # for cgroup programs
```

### Check available features

```bash
bpftool feature probe kernel
bpftool feature probe | grep -i "Program type\|Map type\|Helper"
```

---

## Security & Permissions

| Capability | Required for |
|------------|-------------|
| `CAP_BPF` | Load most BPF programs (kernel 5.8+) |
| `CAP_NET_ADMIN` | Network BPF (XDP, TC, socket) |
| `CAP_PERFMON` | Tracing/perf BPF programs |
| `CAP_SYS_ADMIN` | All BPF (legacy, before 5.8) |

```bash
# Check current process capabilities
capsh --print

# Grant capabilities to a binary
setcap cap_bpf,cap_perfmon+eip /path/to/tool

# Run with specific capabilities
capsh --caps="cap_bpf+eip cap_perfmon+eip" -- -c "bpftrace ..."
```

### Verifier limits (kernel defaults)

| Limit | Default |
|-------|---------|
| Max instructions | 1,000,000 |
| Max stack depth | 512 bytes |
| Max tail call depth | 33 |
| Max map entries | per map type |
| Max BPF-to-BPF call depth | 8 |

---

## Observability Tools Built on eBPF

| Tool | Project | Description |
|------|---------|-------------|
| `bpftrace` | bpftrace | High-level DTrace-like tracing language |
| `bcc` tools | BCC | Python-based toolkit (100+ tools) |
| `kubectl trace` | kubectl-trace | Run bpftrace on Kubernetes nodes |
| Cilium | Cilium | Kubernetes CNI + network policy |
| Falco | Falco | Runtime security using eBPF |
| Pixie | px.dev | Kubernetes observability |
| Parca | Parca | Continuous profiling with eBPF |
| Pyroscope | Pyroscope | Continuous profiling |
| Tetragon | Cilium/Tetragon | eBPF-based security observability |
| Katran | Facebook | L4 load balancer using XDP |
| Tracee | Aqua Security | Runtime security and forensics |

---

## Useful /proc and /sys Paths

```bash
/proc/sys/net/core/bpf_jit_enable       # JIT on/off
/proc/sys/net/core/bpf_jit_harden       # JIT hardening level
/proc/sys/kernel/unprivileged_bpf_disabled
/sys/kernel/btf/vmlinux                 # kernel BTF blob
/sys/kernel/debug/tracing/trace_pipe    # bpf_printk output
/sys/kernel/debug/tracing/trace         # snapshot trace buffer
/sys/kernel/debug/tracing/events/       # available tracepoints
/sys/kernel/debug/tracing/available_filter_functions  # kprobeable fns
/sys/kernel/debug/tracing/kprobe_events
/sys/kernel/debug/tracing/uprobe_events
/sys/fs/bpf/                            # BPF filesystem (pinned objects)
/proc/kallsyms                          # kernel symbol table
```

---

## Quick Debugging Checklist

```bash
# 1. Check kernel BPF support
bpftool feature probe kernel | head -20

# 2. Check BTF availability (needed for CO-RE and many tools)
ls -lh /sys/kernel/btf/vmlinux

# 3. List loaded programs
bpftool prog list

# 4. List loaded maps
bpftool map list

# 5. Verify JIT is on
cat /proc/sys/net/core/bpf_jit_enable

# 6. Watch bpf_printk output
cat /sys/kernel/debug/tracing/trace_pipe

# 7. Check verifier log (on load failure — enable in libbpf)
bpf_object__open_opts with .kernel_log_level = 2

# 8. Inspect pinned BPF filesystem objects
ls /sys/fs/bpf/

# 9. Validate an object file
bpftool prog load prog.o /sys/fs/bpf/test_prog && echo "OK"

# 10. Check kernel log for BPF errors
dmesg | grep -i bpf
```

---

## Minimum Kernel Version Reference

| Feature | Min kernel |
|---------|-----------|
| eBPF (basic) | 3.18 |
| Maps | 3.19 |
| JIT (x86_64) | 3.18 |
| Tail calls | 4.2 |
| kprobes | 4.1 |
| Tracepoints | 4.7 |
| XDP | 4.8 |
| cgroups BPF | 4.10 |
| Socket ops | 4.13 |
| BTF | 4.18 |
| Raw tracepoints | 4.17 |
| Ring buffer | 5.8 |
| `CAP_BPF` split | 5.8 |
| CO-RE (stable) | 5.5 |
| fentry/fexit | 5.5 |
| LSM BPF | 5.7 |
| BPF iterators | 5.9 |
| Timer helpers | 5.15 |
| kfuncs (stable) | 5.18 |

---

## See Also

| Resource | URL |
|----------|-----|
| Kernel BPF docs | https://docs.kernel.org/bpf/ |
| libbpf docs | https://libbpf.readthedocs.io |
| bpftrace reference | https://github.com/bpftrace/bpftrace/blob/master/docs/reference_guide.md |
| BCC reference | https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md |
| eBPF.io | https://ebpf.io |
| Cilium BPF guide | https://docs.cilium.io/en/stable/bpf/ |
| BPF Performance Tools (book) | Brendan Gregg, 2019 |

---

*eBPF evolves rapidly — features, helpers, and program types are added with nearly every kernel release. Always check `bpftool feature probe` against your specific kernel version.*
