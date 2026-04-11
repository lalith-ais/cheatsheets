# Linux Namespaces Cheatsheet

## Table of Contents
1. [Overview](#overview)
2. [Namespace Types](#namespace-types)
3. [Core Syscalls](#core-syscalls)
4. [Namespace Files in /proc](#namespace-files-in-proc)
5. [unshare — Create Namespaces](#unshare--create-namespaces)
6. [nsenter — Enter Namespaces](#nsenter--enter-namespaces)
7. [ip netns — Network Namespaces](#ip-netns--network-namespaces)
8. [lsns — List Namespaces](#lsns--list-namespaces)
9. [Mount Namespace](#mount-namespace)
10. [UTS Namespace](#uts-namespace)
11. [IPC Namespace](#ipc-namespace)
12. [PID Namespace](#pid-namespace)
13. [Network Namespace](#network-namespace)
14. [User Namespace](#user-namespace)
15. [Cgroup Namespace](#cgroup-namespace)
16. [Time Namespace](#time-namespace)
17. [UID/GID Mapping](#uidgid-mapping)
18. [Combining Namespaces](#combining-namespaces)
19. [Namespaces in C](#namespaces-in-c)
20. [Inspecting Running Containers](#inspecting-running-containers)
21. [Security Considerations](#security-considerations)
22. [Quick Reference Card](#quick-reference-card)

---

## Overview

Linux namespaces wrap a global system resource in an abstraction layer, giving processes the illusion they have their own isolated instance of that resource. They are one of the two core kernel primitives underpinning containers (the other being cgroups).

```
Process A ──┐
            ├──► Namespace (isolated view of resource)
Process B ──┘

Host ──────────► Global resource (kernel-managed)
```

### Key facts

- A namespace is created with `clone(2)`, `unshare(2)`, or `setns(2)`
- Every process belongs to exactly one instance of each namespace type
- Namespaces are reference-counted — destroyed when the last member exits (unless bind-mounted or held open via `/proc/<pid>/ns/`)
- Namespaces can be nested (PID, User)
- Non-root processes can create User namespaces; all other types require `CAP_SYS_ADMIN`

---

## Namespace Types

| Flag | Constant | Kernel | Isolates |
|------|----------|--------|----------|
| `CLONE_NEWNS` | `mnt` | 2.4.19 | Mount points, filesystem tree |
| `CLONE_NEWUTS` | `uts` | 2.6.19 | Hostname and NIS domain name |
| `CLONE_NEWIPC` | `ipc` | 2.6.19 | System V IPC, POSIX message queues |
| `CLONE_NEWPID` | `pid` | 2.6.24 | Process IDs |
| `CLONE_NEWNET` | `net` | 2.6.24 | Network interfaces, routing, iptables |
| `CLONE_NEWUSER` | `user` | 3.8 | UIDs, GIDs, capabilities |
| `CLONE_NEWCGROUP` | `cgroup` | 4.6 | cgroup root directory |
| `CLONE_NEWTIME` | `time` | 5.6 | Boot and monotonic clocks |

---

## Core Syscalls

| Syscall | Purpose |
|---------|---------|
| `clone(2)` | Create a new process in new namespace(s) |
| `unshare(2)` | Disassociate parts of the calling process's execution context |
| `setns(2)` | Reassociate a thread with an existing namespace (via fd) |
| `ioctl(2)` `NS_GET_USERNS` | Get the owning user namespace of a namespace |
| `ioctl(2)` `NS_GET_PARENT` | Get the parent namespace (PID/user) |

```c
// clone flags combine namespace + fork behaviour
clone(child_fn, stack, CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, arg);

// unshare from within a running process
unshare(CLONE_NEWNS | CLONE_NEWUTS);

// setns — join an existing namespace via an open fd
int fd = open("/proc/1234/ns/net", O_RDONLY);
setns(fd, CLONE_NEWNET);
close(fd);
```

---

## Namespace Files in /proc

Every process has a symlink for each namespace type under `/proc/<pid>/ns/`:

```bash
ls -la /proc/$$/ns/
# lrwxrwxrwx cgroup -> cgroup:[4026531835]
# lrwxrwxrwx ipc    -> ipc:[4026531839]
# lrwxrwxrwx mnt    -> mnt:[4026531840]
# lrwxrwxrwx net    -> net:[4026531992]
# lrwxrwxrwx pid    -> pid:[4026531836]
# lrwxrwxrwx pid_for_children -> pid:[4026531836]
# lrwxrwxrwx time   -> time:[4026531834]
# lrwxrwxrwx time_for_children -> time:[4026531834]
# lrwxrwxrwx user   -> user:[4026531837]
# lrwxrwxrwx uts    -> uts:[4026531838]

# The inode number in brackets is the namespace identifier
# Two processes share a namespace if their symlinks have the same inode

# Compare namespaces between two processes
stat -L /proc/1234/ns/net /proc/5678/ns/net

# Keep a namespace alive after all processes exit (bind mount)
touch /run/netns/mynet
mount --bind /proc/$$/ns/net /run/netns/mynet

# Open a namespace fd for setns(2)
exec 7</proc/1234/ns/net      # Open on fd 7
nsenter --net=/proc/self/fd/7 -- ip a
exec 7>&-                     # Close fd
```

---

## unshare — Create Namespaces

`unshare` runs a program in new namespace(s), optionally creating them from scratch.

### Syntax

```
unshare [OPTIONS] [PROGRAM [ARGS]]
```

### Flags

| Flag | Namespace | Notes |
|------|-----------|-------|
| `-m`, `--mount` | Mount | Also accepts `=<file>` to join existing |
| `-u`, `--uts` | UTS | |
| `-i`, `--ipc` | IPC | |
| `-p`, `--pid` | PID | Use with `--fork` and `--mount-proc` |
| `-n`, `--net` | Network | |
| `-U`, `--user` | User | No root required |
| `-C`, `--cgroup` | Cgroup | |
| `-T`, `--time` | Time | |
| `-f`, `--fork` | — | Fork before exec (required for `--pid`) |
| `--mount-proc[=PATH]` | — | Mount fresh `/proc` (required for `--pid`) |
| `-r`, `--map-root-user` | — | Map current UID/GID to 0 in user ns |
| `--map-user=UID` | — | Map calling UID to UID in user ns |
| `--map-group=GID` | — | Map calling GID to GID in user ns |
| `--map-auto` | — | Automatically map UIDs/GIDs from `/etc/sub{u,g}id` |
| `--keep-caps` | — | Retain capabilities after user ns creation |
| `--kill-child[=SIGNAME]` | — | Send signal to child when unshare exits |
| `--propagation slave|shared|private|unchanged` | — | Mount propagation mode |
| `-S`, `--setuid UID` | — | Set UID inside the namespace |
| `-G`, `--setgid GID` | — | Set GID inside the namespace |

### Examples

```bash
# New UTS namespace — change hostname without affecting host
unshare --uts bash
hostname container-01
hostname         # container-01
exit
hostname         # unchanged on host

# New mount namespace — mount without affecting host
unshare --mount bash
mount -t tmpfs tmpfs /mnt
findmnt /mnt     # visible here
exit
findmnt /mnt     # gone on host

# New PID namespace with fresh /proc
unshare --pid --fork --mount-proc bash
ps aux           # only sees processes in this namespace
echo $$          # prints 1 (init of the new namespace)

# New network namespace
unshare --net bash
ip link          # only lo, no host interfaces

# Full unprivileged container-like environment (no root needed)
unshare --user --map-root-user --mount --pid --fork --mount-proc --uts --ipc --net bash

# New user namespace mapping your UID as root
unshare -Ur bash
id               # uid=0(root) inside the namespace
cat /proc/$$/uid_map   # 0 <your-uid> 1

# New cgroup namespace
unshare --cgroup bash
cat /proc/$$/cgroup    # sees / as root of cgroup tree

# New time namespace (adjust clocks per-container)
unshare --time bash
```

---

## nsenter — Enter Namespaces

`nsenter` joins one or more namespaces of an existing process.

### Syntax

```
nsenter [OPTIONS] [PROGRAM [ARGS]]
```

### Flags

| Flag | Namespace | Notes |
|------|-----------|-------|
| `-t PID`, `--target PID` | — | PID whose namespaces to enter (required) |
| `-m[=FILE]`, `--mount` | Mount | |
| `-u[=FILE]`, `--uts` | UTS | |
| `-i[=FILE]`, `--ipc` | IPC | |
| `-p[=FILE]`, `--pid` | PID | |
| `-n[=FILE]`, `--net` | Network | |
| `-U[=FILE]`, `--user` | User | |
| `-C[=FILE]`, `--cgroup` | Cgroup | |
| `-T[=FILE]`, `--time` | Time | |
| `-a`, `--all` | All | Enter all namespaces of target |
| `-r[=DIR]`, `--root` | — | Set root directory |
| `-w[=DIR]`, `--wd` | — | Set working directory |
| `--preserve-credentials` | — | Don't drop caps/groups when entering user ns |
| `-S UID`, `--setuid` | — | Set UID after entering |
| `-G GID`, `--setgid` | — | Set GID after entering |

### Examples

```bash
# Enter all namespaces of PID 1234
nsenter --target 1234 --all bash

# Enter only the network namespace of a process
nsenter --target 1234 --net ip addr

# Enter network namespace of a container (Docker example)
PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)
nsenter --target $PID --net ip addr

# Enter mount + pid namespace
nsenter --target 1234 --mount --pid bash

# Enter a network namespace kept alive by bind mount
nsenter --net=/run/netns/mynet ip addr

# Enter a namespace via its /proc/pid/ns/ path directly
nsenter --net=/proc/1234/ns/net -- ip route

# Run a command in the same namespaces as another shell
nsenter -t $$ --all bash
```

---

## ip netns — Network Namespaces

`iproute2` has first-class support for named network namespaces stored in `/run/netns/`.

```bash
# Create a named network namespace
ip netns add mynet

# List named network namespaces
ip netns list
ip netns show

# Delete a named network namespace
ip netns delete mynet

# Execute a command inside a network namespace
ip netns exec mynet bash
ip netns exec mynet ip link
ip netns exec mynet ip addr

# Identify which netns a process is in
ip netns identify $$
ip netns identify <pid>

# Monitor namespace events
ip netns monitor

# Assign an interface to a namespace
ip link set eth1 netns mynet

# Move a veth pair endpoint to a namespace
ip link add veth0 type veth peer name veth1
ip link set veth1 netns mynet

# Configure the interface from within the namespace
ip netns exec mynet ip link set veth1 up
ip netns exec mynet ip addr add 10.0.0.2/24 dev veth1

# Add a loopback inside the namespace
ip netns exec mynet ip link set lo up

# Connect two namespaces via a veth pair
ip netns add ns1
ip netns add ns2
ip link add veth-ns1 type veth peer name veth-ns2
ip link set veth-ns1 netns ns1
ip link set veth-ns2 netns ns2
ip netns exec ns1 ip addr add 192.168.1.1/24 dev veth-ns1
ip netns exec ns2 ip addr add 192.168.1.2/24 dev veth-ns2
ip netns exec ns1 ip link set veth-ns1 up
ip netns exec ns2 ip link set veth-ns2 up
ip netns exec ns1 ping -c1 192.168.1.2
```

---

## lsns — List Namespaces

```bash
# List all namespaces on the system
lsns

# List namespaces of a specific type
lsns --type net
lsns --type pid
lsns --type mnt
lsns --type user
lsns --type uts
lsns --type ipc
lsns --type cgroup
lsns --type time

# List namespaces for a specific PID
lsns --task <pid>

# Output as JSON
lsns --json

# Output specific columns
lsns -o NS,TYPE,NPROCS,PID,PPID,COMMAND

# All available columns
lsns --output-all
```

### lsns column reference

| Column | Description |
|--------|-------------|
| `NS` | Namespace inode (unique identifier) |
| `TYPE` | Namespace type |
| `NPROCS` | Number of processes in namespace |
| `PID` | Lead process PID |
| `PPID` | Parent PID |
| `USER` | Username of the lead process |
| `UIDMAP` | UID mapping (user ns only) |
| `GIDMAP` | GID mapping (user ns only) |
| `COMMAND` | Command of lead process |
| `PATH` | Path of the namespace file |

---

## Mount Namespace

Isolates the filesystem mount table. Each namespace has an independent view of the mounted filesystem tree.

```bash
# Create a new mount namespace
unshare --mount bash

# Mount propagation modes
# private   — mounts/unmounts do NOT propagate in or out
# shared    — mounts/unmounts propagate bidirectionally
# slave     — propagates FROM parent but NOT back to parent
# unbindable — private and cannot be bind-mounted

# Set propagation on all mounts
mount --make-rprivate /      # Make everything private
mount --make-rshared /       # Make everything shared
mount --make-rslave /

# Make a specific mount private
mount --make-private /mnt

# Bind mount (expose a path at another location)
mount --bind /source /destination
mount --rbind /source /destination   # Recursive (include submounts)

# Read-only bind mount
mount --bind /source /destination
mount -o remount,ro,bind /destination

# Overlay filesystem (used in containers for layers)
mount -t overlay overlay \
  -o lowerdir=/lower,upperdir=/upper,workdir=/work \
  /merged

# Useful: see all mounts in current namespace
findmnt
findmnt --tree
cat /proc/$$/mounts
cat /proc/mounts

# See mount propagation
findmnt -o TARGET,PROPAGATION
cat /proc/$$/mountinfo        # Full mount info including propagation
```

---

## UTS Namespace

Isolates the hostname and NIS domain name. UTS = UNIX Time-sharing System.

```bash
# Create a new UTS namespace
unshare --uts bash

# Get / set hostname
hostname
hostname new-hostname
uname -n

# Get / set NIS domain name
domainname
domainname new-domain

# Programmatically
sethostname("mycontainer", 11);    # C syscall
setdomainname("(none)", 6);        # C syscall

# Verify isolation
unshare --uts sh -c 'hostname test-ns; hostname'
hostname   # unchanged on host
```

---

## IPC Namespace

Isolates System V IPC objects (message queues, semaphores, shared memory) and POSIX message queues.

```bash
# Create a new IPC namespace
unshare --ipc bash

# System V IPC tools
ipcs                    # List all IPC objects
ipcs -m                 # Shared memory segments
ipcs -q                 # Message queues
ipcs -s                 # Semaphore arrays

# Create IPC objects (only visible inside this namespace)
ipcmk -M 4096           # Create shared memory segment
ipcmk -Q                # Create message queue
ipcmk -S 1              # Create semaphore

# Remove IPC objects
ipcrm -M <shmid>        # Remove shared memory
ipcrm -Q <msqid>        # Remove message queue
ipcrm -S <semid>        # Remove semaphore
ipcrm --all             # Remove everything (dangerous!)

# POSIX message queues (appear under /dev/mqueue)
ls /dev/mqueue
# Each namespace has its own /dev/mqueue

# Verify isolation
unshare --ipc bash
ipcmk -Q                # Create queue (ipc namespace A)
ipcs -q                 # Visible here
# In another terminal (host): ipcs -q → not visible
```

---

## PID Namespace

Isolates the process ID number space. Processes in a child PID namespace have a different PID inside the namespace versus their PID as seen from the parent namespace.

```bash
# Create a new PID namespace
# --fork is required: the first child becomes PID 1 (init)
# --mount-proc: mount a fresh /proc (otherwise ps/top show host PIDs)
unshare --pid --fork --mount-proc bash

echo $$              # 1 — we are "init" in this namespace
ps aux               # only processes in this namespace

# If PID 1 exits, all other processes in the namespace are killed

# Nested PID namespaces
# From host: process has PID 5000
# In container namespace: same process has PID 42
# In nested namespace: same process has PID 7
# Each layer has its own /proc

# View the PID translation
cat /proc/<host-pid>/status | grep NSpid
# NSpid: 5000  42  7   ← host, container, nested

# /proc/PID/ns/pid         → namespace the process is currently in
# /proc/PID/ns/pid_for_children → namespace new children will be placed in
# (These differ after unshare --pid before forking)

# Kill a process by its host PID (always works from host)
kill -9 <host-pid>

# Run a minimal init in the PID namespace
unshare --pid --fork --mount-proc /sbin/init

# Check PID namespace hierarchy
readlink /proc/$$/ns/pid
readlink /proc/1/ns/pid
```

---

## Network Namespace

Isolates network stack: interfaces, IP addresses, routing tables, iptables rules, sockets, and `/proc/net`.

```bash
# Create a new, empty network namespace
unshare --net bash
ip link              # only loopback lo (DOWN by default)
ip addr              # no addresses

# Bring up loopback
ip link set lo up
ip addr              # lo is up

# Full isolated network setup example
# 1. Create two namespaces
ip netns add ns-left
ip netns add ns-right

# 2. Create a veth pair (virtual ethernet cable)
ip link add veth-l type veth peer name veth-r

# 3. Assign each end to a namespace
ip link set veth-l netns ns-left
ip link set veth-r netns ns-right

# 4. Configure IP addresses
ip netns exec ns-left  ip addr add 10.1.0.1/24 dev veth-l
ip netns exec ns-right ip addr add 10.1.0.2/24 dev veth-r

# 5. Bring interfaces up
ip netns exec ns-left  ip link set veth-l up
ip netns exec ns-right ip link set veth-r up
ip netns exec ns-left  ip link set lo up
ip netns exec ns-right ip link set lo up

# 6. Test connectivity
ip netns exec ns-left ping -c3 10.1.0.2

# Connect a namespace to the host via a bridge
ip link add br0 type bridge
ip link set br0 up
ip addr add 10.2.0.1/24 dev br0

ip netns add client
ip link add veth-host type veth peer name veth-client
ip link set veth-client netns client
ip link set veth-host master br0
ip link set veth-host up
ip netns exec client ip addr add 10.2.0.100/24 dev veth-client
ip netns exec client ip link set veth-client up
ip netns exec client ip link set lo up
ip netns exec client ip route add default via 10.2.0.1
ip netns exec client ping -c3 10.2.0.1

# Enable NAT for outbound internet access from namespace
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 10.2.0.0/24 -j MASQUERADE

# Inspect network namespace from host
ip netns exec client ip route show
ip netns exec client ss -tulpn
ip netns exec client iptables -L
```

---

## User Namespace

Isolates user and group IDs and the capability set. Allows unprivileged processes to appear as root inside a namespace. Introduced in kernel 3.8.

```bash
# Create user namespace as non-root (no sudo required)
unshare --user bash
id                   # uid=65534(nobody) — unmapped initially

# Map calling user to root inside the namespace
unshare --user --map-root-user bash
id                   # uid=0(root) gid=0(root) inside namespace
# But on the host, still running as your real UID

# Map full range from /etc/subuid and /etc/subgid
unshare --user --map-auto bash

# Manually inspect UID/GID maps
cat /proc/$$/uid_map    # inside-uid  outside-uid  count
cat /proc/$$/gid_map

# Write UID map manually (from parent namespace only, once)
# Format: <inside-start> <outside-start> <count>
echo "0 1000 1" > /proc/<child-pid>/uid_map
echo "deny"     > /proc/<child-pid>/setgroups   # required before gid_map
echo "0 1000 1" > /proc/<child-pid>/gid_map

# User namespaces and capabilities
# Inside a user namespace, a process can have a full set of capabilities
# but they only apply to resources governed by that namespace
# CAP_SYS_ADMIN inside user ns ≠ CAP_SYS_ADMIN on host

# Nest user namespaces (limit: controlled by /proc/sys/user/max_user_namespaces)
cat /proc/sys/user/max_user_namespaces

# User namespace ownership
# Every other namespace type is owned by a user namespace
# The owning user namespace determines which UIDs have caps over it
ioctl(fd, NS_GET_USERNS)    # get owning user namespace (C)
```

---

## Cgroup Namespace

Isolates the cgroup root directory. Processes in a cgroup namespace see their cgroup root as `/`, hiding ancestor cgroup hierarchy from the container.

```bash
# Create a new cgroup namespace
unshare --cgroup bash

# View cgroup membership
cat /proc/$$/cgroup

# Inside a cgroup namespace, cgroup paths appear relative to the
# namespace root — ancestors are hidden

# cgroupv2 hierarchy
ls /sys/fs/cgroup/

# View cgroup of a specific process
cat /proc/<pid>/cgroup

# Without cgroup namespace:
# /sys/fs/cgroup/memory/docker/abc123.../
# With cgroup namespace (inside container):
# /sys/fs/cgroup/memory/   (looks like root)

# Useful: show cgroup of current shell
cat /proc/self/cgroup
```

---

## Time Namespace

Isolates the `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME` clocks. Allows containers to have a different perception of uptime and monotonic time. Introduced in kernel 5.6.

```bash
# Create a new time namespace
unshare --time bash

# Adjust clock offsets (written before exec, via /proc/self/timens_offsets)
# Format: <clock-id> <seconds> <nanoseconds>
# CLOCK_MONOTONIC = 1, CLOCK_BOOTTIME = 7

# Set monotonic clock 100 seconds behind host
echo "monotonic 100 0" > /proc/self/timens_offsets
unshare --time bash
# Now CLOCK_MONOTONIC reads ~100s behind host

# Read current time namespace offsets
cat /proc/$$/timens_offsets

# Check clock values
clock_gettime CLOCK_MONOTONIC   # via custom C program
date                            # wall-clock (CLOCK_REALTIME) is NOT affected

# Use case: checkpointing — restore a process with the same
# apparent uptime/monotonic time as when it was checkpointed
```

---

## UID/GID Mapping

User namespace UID/GID maps define how UIDs/GIDs inside the namespace correspond to UIDs/GIDs outside.

```
Inside namespace ←──── uid_map ────→ Outside (host) namespace
uid 0               maps to         uid 100000
uid 1               maps to         uid 100001
...                 ...             ...
uid 65535           maps to         uid 165535
```

```bash
# View maps for a process
cat /proc/$$/uid_map     # <ns-uid-start> <host-uid-start> <count>
cat /proc/$$/gid_map

# /etc/subuid and /etc/subgid — subordinate ID ranges
# Username:start-uid:count
cat /etc/subuid          # alice:100000:65536
cat /etc/subgid          # alice:100000:65536

# Add subuid/subgid ranges
usermod --add-subuids 100000-165535 alice
usermod --add-subgids 100000-165535 alice

# newuidmap / newgidmap — write maps with multiple ranges
# (setuid helpers that allow non-root users to write uid_map)
newuidmap <pid> <ns-uid> <host-uid> <count> [...]
newgidmap <pid> <ns-gid> <host-gid> <count> [...]

# Example: map UID 0 (container root) → UID 1000 (your user)
#          map UID 1-65535 → subordinate range 100000-165534
newuidmap $childpid   0 1000 1   1 100000 65535
newgidmap $childpid   0 1000 1   1 100000 65535

# Inside a user namespace with --map-root-user
unshare --user --map-root-user bash
cat /proc/$$/uid_map    # 0  <your-uid>  1

# Disable setgroups (required before writing gid_map from parent)
echo deny > /proc/<childpid>/setgroups
```

---

## Combining Namespaces

Real container runtimes combine multiple namespace types. Here is the progression from minimal to full isolation:

```bash
# Minimal: just a new shell with a new hostname
unshare --uts bash
hostname container

# Add process isolation
unshare --uts --pid --fork --mount-proc bash

# Add network isolation
unshare --uts --pid --fork --mount-proc --net bash

# Add IPC isolation
unshare --uts --pid --fork --mount-proc --net --ipc bash

# Full unprivileged container (no root required)
unshare \
  --user --map-root-user \
  --mount \
  --uts \
  --ipc \
  --pid --fork --mount-proc \
  --net \
  bash

# Full container with a root filesystem (using chroot)
unshare \
  --user --map-root-user \
  --mount \
  --uts \
  --ipc \
  --pid --fork \
  --net \
  bash -c "
    mount -t proc proc /rootfs/proc
    mount --bind /dev /rootfs/dev
    mount --bind /sys /rootfs/sys
    chroot /rootfs /bin/bash
  "

# What Docker / containerd do under the hood:
# clone(CLONE_NEWUSER | CLONE_NEWNS | CLONE_NEWUTS |
#       CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNET | SIGCHLD)
```

### Namespace isolation matrix

| Resource | mnt | uts | ipc | pid | net | user | cgroup | time |
|----------|:---:|:---:|:---:|:---:|:---:|:----:|:------:|:----:|
| Filesystem mounts | ✓ | | | | | | | |
| Hostname / domainname | | ✓ | | | | | | |
| SysV / POSIX IPC | | | ✓ | | | | | |
| Process IDs | | | | ✓ | | | | |
| Network stack | | | | | ✓ | | | |
| UID / GID / capabilities | | | | | | ✓ | | |
| cgroup root view | | | | | | | ✓ | |
| Monotonic / boot clocks | | | | | | | | ✓ |

---

## Namespaces in C

```c
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <stdlib.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];

static int child_fn(void *arg) {
    // Inside new namespaces
    sethostname("container", 9);
    
    // Mount a fresh /proc
    mount("proc", "/proc", "proc", 0, NULL);
    
    execl("/bin/bash", "bash", NULL);
    return 1;
}

int main(void) {
    pid_t pid = clone(
        child_fn,
        child_stack + STACK_SIZE,   // stack grows down
        CLONE_NEWUTS |
        CLONE_NEWPID |
        CLONE_NEWNS  |
        CLONE_NEWNET |
        CLONE_NEWIPC |
        SIGCHLD,
        NULL
    );
    
    if (pid < 0) { perror("clone"); exit(1); }
    
    waitpid(pid, NULL, 0);
    return 0;
}
```

```c
// unshare(2) — drop into new namespace from running process
#include <sched.h>

if (unshare(CLONE_NEWNS | CLONE_NEWUTS) < 0) {
    perror("unshare");
    exit(1);
}

// setns(2) — join an existing namespace
#include <fcntl.h>
#include <sched.h>

int fd = open("/proc/1234/ns/net", O_RDONLY | O_CLOEXEC);
if (setns(fd, CLONE_NEWNET) < 0) {
    perror("setns");
    exit(1);
}
close(fd);

// Write UID map (from parent process after clone)
void write_uid_map(pid_t child_pid) {
    char path[64];
    snprintf(path, sizeof(path), "/proc/%d/uid_map", child_pid);
    FILE *f = fopen(path, "w");
    fprintf(f, "0 %d 1\n", getuid());   // map ns uid 0 → our uid
    fclose(f);
}
```

---

## Inspecting Running Containers

```bash
# Get the PID of a running Docker container
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)

# List all namespaces of the container process
lsns --task $PID
ls -la /proc/$PID/ns/

# Compare container ns to host ns
readlink /proc/$PID/ns/net
readlink /proc/1/ns/net            # host net namespace

# Enter the container's network namespace
nsenter --target $PID --net bash
ip addr
ip route

# Enter all namespaces (become the container)
nsenter --target $PID --all bash

# Enter just mount namespace (see container filesystem)
nsenter --target $PID --mount bash
ls /

# Enter network namespace of a Docker container by name
nsenter --target $(docker inspect -f '{{.State.Pid}}' mycontainer) \
  --net -- ip addr

# Run tcpdump inside a container's network namespace
nsenter --target $PID --net -- tcpdump -i eth0

# Find all processes in the same namespace as a PID
lsns -o NS,TYPE,NPROCS,PID,COMMAND | grep $(readlink /proc/$PID/ns/pid | grep -o '[0-9]*')

# Check if two processes share a network namespace
[[ $(readlink /proc/PID1/ns/net) == $(readlink /proc/PID2/ns/net) ]] \
  && echo "same" || echo "different"

# podman / containerd / runc equivalents
runc state mycontainer          # get container init PID
crictl inspect --output go-template \
  --template '{{.info.pid}}' <container-id>
```

---

## Security Considerations

### Privilege implications

```
Namespace type    | Requires root? | Notes
─────────────────────────────────────────────────────────────────
user              | No             | Gateway to other ns types unprivileged
mnt               | Yes*           | *No if inside user ns
uts               | Yes*           | *No if inside user ns
ipc               | Yes*           | *No if inside user ns
pid               | Yes*           | *No if inside user ns
net               | Yes*           | *No if inside user ns
cgroup            | Yes*           | *No if inside user ns
time              | Yes*           | *No if inside user ns
```

### Attack surface & hardening

```bash
# Limit user namespace creation (mitigates many kernel exploits)
# (breaks rootless containers — choose carefully)
sysctl -w kernel.unprivileged_userns_clone=0   # Debian/Ubuntu
echo 0 > /proc/sys/user/max_user_namespaces    # Limit to 0

# Current limit on user namespaces
cat /proc/sys/user/max_user_namespaces

# Current limit on other namespace types
cat /proc/sys/user/max_mnt_namespaces
cat /proc/sys/user/max_pid_namespaces
cat /proc/sys/user/max_net_namespaces
cat /proc/sys/user/max_ipc_namespaces
cat /proc/sys/user/max_uts_namespaces
cat /proc/sys/user/max_cgroup_namespaces

# Seccomp: restrict clone() flags in untrusted containers
# Docker default seccomp profile blocks CLONE_NEWUSER by default

# AppArmor / SELinux profiles restrict what namespaced processes can do

# Capabilities inside user namespaces
# CAP_SYS_ADMIN inside user ns allows:
#   - mounting filesystems (in that ns)
#   - creating further namespaces
# But NOT: loading kernel modules, raw device access, etc.

# Check if running inside a container/namespace
cat /proc/1/cgroup             # Non-root cgroup path = inside container
[[ -f /.dockerenv ]] && echo "docker"
systemd-detect-virt --container
```

### Key CVEs related to namespaces

| CVE | Namespace | Summary |
|-----|-----------|---------|
| CVE-2016-8655 | net | Race condition in packet socket |
| CVE-2020-14386 | net | Privilege escalation via AF_PACKET |
| CVE-2022-0185 | mnt | Heap overflow in legacy_parse_param |
| CVE-2022-25636 | net | Heap OOB in nf_dup_netdev |
| CVE-2023-32233 | net | Use-after-free in Netfilter |

---

## Quick Reference Card

```
NAMESPACE TYPES
  mnt     Mount points                  CLONE_NEWNS
  uts     Hostname / domainname         CLONE_NEWUTS
  ipc     SysV IPC / POSIX mqueues      CLONE_NEWIPC
  pid     Process IDs                   CLONE_NEWPID
  net     Network stack                 CLONE_NEWNET
  user    UIDs, GIDs, capabilities      CLONE_NEWUSER
  cgroup  cgroup root view              CLONE_NEWCGROUP
  time    Monotonic / boot clocks       CLONE_NEWTIME

SYSCALLS
  clone(fn, stack, flags, arg)          Fork into new namespace(s)
  unshare(flags)                        Create namespace in-place
  setns(fd, nstype)                     Join existing namespace via fd

UNSHARE QUICK EXAMPLES
  unshare --uts bash                    New hostname isolation
  unshare --pid --fork --mount-proc sh  New PID isolation + /proc
  unshare --net bash                    New network stack
  unshare --user --map-root-user bash   Unprivileged root in user ns
  unshare -Urn bash                     user + net, mapped as root
  unshare --mount bash                  New mount table

NSENTER QUICK EXAMPLES
  nsenter -t PID --all bash             Enter all namespaces of PID
  nsenter -t PID --net ip addr          Enter only network namespace
  nsenter -t PID --mount --pid bash     Enter mount + PID namespace
  nsenter --net=/run/netns/mynet bash   Enter named network namespace

IP NETNS QUICK EXAMPLES
  ip netns add mynet                    Create named network namespace
  ip netns list                         List named network namespaces
  ip netns exec mynet ip addr           Run command in namespace
  ip netns delete mynet                 Delete namespace
  ip link set eth1 netns mynet          Move interface to namespace

LSNS QUICK EXAMPLES
  lsns                                  List all namespaces
  lsns --type net                       List only network namespaces
  lsns --task <pid>                     Namespaces of a process
  lsns -o NS,TYPE,NPROCS,PID,COMMAND    Custom columns

PROC FILES
  /proc/<pid>/ns/<type>                 Namespace fd / inode
  /proc/<pid>/uid_map                   UID mapping (user ns)
  /proc/<pid>/gid_map                   GID mapping (user ns)
  /proc/<pid>/setgroups                 Control setgroups (user ns)
  /proc/self/timens_offsets             Time offsets (time ns)
  /proc/<pid>/status | grep NSpid       PID in each nested ns

KEEP A NAMESPACE ALIVE
  touch /run/netns/mynet
  mount --bind /proc/$$/ns/net /run/netns/mynet

CONTAINER INTROSPECTION
  PID=$(docker inspect -f '{{.State.Pid}}' myc)
  lsns --task $PID                      All namespaces of container
  nsenter --target $PID --net ip addr   Inspect container network
  nsenter --target $PID --all bash      Become the container

LIMITS (sysctl)
  user.max_user_namespaces              Max user namespaces (0 = disable)
  user.max_net_namespaces               Max network namespaces
  user.max_pid_namespaces               Max PID namespaces
  user.max_mnt_namespaces               Max mount namespaces
```

---

*See `man 7 namespaces`, `man 1 unshare`, `man 1 nsenter`, `man 8 ip-netns`, `man 8 lsns`,*
*and https://man7.org/linux/man-pages/man7/namespaces.7.html for full documentation.*
