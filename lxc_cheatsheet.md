# LXC / LXD Cheatsheet

## Table of Contents
1. [Overview & Concepts](#overview--concepts)
2. [Installation](#installation)
3. [LXD Initialisation](#lxd-initialisation)
4. [Instance Management](#instance-management)
5. [Images](#images)
6. [Console & Shell Access](#console--shell-access)
7. [File Transfer](#file-transfer)
8. [Snapshots & Backups](#snapshots--backups)
9. [Configuration](#configuration)
10. [Devices](#devices)
11. [Networking](#networking)
12. [Storage](#storage)
13. [Profiles](#profiles)
14. [Projects](#projects)
15. [Resource Limits (cgroups)](#resource-limits-cgroups)
16. [Security](#security)
17. [Low-level LXC Commands](#low-level-lxc-commands)
18. [lxc-* Tool Reference](#lxc--tool-reference)
19. [Useful Patterns](#useful-patterns)
20. [Quick Reference Card](#quick-reference-card)

---

## Overview & Concepts

| Term | Description |
|------|-------------|
| **LXC** | Linux Containers — low-level container runtime using kernel namespaces & cgroups |
| **LXD** | Higher-level container & VM manager built on top of LXC (now Incus fork exists too) |
| **Container** | Lightweight OS-level virtualisation — shares the host kernel |
| **VM** | Full virtual machine managed by LXD (via QEMU) |
| **Image** | Read-only template used to create instances |
| **Profile** | Reusable configuration applied to one or more instances |
| **Storage pool** | Where container/VM disk images are stored (dir, btrfs, zfs, lvm, ceph) |
| **Network bridge** | Virtual switch that connects containers (default: `lxdbr0`) |
| **Snapshot** | Point-in-time copy of an instance's state |
| **Project** | Namespace for isolating groups of instances, images, and profiles |

---

## Installation

```bash
# Ubuntu / Debian — via snap (recommended)
snap install lxd
snap install lxd --channel=latest/stable

# Add your user to the lxd group
sudo usermod -aG lxd $USER
newgrp lxd                        # Apply without re-login

# Alpine Linux
apk add lxc lxc-templates

# Arch Linux
pacman -S lxd

# Check version
lxd --version
lxc --version

# Incus (community fork of LXD post-2023)
# https://linuxcontainers.org/incus/
# Commands are identical but use `incus` instead of `lxc`
```

---

## LXD Initialisation

```bash
# Interactive setup wizard
lxd init

# Minimal non-interactive setup (good for scripts)
lxd init --minimal

# Preseed from YAML
lxd init --preseed < preseed.yaml
```

**preseed.yaml example:**

```yaml
config: {}
networks:
  - name: lxdbr0
    type: bridge
    config:
      ipv4.address: 10.0.0.1/24
      ipv4.nat: "true"
      ipv6.address: none
storage_pools:
  - name: default
    driver: dir
    config:
      source: /var/lib/lxd/storage-pools/default
profiles:
  - name: default
    devices:
      eth0:
        name: eth0
        network: lxdbr0
        type: nic
      root:
        path: /
        pool: default
        type: disk
```

---

## Instance Management

### Create & Launch

```bash
# Launch a container (create + start)
lxc launch ubuntu:24.04 mycontainer
lxc launch ubuntu:22.04 mycontainer
lxc launch images:alpine/3.19 myalpine
lxc launch images:debian/12 mydebian

# Launch a VM (not a container)
lxc launch ubuntu:24.04 myvm --vm

# Create without starting
lxc init ubuntu:24.04 mycontainer
lxc init ubuntu:24.04 myvm --vm

# Launch with a specific profile
lxc launch ubuntu:24.04 mycontainer --profile default --profile gpu

# Launch with inline config
lxc launch ubuntu:24.04 mycontainer -c limits.memory=512MB -c limits.cpu=2

# Launch with a specific storage pool
lxc launch ubuntu:24.04 mycontainer -s mypool

# Launch with a specific network
lxc launch ubuntu:24.04 mycontainer -n lxdbr0
```

### Start / Stop / Restart

```bash
lxc start   mycontainer
lxc stop    mycontainer
lxc stop    mycontainer --force         # Equivalent to power cut
lxc restart mycontainer
lxc restart mycontainer --force
lxc pause   mycontainer                 # Freeze (SIGSTOP)
lxc resume  mycontainer                 # Unfreeze
```

### Delete & Rebuild

```bash
lxc delete mycontainer                  # Must be stopped first
lxc delete mycontainer --force          # Stop and delete in one step

# Rebuild from image (wipe and re-initialise)
lxc rebuild mycontainer ubuntu:24.04
```

### List & Info

```bash
lxc list                                # All instances
lxc list --format table                 # Table format (default)
lxc list --format json                  # JSON output
lxc list --format csv                   # CSV
lxc list -c n,s,4,P                     # Custom columns
                                        # n=name s=state 4=IPv4 P=pid

# Filter
lxc list status=running
lxc list "^my"                          # Name regex
lxc list type=container
lxc list type=virtual-machine

# Detailed info
lxc info mycontainer
lxc info mycontainer --show-log         # Include recent log
```

### Column reference for `lxc list -c`

| Code | Column |
|------|--------|
| `n` | Name |
| `s` | State |
| `4` | IPv4 address |
| `6` | IPv6 address |
| `t` | Type |
| `P` | PID |
| `S` | Snapshots count |
| `b` | Storage pool |
| `p` | Project |
| `l` | Location (cluster node) |

---

## Images

```bash
# List available remote images
lxc image list images:                  # Community image server
lxc image list ubuntu:                  # Ubuntu image server
lxc image list images: alpine           # Filter by name
lxc image list images: arch=amd64

# List local images
lxc image list

# Show image info
lxc image info ubuntu:24.04

# Copy image from remote to local cache
lxc image copy ubuntu:24.04 local: --alias ubuntu-noble

# Export image to a file
lxc image export ubuntu:24.04 ./my-image
lxc image export mycontainer ./backup   # Export from a container

# Import image from a file
lxc image import ./my-image.tar.gz --alias myimage
lxc image import meta.tar.xz rootfs.tar.xz --alias myimage

# Create image from a container (snapshot required)
lxc snapshot mycontainer snap0
lxc publish mycontainer/snap0 --alias my-golden-image

# Publish a running container as image
lxc publish mycontainer --alias my-image --force

# Delete a local image
lxc image delete my-golden-image

# Refresh cached images
lxc image refresh

# Manage aliases
lxc image alias create myalias <fingerprint>
lxc image alias delete myalias
lxc image alias list
```

### Common Image Remotes

| Remote | Notes |
|--------|-------|
| `ubuntu:` | Official Ubuntu releases |
| `ubuntu-daily:` | Ubuntu daily builds |
| `images:` | Community (Alpine, Arch, Debian, Fedora, etc.) |
| `local:` | Local LXD instance |

```bash
# Manage remotes
lxc remote list
lxc remote add myremote https://10.0.0.1:8443
lxc remote remove myremote
lxc remote set-default myremote
```

---

## Console & Shell Access

```bash
# Open a shell inside the container
lxc exec mycontainer -- bash
lxc exec mycontainer -- /bin/sh
lxc exec mycontainer -- su - alice      # As a specific user

# Run a single command
lxc exec mycontainer -- ls /etc
lxc exec mycontainer -- apt update

# Run as a specific user
lxc exec mycontainer --user 1000 -- whoami
lxc exec mycontainer --user 0 -- bash   # Root

# Set environment variables
lxc exec mycontainer --env MYVAR=hello -- bash

# Attach to the console (tty0)
lxc console mycontainer
lxc console mycontainer --type=console  # Serial console
lxc console mycontainer --type=vga      # VGA console (VMs only)
# Detach from console: Ctrl+a q
```

---

## File Transfer

```bash
# Copy file from host to container
lxc file push localfile.txt mycontainer/root/localfile.txt

# Copy directory recursively
lxc file push -r mydir/ mycontainer/opt/mydir/

# Copy file from container to host
lxc file pull mycontainer/etc/nginx/nginx.conf ./nginx.conf

# Copy directory from container
lxc file pull -r mycontainer/var/log/nginx/ ./nginx-logs/

# Edit a file in-place (opens $EDITOR)
lxc file edit mycontainer/etc/hosts

# Mount a directory into a container at runtime
lxc config device add mycontainer myshare disk \
  source=/host/path \
  path=/container/path
```

---

## Snapshots & Backups

### Snapshots

```bash
# Create snapshot
lxc snapshot mycontainer
lxc snapshot mycontainer snapname
lxc snapshot mycontainer snapname --stateful   # Include RAM state (requires CRIU)

# List snapshots
lxc info mycontainer                           # Snapshots listed at bottom

# Restore snapshot
lxc restore mycontainer snapname
lxc restore mycontainer snapname --stateful

# Delete snapshot
lxc delete mycontainer/snapname

# Copy snapshot to new container
lxc copy mycontainer/snapname newcontainer

# Rename snapshot
lxc move mycontainer/oldsnap mycontainer/newsnap

# Automatic snapshots via config
lxc config set mycontainer snapshots.schedule "0 * * * *"    # Hourly
lxc config set mycontainer snapshots.expiry 7d               # Keep for 7 days
lxc config set mycontainer snapshots.pattern "auto-%d"       # Name pattern
```

### Backups (Export/Import)

```bash
# Export container to tarball
lxc export mycontainer mycontainer-backup.tar.gz
lxc export mycontainer mycontainer-backup.tar.gz --optimized-storage
lxc export mycontainer mycontainer-backup.tar.gz --instance-only   # Skip snapshots

# Import from tarball
lxc import mycontainer-backup.tar.gz
lxc import mycontainer-backup.tar.gz --storage mypool

# Copy to another LXD server
lxc copy mycontainer remote2:mycontainer
lxc move mycontainer remote2:mycontainer    # Move (delete from source)
```

---

## Configuration

```bash
# Show all config
lxc config show mycontainer
lxc config show mycontainer --expanded     # Include profile config

# Get / set / unset a key
lxc config get mycontainer limits.memory
lxc config set mycontainer limits.memory 1GB
lxc config unset mycontainer limits.memory

# Edit full config in $EDITOR (YAML)
lxc config edit mycontainer

# Global LXD server config
lxc config show
lxc config set core.https_address :8443
lxc config set core.trust_password mysecret
```

### Common Config Keys

```bash
# Resource limits
lxc config set mycontainer limits.cpu 2
lxc config set mycontainer limits.cpu.allowance 50%
lxc config set mycontainer limits.memory 2GB
lxc config set mycontainer limits.memory.swap false
lxc config set mycontainer limits.processes 200

# Boot behaviour
lxc config set mycontainer boot.autostart true
lxc config set mycontainer boot.autostart.priority 10
lxc config set mycontainer boot.autostart.delay 5

# Snapshots
lxc config set mycontainer snapshots.schedule "0 2 * * *"
lxc config set mycontainer snapshots.expiry 14d

# Cloud-init
lxc config set mycontainer user.user-data - << 'EOF'
#cloud-config
packages:
  - nginx
runcmd:
  - systemctl enable nginx
EOF
```

---

## Devices

```bash
# List / show devices
lxc config device list mycontainer
lxc config device show mycontainer

# Add / remove a device
lxc config device add mycontainer <name> <type> [key=value ...]
lxc config device remove mycontainer <name>

# Override a device from a profile
lxc config device override mycontainer eth0 ipv4.address=10.0.0.100
```

### Disk Devices

```bash
# Bind-mount a host directory
lxc config device add mycontainer myshare disk \
  source=/host/path \
  path=/container/path \
  readonly=true

# Raw block device
lxc config device add mycontainer myblock disk \
  source=/dev/sdb \
  path=/dev/sdb

# Resize root disk (zfs/btrfs/lvm pools)
lxc config device override mycontainer root size=20GB
```

### Network Devices (NIC)

```bash
# Bridged NIC
lxc config device add mycontainer eth1 nic \
  nictype=bridged parent=lxdbr0

# Macvlan NIC (direct physical network access)
lxc config device add mycontainer eth1 nic \
  nictype=macvlan parent=eth0

# Static IP on a NIC
lxc config device override mycontainer eth0 \
  ipv4.address=10.0.0.50
```

### GPU & USB Devices

```bash
# Pass through all GPUs
lxc config device add mycontainer mygpu gpu

# Pass through a specific GPU
lxc config device add mycontainer mygpu gpu \
  vendorid=10de productid=1b80

# Pass through a USB device
lxc config device add mycontainer myusb usb \
  vendorid=046d productid=c52b

# Unix char device
lxc config device add mycontainer snd unix-char \
  source=/dev/snd/controlC0 \
  path=/dev/snd/controlC0
```

---

## Networking

```bash
# List / inspect networks
lxc network list
lxc network info lxdbr0
lxc network show lxdbr0

# Create networks
lxc network create lxdbr1 \
  ipv4.address=10.1.0.1/24 \
  ipv4.nat=true \
  ipv6.address=none

lxc network create mymacvlan \
  --type=macvlan \
  parent=eth0

# Edit / configure
lxc network edit lxdbr0
lxc network set lxdbr0 ipv4.dhcp.ranges 10.0.0.100-10.0.0.200

# Delete
lxc network delete lxdbr1

# DHCP leases
lxc network list-leases lxdbr0

# Attach / detach
lxc network attach lxdbr1 mycontainer eth1
lxc network detach lxdbr1 mycontainer eth1

# Port forward (proxy device)
lxc config device add mycontainer webproxy proxy \
  listen=tcp:0.0.0.0:8080 \
  connect=tcp:127.0.0.1:80
```

### Common Network Config Keys

```bash
lxc network set lxdbr0 ipv4.address 10.0.0.1/24
lxc network set lxdbr0 ipv4.nat true
lxc network set lxdbr0 ipv4.dhcp true
lxc network set lxdbr0 ipv4.dhcp.ranges 10.0.0.50-10.0.0.200
lxc network set lxdbr0 dns.domain lxd.internal
lxc network set lxdbr0 dns.mode managed
```

---

## Storage

```bash
# List / inspect pools
lxc storage list
lxc storage info mypool
lxc storage show mypool

# Create pools
lxc storage create mydir   dir   source=/mnt/storage
lxc storage create mybtrfs btrfs source=/dev/sdb
lxc storage create myzfs   zfs   source=tank/lxd
lxc storage create mylvm   lvm   source=/dev/sdc

# Edit / configure pool
lxc storage edit mypool
lxc storage set mypool volume.size 20GB

# Delete pool
lxc storage delete mypool

# Volumes
lxc storage volume list mypool
lxc storage volume create mypool myvol
lxc storage volume create mypool myvol size=10GB
lxc storage volume delete mypool myvol
lxc storage volume show mypool myvol

# Attach a custom volume to a container
lxc storage volume attach mypool myvol mycontainer mydevname /mnt/data
lxc storage volume detach mypool myvol mycontainer

# Copy / move volumes
lxc storage volume copy mypool/myvol mypool/myvol-backup
lxc storage volume move mypool/myvol newpool/myvol

# Snapshot a volume
lxc storage volume snapshot mypool myvol snapname
lxc storage volume restore mypool myvol snapname
```

---

## Profiles

```bash
# List / show
lxc profile list
lxc profile show default
lxc profile show myprofile

# Create / edit / delete
lxc profile create myprofile
lxc profile edit myprofile
lxc profile delete myprofile
lxc profile copy myprofile myprofile-backup

# Set config and devices
lxc profile set myprofile limits.memory 2GB
lxc profile device add myprofile eth0 nic nictype=bridged parent=lxdbr0
lxc profile device remove myprofile eth0

# Apply to containers
lxc profile assign mycontainer default,myprofile   # Replace all profiles
lxc profile add mycontainer myprofile              # Append a profile
lxc profile remove mycontainer myprofile           # Remove a profile
```

**Example: GPU profile**

```yaml
# lxc profile edit gpu-profile
config:
  environment.NVIDIA_VISIBLE_DEVICES: all
  environment.NVIDIA_DRIVER_CAPABILITIES: all
description: NVIDIA GPU pass-through
devices:
  gpu:
    type: gpu
name: gpu-profile
```

**Example: Standard resource-limited profile**

```yaml
config:
  limits.cpu: "2"
  limits.memory: 2GB
  limits.memory.swap: "false"
  boot.autostart: "true"
description: Standard limited container
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    size: 20GB
    type: disk
name: standard
```

---

## Projects

```bash
# List / inspect
lxc project list
lxc project info myproject
lxc project show myproject

# Create
lxc project create myproject
lxc project create myproject \
  --config features.images=true \
  --config features.profiles=true

# Switch active project
lxc project switch myproject
lxc project switch default

# Edit / delete
lxc project edit myproject
lxc project delete myproject      # Must be empty

# Target a project in any command
lxc --project myproject list
lxc --project myproject launch ubuntu:24.04 mycontainer
```

---

## Resource Limits (cgroups)

```bash
# CPU
lxc config set mycontainer limits.cpu 2                      # vCPU count
lxc config set mycontainer limits.cpu.allowance 50%          # CPU time quota
lxc config set mycontainer limits.cpu.allowance 200ms/500ms  # Period/quota
lxc config set mycontainer limits.cpu.priority 5             # 0-10

# Memory
lxc config set mycontainer limits.memory 2GB
lxc config set mycontainer limits.memory.swap false
lxc config set mycontainer limits.memory.enforce hard         # hard|soft

# Disk I/O
lxc config set mycontainer limits.disk.priority 5            # 0-10

# Network I/O (per NIC device)
lxc config device override mycontainer eth0 \
  limits.ingress=100Mbit \
  limits.egress=100Mbit

# Processes
lxc config set mycontainer limits.processes 500

# Live resource usage
lxc info mycontainer       # Shows CPU/memory/disk usage
lxc top                    # Live top-like view of all containers
```

---

## Security

### Privilege Model

```bash
# Unprivileged (default — recommended)
lxc config set mycontainer security.privileged false

# Privileged (UID 0 inside = UID 0 on host — use with caution)
lxc config set mycontainer security.privileged true

# Allow nested containers
lxc config set mycontainer security.nesting true
```

### ID Mapping

```bash
# Custom idmap
lxc config set mycontainer raw.idmap "both 1000 1000 1"

# Auto-shift UID/GID for a bind-mounted directory
lxc config device add mycontainer myshare disk \
  source=/host/path \
  path=/mnt/data \
  shift=true
```

### TLS & Remote Access

```bash
# Enable remote API
lxc config set core.https_address :8443

# Trust a client certificate
lxc config trust add /path/to/client.crt

# List / remove trusted certificates
lxc config trust list
lxc config trust remove <fingerprint>

# Add a remote LXD server
lxc remote add myserver https://10.0.0.1:8443 --accept-certificate
```

---

## Low-level LXC Commands

These are the underlying `lxc-*` tools (separate from the LXD `lxc` CLI).

```bash
# Check kernel LXC support
lxc-checkconfig

# List containers
lxc-ls
lxc-ls --fancy                    # Detailed output

# Create from template
lxc-create -n mycontainer -t download -- -d ubuntu -r focal -a amd64
lxc-create -n mycontainer -t busybox

# Start / Stop
lxc-start  -n mycontainer
lxc-start  -n mycontainer -F      # Foreground
lxc-stop   -n mycontainer
lxc-stop   -n mycontainer --kill

# Info & status
lxc-info   -n mycontainer
lxc-info   -n mycontainer -s      # State only

# Attach / exec
lxc-attach -n mycontainer
lxc-attach -n mycontainer -- /bin/bash

# Destroy
lxc-destroy -n mycontainer
lxc-destroy -n mycontainer -f     # Force (stop first)

# Clone
lxc-copy -n src -N dst
lxc-copy -n src -N dst -s         # Snapshot clone

# Snapshots
lxc-snapshot -n mycontainer -c snapname
lxc-snapshot -L -n mycontainer            # List snapshots
lxc-snapshot -r -n mycontainer snapname   # Restore

# Monitor
lxc-monitor -n mycontainer
lxc-wait -n mycontainer -s RUNNING
lxc-wait -n mycontainer -s STOPPED

# Freeze / Unfreeze
lxc-freeze   -n mycontainer
lxc-unfreeze -n mycontainer

# Checkpoint / restore (CRIU required)
lxc-checkpoint -n mycontainer -D /tmp/ckpt -s   # Stop and checkpoint
lxc-checkpoint -n mycontainer -D /tmp/ckpt -r   # Restore
```

---

## lxc-* Tool Reference

| Command | Purpose |
|---------|---------|
| `lxc-create` | Create a container from a template |
| `lxc-destroy` | Destroy a container |
| `lxc-start` | Start a container |
| `lxc-stop` | Stop a container |
| `lxc-restart` | Restart a container |
| `lxc-info` | Show container information |
| `lxc-ls` | List containers |
| `lxc-attach` | Attach to or exec in a running container |
| `lxc-execute` | Start, run a command, then stop |
| `lxc-copy` | Clone a container |
| `lxc-snapshot` | Snapshot management |
| `lxc-monitor` | Monitor container state changes |
| `lxc-wait` | Wait for a container state |
| `lxc-freeze` | Freeze all processes (cgroup freezer) |
| `lxc-unfreeze` | Unfreeze processes |
| `lxc-checkpoint` | Checkpoint / restore via CRIU |
| `lxc-top` | Live resource monitor |
| `lxc-console` | Attach to container console |
| `lxc-device` | Manage hot-plugged devices |
| `lxc-checkconfig` | Check kernel config for LXC support |
| `lxc-update-config` | Migrate legacy config files |

---

## Useful Patterns

### Port Forwarding (Proxy Device)

```bash
# TCP: host port 8080 → container port 80
lxc config device add mycontainer webproxy proxy \
  listen=tcp:0.0.0.0:8080 \
  connect=tcp:127.0.0.1:80

# SSH: host 2222 → container 22
lxc config device add mycontainer sshproxy proxy \
  listen=tcp:0.0.0.0:2222 \
  connect=tcp:127.0.0.1:22

# UDP
lxc config device add mycontainer dnsProxy proxy \
  listen=udp:0.0.0.0:5353 \
  connect=udp:127.0.0.1:53
```

### Cloud-init Configuration

```bash
lxc config set mycontainer user.user-data - << 'EOF'
#cloud-config
package_update: true
packages:
  - nginx
  - git
users:
  - name: deploy
    groups: sudo
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... mykey
runcmd:
  - systemctl enable --now nginx
EOF
lxc restart mycontainer
```

### Batch Operations

```bash
# Run a command on all running containers
lxc list --format csv -c n,s \
  | awk -F, '$2=="RUNNING"{print $1}' \
  | xargs -I{} lxc exec {} -- apt-get update -y

# Stop all containers
lxc list --format csv -c n \
  | xargs -I{} lxc stop {} --force

# Snapshot all containers
lxc list --format csv -c n \
  | xargs -I{} lxc snapshot {} auto-$(date +%F)
```

### Move Container Between Storage Pools

```bash
lxc move mycontainer mycontainer-tmp --storage newpool
lxc move mycontainer-tmp mycontainer     # Rename back if needed
```

### Nested LXD (LXD inside LXD)

```bash
lxc config set mycontainer security.nesting true
lxc config set mycontainer security.privileged true
lxc restart mycontainer
lxc exec mycontainer -- snap install lxd
```

### Live Migration (Cluster)

```bash
# Move container to another cluster node
lxc move mycontainer --target node2

# Copy without shared storage
lxc move mycontainer --target node2 --stateless
```

### Golden Image Workflow

```bash
# Build a base container
lxc launch ubuntu:24.04 golden
lxc exec golden -- bash
# ... install packages, configure ...

# Freeze and publish as an image
lxc stop golden
lxc snapshot golden snap0
lxc publish golden/snap0 --alias my-golden-24.04

# Launch instances from the image
lxc launch my-golden-24.04 app1
lxc launch my-golden-24.04 app2

# Clean up builder
lxc delete golden --force
```

---

## Quick Reference Card

```
LAUNCH & MANAGE
  lxc launch ubuntu:24.04 myc         Create and start a container
  lxc launch ubuntu:24.04 myvm --vm   Create and start a VM
  lxc init ubuntu:24.04 myc           Create without starting
  lxc start / stop / restart myc      Lifecycle control
  lxc stop myc --force                Force stop (power cut)
  lxc delete myc --force              Stop and delete
  lxc rebuild myc ubuntu:24.04        Wipe and rebuild from image
  lxc list                            List all instances
  lxc list status=running             Filter by state
  lxc info myc                        Detailed info + stats
  lxc top                             Live resource monitor

SHELL & FILES
  lxc exec myc -- bash                Shell in container
  lxc exec myc --user 1000 -- bash    Shell as specific user
  lxc exec myc --env K=V -- bash      Set env vars
  lxc console myc                     Console (Ctrl+a q to exit)
  lxc file push src myc/dest          Copy to container
  lxc file pull myc/src dest          Copy from container
  lxc file edit myc/etc/hosts         Edit file in-place

SNAPSHOTS & BACKUP
  lxc snapshot myc snapname           Create snapshot
  lxc restore myc snapname            Restore snapshot
  lxc delete myc/snapname             Delete snapshot
  lxc export myc backup.tar.gz        Export to tarball
  lxc import backup.tar.gz            Import from tarball
  lxc copy myc remote:myc             Copy to remote server
  lxc move myc remote:myc             Move to remote server

CONFIG & DEVICES
  lxc config show myc                 Show full config
  lxc config show myc --expanded      Include profile config
  lxc config set myc key value        Set a config key
  lxc config edit myc                 Edit full YAML config
  lxc config device add myc ...       Add a device
  lxc config device list myc          List devices

NETWORKING
  lxc network list                    List networks
  lxc network list-leases lxdbr0      Show DHCP leases
  lxc config device add myc proxy \
    listen=tcp:0.0.0.0:8080 \
    connect=tcp:127.0.0.1:80          Port forward

STORAGE
  lxc storage list                    List storage pools
  lxc storage volume list mypool      List volumes
  lxc storage volume create mypool v  Create a volume
  lxc storage volume attach \
    mypool vol myc name /mnt/p        Attach volume

IMAGES
  lxc image list images:              Browse community images
  lxc image list ubuntu:              Browse Ubuntu images
  lxc image copy ubuntu:24.04 local:  Cache an image locally
  lxc publish myc/snap --alias img    Publish container as image

PROFILES
  lxc profile list                    List profiles
  lxc profile create / edit myp       Create or edit a profile
  lxc profile add myc myp             Apply a profile
  lxc profile assign myc default,myp  Replace all profiles

RESOURCE LIMITS
  limits.cpu 2                        vCPU count
  limits.cpu.allowance 50%            CPU quota
  limits.memory 2GB                   Memory cap
  limits.memory.swap false            Disable swap
  limits.processes 500                Max PID count

USEFUL FLAGS
  --vm                                Launch as virtual machine
  --force                             Force stop / delete
  --profile / -p name                 Apply a profile at launch
  --storage / -s pool                 Specify storage pool
  --config / -c key=val               Inline config at launch
  --project name                      Target a specific project
  --target node                       Target a cluster node
  --expanded                          Include inherited config
```

---

*See `lxc help`, `lxc <command> --help`, or https://linuxcontainers.org/lxd/docs/ for full documentation.*
