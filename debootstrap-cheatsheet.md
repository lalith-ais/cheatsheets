# debootstrap Cheatsheet

A complete reference for `debootstrap` — the tool that installs a Debian base system into a directory, used for chroots, containers, custom installs, and system rescue.

---

## Overview

`debootstrap` bootstraps a minimal Debian (or derivative) system into a target directory without needing an existing Debian installation. It works in two stages:

| Stage | What happens |
|-------|-------------|
| **Stage 1** | Downloads and unpacks essential packages into the target directory |
| **Stage 2** | Runs inside a chroot to configure and install the base system |

---

## Syntax

```bash
debootstrap [options] <suite> <target> [<mirror>] [<script>]
```

| Argument | Description |
|----------|-------------|
| `<suite>` | Release codename e.g. `bookworm`, `jammy`, `trixie` |
| `<target>` | Directory to install into e.g. `/mnt/debroot` |
| `<mirror>` | APT mirror URL (optional, defaults to Debian CDN) |
| `<script>` | Bootstrap script to use (optional, usually auto-detected) |

---

## Installation

```bash
# Debian / Ubuntu
apt install debootstrap

# Fedora / RHEL / CentOS
dnf install debootstrap

# Arch Linux
pacman -S debootstrap

# From source (latest version)
git clone https://salsa.debian.org/installer-team/debootstrap.git
```

---

## Debian Release Codenames

| Release | Codename | Status |
|---------|----------|--------|
| Debian 13 | `trixie` | Testing |
| Debian 12 | `bookworm` | Stable |
| Debian 11 | `bullseye` | Oldstable |
| Debian 10 | `buster` | LTS |
| Debian 9 | `stretch` | ELTS |

### Ubuntu LTS Codenames

| Release | Codename |
|---------|----------|
| Ubuntu 24.04 | `noble` |
| Ubuntu 22.04 | `jammy` |
| Ubuntu 20.04 | `focal` |
| Ubuntu 18.04 | `bionic` |

### Symbolic suite names (Debian)
| Name | Points to |
|------|-----------|
| `stable` | Current stable release |
| `testing` | Next stable |
| `unstable` / `sid` | Rolling development |
| `experimental` | Pre-unstable |

---

## Options Reference

### Architecture

| Option | Description |
|--------|-------------|
| `--arch=<arch>` | Target architecture (default: host architecture) |

Common architectures:

| Value | Description |
|-------|-------------|
| `amd64` | 64-bit x86 (most common) |
| `i386` | 32-bit x86 |
| `arm64` | 64-bit ARM (Raspberry Pi 4+, Apple Silicon via QEMU) |
| `armhf` | 32-bit ARM hard-float |
| `armel` | 32-bit ARM soft-float |
| `riscv64` | 64-bit RISC-V |
| `ppc64el` | 64-bit PowerPC little-endian |
| `s390x` | IBM Z (mainframe) |
| `mips64el` | 64-bit MIPS little-endian |

### Variants

| Option | Description |
|--------|-------------|
| `--variant=minbase` | Minimal — only `Essential: yes` packages + apt |
| `--variant=buildd` | Build environment — adds build tools |
| `--variant=fakechroot` | Use fakechroot (no root required) |
| `--variant=scratchbox` | For scratchbox cross-compilation environments |
| (none) | Default — standard base system |

### Package Selection

| Option | Description |
|--------|-------------|
| `--include=<pkg1,pkg2>` | Additional packages to install |
| `--exclude=<pkg1,pkg2>` | Packages to exclude (use carefully) |
| `--components=<c1,c2>` | Repository components (default: `main`) |

### Authentication & Security

| Option | Description |
|--------|-------------|
| `--keyring=<file>` | GPG keyring to use for verification |
| `--no-check-gpg` | Skip GPG verification (not recommended) |
| `--no-check-certificate` | Skip SSL certificate checks |
| `--force-check-gpg` | Force GPG check even if not normally done |

### Staged / Cross Bootstrapping

| Option | Description |
|--------|-------------|
| `--foreign` | Stage 1 only — for cross-architecture bootstrapping |
| `--second-stage` | Run stage 2 inside the chroot |
| `--second-stage-target=<dir>` | Target dir for second stage (when dir differs inside chroot) |
| `--extractor=<tool>` | Package extractor: `ar`, `dpkg-deb` |

### Proxy & Network

| Option | Description |
|--------|-------------|
| `--no-resolve-deps` | Skip dependency resolution |
| Set `http_proxy` env var | Use HTTP proxy for downloads |

### Caching & Offline

| Option | Description |
|--------|-------------|
| `--cache-dir=<dir>` | Cache downloaded packages in `<dir>` |
| `--unpack-tarball=<file>` | Use pre-downloaded tarball instead of network |
| `--make-tarball=<file>` | Download packages and save as tarball (stage 1 only) |
| `--keep-debootstrap-dir` | Keep `/debootstrap` dir after completion |

### Output

| Option | Description |
|--------|-------------|
| `--verbose` | Show more detail during bootstrapping |
| `--quiet` | Suppress most output (errors only) |
| `--print-debs` | Print list of packages that would be installed and exit |

---

## Basic Examples

### Minimal Debian stable chroot
```bash
debootstrap bookworm /mnt/debroot
```

### With an explicit mirror
```bash
debootstrap bookworm /mnt/debroot http://deb.debian.org/debian
```

### Minimal variant (smallest possible system)
```bash
debootstrap --variant=minbase bookworm /mnt/debroot
```

### Include extra packages during bootstrap
```bash
debootstrap --include=vim,curl,sudo,openssh-server \
    bookworm /mnt/debroot http://deb.debian.org/debian
```

### Include non-free components
```bash
debootstrap --components=main,contrib,non-free,non-free-firmware \
    bookworm /mnt/debroot http://deb.debian.org/debian
```

### Ubuntu bootstrap
```bash
debootstrap jammy /mnt/ubuntu http://archive.ubuntu.com/ubuntu
```

### Ubuntu with Universe component
```bash
debootstrap --components=main,universe,restricted,multiverse \
    jammy /mnt/ubuntu http://archive.ubuntu.com/ubuntu
```

### Build environment variant
```bash
debootstrap --variant=buildd bookworm /mnt/buildd
```

### Specific architecture on same machine
```bash
debootstrap --arch=i386 bookworm /mnt/deb32 http://deb.debian.org/debian
```

### Via proxy
```bash
http_proxy=http://proxy.example.com:3128 \
    debootstrap bookworm /mnt/debroot http://deb.debian.org/debian
```

---

## Cross-Architecture Bootstrapping

When the target architecture differs from the host, stage 2 cannot run natively and requires `qemu-user-static`.

### Install QEMU user static emulation
```bash
apt install qemu-user-static binfmt-support
update-binfmts --enable                   # enable binfmt handlers
```

### Stage 1 — download and unpack only
```bash
debootstrap --foreign --arch=arm64 \
    bookworm /mnt/arm64root http://deb.debian.org/debian
```

### Copy QEMU binary into chroot
```bash
cp /usr/bin/qemu-aarch64-static /mnt/arm64root/usr/bin/
# For armhf:
cp /usr/bin/qemu-arm-static /mnt/arm64root/usr/bin/
```

### Stage 2 — complete configuration inside chroot
```bash
chroot /mnt/arm64root /debootstrap/debootstrap --second-stage
```

### QEMU binary names by architecture

| Target arch | QEMU binary |
|-------------|-------------|
| `arm64` / `aarch64` | `qemu-aarch64-static` |
| `armhf` / `armel` | `qemu-arm-static` |
| `riscv64` | `qemu-riscv64-static` |
| `ppc64el` | `qemu-ppc64le-static` |
| `s390x` | `qemu-s390x-static` |
| `mips64el` | `qemu-mips64el-static` |
| `i386` on amd64 | Not needed (native execution) |

---

## Offline / Cached Bootstrapping

### Create a tarball of packages (no installation)
```bash
debootstrap --make-tarball=/tmp/bookworm-packages.tar \
    bookworm /tmp/bookworm-stage1 http://deb.debian.org/debian
```

### Bootstrap from the tarball (no network needed)
```bash
debootstrap --unpack-tarball=/tmp/bookworm-packages.tar \
    bookworm /mnt/debroot
```

### Cache packages for reuse
```bash
debootstrap --cache-dir=/var/cache/debootstrap \
    bookworm /mnt/debroot http://deb.debian.org/debian
```

---

## After Bootstrapping — Entering the Chroot

### Mount required virtual filesystems
```bash
mount --bind /proc  /mnt/debroot/proc
mount --bind /sys   /mnt/debroot/sys
mount --bind /dev   /mnt/debroot/dev
mount --bind /dev/pts /mnt/debroot/dev/pts
mount --bind /run   /mnt/debroot/run
```

### Enter the chroot
```bash
chroot /mnt/debroot /bin/bash
```

### Or use a one-liner that mounts and enters
```bash
chroot /mnt/debroot /usr/bin/env -i \
    HOME=/root TERM="$TERM" PS1='(chroot) \u:\w\$ ' \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    /bin/bash --login
```

### Using systemd-nspawn (cleaner alternative)
```bash
apt install systemd-container
systemd-nspawn -D /mnt/debroot        # basic shell
systemd-nspawn -bD /mnt/debroot       # boot the container fully
```

### Unmounting after use
```bash
umount /mnt/debroot/run
umount /mnt/debroot/dev/pts
umount /mnt/debroot/dev
umount /mnt/debroot/sys
umount /mnt/debroot/proc
```

### Lazy unmount (if busy)
```bash
umount -l /mnt/debroot/proc
umount -l /mnt/debroot/sys
umount -l /mnt/debroot/dev/pts
umount -l /mnt/debroot/dev
umount -l /mnt/debroot/run
```

---

## Essential Post-Bootstrap Configuration

After entering the chroot, you typically need to configure:

### Set hostname
```bash
echo "myhostname" > /etc/hostname
```

### Set locale
```bash
apt install locales
dpkg-reconfigure locales
# or non-interactively:
echo "en_GB.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
update-locale LANG=en_GB.UTF-8
```

### Set timezone
```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
dpkg-reconfigure tzdata
```

### Configure /etc/hosts
```bash
cat > /etc/hosts << 'EOF'
127.0.0.1   localhost
127.0.1.1   myhostname

::1         localhost ip6-localhost ip6-loopback
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
EOF
```

### Set root password
```bash
passwd root
```

### Create a user
```bash
useradd -m -s /bin/bash -G sudo username
passwd username
```

### Configure APT sources
```bash
cat > /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
EOF
apt update
```

### Install and configure networking
```bash
apt install ifupdown iproute2 iputils-ping resolvconf

cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF
```

### Install kernel and bootloader (for full installs)
```bash
apt install linux-image-amd64 grub-pc

# Configure grub (from outside chroot, or after mounting /boot)
grub-install /dev/sda
update-grub
```

### Install SSH server
```bash
apt install openssh-server
systemctl enable ssh
```

---

## Full System Installation Walkthrough

### Step 1 — Partition and format target disk
```bash
fdisk /dev/sda                    # create partitions
# or
gdisk /dev/sda                    # for GPT / UEFI

mkfs.ext4 /dev/sda2               # root partition
mkswap /dev/sda1                  # swap
# For UEFI:
mkfs.fat -F32 /dev/sda1           # EFI system partition
```

### Step 2 — Mount target
```bash
mount /dev/sda2 /mnt
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi    # UEFI only
```

### Step 3 — Bootstrap base system
```bash
debootstrap --arch=amd64 \
    --include=linux-image-amd64,grub-efi-amd64,sudo,vim,curl \
    bookworm /mnt http://deb.debian.org/debian
```

### Step 4 — Generate fstab
```bash
# From outside chroot
apt install genfstab                        # if available
genfstab -U /mnt >> /mnt/etc/fstab

# Or manually
blkid /dev/sda2                            # get UUID
cat >> /mnt/etc/fstab << EOF
UUID=xxxx-xxxx  /     ext4  errors=remount-ro  0  1
UUID=yyyy-yyyy  none  swap  sw                 0  0
EOF
```

### Step 5 — Mount virtual filesystems and chroot
```bash
for dir in proc sys dev dev/pts run; do
    mount --bind /$dir /mnt/$dir
done
chroot /mnt /bin/bash
```

### Step 6 — Configure inside chroot
```bash
echo "myhostname" > /etc/hostname
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
dpkg-reconfigure locales tzdata
passwd root
useradd -m -G sudo myuser && passwd myuser

cat > /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main
deb http://security.debian.org/debian-security bookworm-security main
EOF
apt update
```

### Step 7 — Install bootloader (UEFI)
```bash
apt install grub-efi-amd64 efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi
update-grub
```

### Step 7 — Install bootloader (BIOS/MBR)
```bash
apt install grub-pc
grub-install /dev/sda
update-grub
```

### Step 8 — Exit and unmount
```bash
exit
for dir in run dev/pts dev sys proc; do
    umount /mnt/$dir
done
umount /mnt/boot/efi 2>/dev/null || true
umount /mnt
reboot
```

---

## Container & Chroot Use Cases

### Create a minimal build chroot
```bash
debootstrap --variant=buildd \
    --include=build-essential,devscripts,debhelper \
    bookworm /srv/chroot/bookworm-build

# Enter for building
chroot /srv/chroot/bookworm-build /bin/bash
```

### Create an LXC container manually
```bash
debootstrap bookworm /var/lib/lxc/mycontainer/rootfs
# Then configure /var/lib/lxc/mycontainer/config
```

### Create a Docker base image from debootstrap
```bash
debootstrap --variant=minbase bookworm /tmp/debian-docker
tar -C /tmp/debian-docker -czf debian-bookworm-base.tar.gz .
docker import debian-bookworm-base.tar.gz debian:bookworm-custom
docker run -it debian:bookworm-custom /bin/bash
```

### Create a systemd-nspawn container
```bash
debootstrap bookworm /var/lib/machines/mycontainer
systemd-nspawn -D /var/lib/machines/mycontainer
# Or register and manage with machinectl:
machinectl start mycontainer
machinectl login mycontainer
```

---

## Using mmdebstrap (Modern Alternative)

`mmdebstrap` is a faster, more featureful drop-in replacement using apt's dependency resolver.

```bash
apt install mmdebstrap

# Basic usage (same interface as debootstrap)
mmdebstrap bookworm /mnt/debroot

# Multiple mirrors
mmdebstrap bookworm /mnt/debroot \
    'deb http://deb.debian.org/debian bookworm main contrib non-free'

# Create a tarball directly
mmdebstrap bookworm - | gzip > bookworm-base.tar.gz

# Create OCI/Docker image
mmdebstrap bookworm - --format=tar | docker import - debian:bookworm

# Use without root (via fakechroot)
mmdebstrap --mode=fakechroot bookworm /mnt/debroot

# Include extra packages
mmdebstrap --include=vim,curl,sudo bookworm /mnt/debroot

# Minimal variant
mmdebstrap --variant=minbase bookworm /mnt/debroot
```

---

## Automation Script Template

```bash
#!/bin/bash
# bootstrap-debian.sh — automated Debian chroot setup

set -euo pipefail

SUITE="bookworm"
TARGET="/mnt/debroot"
MIRROR="http://deb.debian.org/debian"
PACKAGES="vim,curl,sudo,openssh-server,locales"
HOSTNAME="mydebian"
TIMEZONE="Europe/London"
LOCALE="en_GB.UTF-8"

echo "==> Creating target directory"
mkdir -p "$TARGET"

echo "==> Running debootstrap stage 1 + 2"
debootstrap \
    --arch=amd64 \
    --include="$PACKAGES" \
    --components=main,contrib,non-free,non-free-firmware \
    "$SUITE" "$TARGET" "$MIRROR"

echo "==> Mounting virtual filesystems"
for fs in proc sys dev dev/pts run; do
    mount --bind "/$fs" "$TARGET/$fs"
done

echo "==> Configuring chroot"
chroot "$TARGET" /bin/bash -s << CHROOT_EOF
set -e

# Hostname
echo "$HOSTNAME" > /etc/hostname

# Timezone
ln -sf /usr/share/zoneinfo/$TIMEZONE /etc/localtime
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure tzdata

# Locale
echo "$LOCALE UTF-8" > /etc/locale.gen
locale-gen
update-locale LANG=$LOCALE

# APT sources
cat > /etc/apt/sources.list << 'SOURCES'
deb http://deb.debian.org/debian $SUITE main contrib non-free non-free-firmware
deb http://deb.debian.org/debian $SUITE-updates main
deb http://security.debian.org/debian-security $SUITE-security main
SOURCES

apt-get update -q
apt-get upgrade -yq

echo "Chroot configuration complete."
CHROOT_EOF

echo "==> Unmounting virtual filesystems"
for fs in run dev/pts dev sys proc; do
    umount "$TARGET/$fs"
done

echo "==> Done. Enter with: chroot $TARGET /bin/bash"
```

---

## Troubleshooting

### E: No such script: /usr/share/debootstrap/scripts/<suite>
The suite is not recognised. Check spelling, or use the full codename instead of a symbolic name. Update debootstrap:
```bash
apt install --reinstall debootstrap
```

### W: Cannot check Release signature — no keyring
```bash
apt install debian-archive-keyring
# or specify one:
debootstrap --keyring=/usr/share/keyrings/debian-archive-keyring.gpg \
    bookworm /mnt/debroot
```

### Hanging or slow downloads — use a closer mirror
```bash
# UK
debootstrap bookworm /mnt/debroot http://ftp.uk.debian.org/debian
# Germany
debootstrap bookworm /mnt/debroot http://ftp.de.debian.org/debian
# Auto-select via redirector
debootstrap bookworm /mnt/debroot http://httpredir.debian.org/debian
```

### chroot: failed to run command — Exec format error (cross-arch)
QEMU static binary is missing or binfmt is not registered:
```bash
apt install qemu-user-static binfmt-support
update-binfmts --enable
cp /usr/bin/qemu-aarch64-static /mnt/arm64root/usr/bin/
```

### dpkg: error processing package — in chroot
Virtual filesystems are not mounted:
```bash
mount --bind /proc /mnt/debroot/proc
mount --bind /sys  /mnt/debroot/sys
mount --bind /dev  /mnt/debroot/dev
```

### No network inside chroot
Copy the host resolver config:
```bash
cp /etc/resolv.conf /mnt/debroot/etc/resolv.conf
```

### /proc/mounts: No such file or directory
Mount `/proc` before entering the chroot:
```bash
mount --bind /proc /mnt/debroot/proc
```

---

## Key Files & Directories

| Path | Description |
|------|-------------|
| `/usr/share/debootstrap/scripts/` | Per-suite bootstrap scripts |
| `/usr/share/debootstrap/functions` | Shared shell functions |
| `<target>/debootstrap/` | Temporary working directory (deleted on success) |
| `<target>/debootstrap/debootstrap` | Stage 2 script (run inside chroot) |
| `<target>/var/lib/dpkg/` | dpkg package database |
| `<target>/etc/apt/sources.list` | APT sources configured post-bootstrap |
| `/usr/share/keyrings/debian-archive-keyring.gpg` | Debian GPG keyring |

---

## See Also

| Tool | Purpose |
|------|---------|
| `mmdebstrap` | Modern, rootless debootstrap alternative |
| `systemd-nspawn` | Lightweight container manager using debootstrap roots |
| `schroot` | Manage multiple named chroot environments |
| `pbuilder` | Debian package build environment using debootstrap |
| `sbuild` | Debian package builder (uses schroot/debootstrap) |
| `multistrap` | Bootstrap from multiple repositories simultaneously |
| `lxc` | Linux containers, can use debootstrap roots |
| `docker import` | Import a debootstrap tarball as a Docker image |
| `qemu-user-static` | User-space emulation for cross-architecture chroots |
| `fakechroot` | Run debootstrap without root privileges |

---

*`debootstrap` is maintained by the Debian Installer team. Always check `man debootstrap` and the scripts in `/usr/share/debootstrap/scripts/` for suite-specific behaviour.*
