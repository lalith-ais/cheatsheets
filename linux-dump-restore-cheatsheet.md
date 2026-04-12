# Linux dump / restore Cheatsheet

A complete reference for backing up and restoring ext2/ext3/ext4 filesystems using `dump` and `restore`.

---

## Overview

| Tool | Purpose |
|------|---------|
| `dump` | Back up an ext2/ext3/ext4 filesystem or directory tree |
| `restore` | Restore files from a dump archive |
| `rdump` | Remote dump (dump over a network to a remote tape/file) |
| `rrestore` | Remote restore |

> **Note:** `dump` operates at the filesystem block level, not the file level. It must be run on an unmounted or read-only mounted filesystem for consistency.

---

## dump — Syntax & Options

```bash
dump [options] -f <destination> <filesystem|directory>
```

### Options

| Option | Description |
|--------|-------------|
| `-0` to `-9` | Dump level. `0` = full backup; `1–9` = incremental (only files changed since last lower-level dump) |
| `-f <file>` | Destination file, device, or `host:file` for remote |
| `-u` | Update `/etc/dumpdates` after a successful dump |
| `-j` | Compress with bzip2 |
| `-z[level]` | Compress with zlib (level 1–9, default 2) |
| `-J` | Compress with xz |
| `-a` | Auto-size — don't prompt when end of tape reached |
| `-b <blocksize>` | Set block size in kilobytes (default 10) |
| `-B <records>` | Set number of blocks per volume |
| `-d <density>` | Set tape density (bits per inch) |
| `-s <feet>` | Set tape length in feet |
| `-T <date>` | Use specified date as the dump time reference (instead of `/etc/dumpdates`) |
| `-e <inode>` | Exclude inode from dump (can be repeated) |
| `-E <file>` | Exclude inodes listed in file |
| `-L <label>` | Tape label string |
| `-n` | Notify all operators in the `operator` group |
| `-q` | Quit immediately on errors instead of prompting |
| `-v` | Verbose output |
| `-W` | Show what filesystems need backing up (reads `/etc/dumpdates` and `/etc/fstab`) |
| `-w` | Like `-W` but only prints filesystems that need a dump |

---

## Dump Levels Explained

| Level | Backs up |
|-------|---------|
| `0` | **Full** — everything on the filesystem |
| `1` | All files changed since last level-0 dump |
| `2` | All files changed since last level-1 dump |
| `3–9` | All files changed since last dump of the next lower level |

### Common Backup Strategies

**Simple full + incremental:**
```
Sunday    → level 0  (full)
Monday    → level 1  (changes since Sunday)
Tuesday   → level 1  (changes since Sunday)
...
Next Sunday → level 0 (new full)
```

**Tower of Hanoi (efficient tape rotation):**
```
Week 1: 0 3 2 3 1 3 2 3
Week 2: 0 3 2 3 1 3 2 3
```

---

## dump — Examples

### Full backup to a file
```bash
dump -0uf /backup/root.dump /dev/sda1
```

### Full backup with bzip2 compression
```bash
dump -0juf /backup/root.dump.bz2 /dev/sda1
```

### Full backup with zlib compression (level 9)
```bash
dump -0z9uf /backup/root.dump.gz /dev/sda1
```

### Incremental level-1 backup
```bash
dump -1uf /backup/root-inc1.dump /dev/sda1
```

### Backup to tape device
```bash
dump -0uf /dev/st0 /dev/sda1
```

### Remote backup (via rsh/ssh)
```bash
dump -0uf user@remotehost:/backup/root.dump /dev/sda1
```

### Backup with label, update dumpdates, verbose
```bash
dump -0uvLf "root-full-$(date +%Y%m%d)" /backup/root.dump /dev/sda1
```

### Check what needs backing up
```bash
dump -W
```

### Exclude specific inodes
```bash
dump -0uf /backup/root.dump -e 12345 -e 67890 /dev/sda1
```

---

## /etc/dumpdates

The file `dump` uses to track when each filesystem was last dumped at each level.

```
/dev/sda1 0 Sun Apr  6 02:00:01 2025 +0000
/dev/sda1 1 Mon Apr  7 02:00:01 2025 +0000
```

- Updated automatically when `-u` flag is used
- Format: `<device> <level> <date>`
- Used by incremental dumps to determine what has changed

---

## restore — Syntax & Options

```bash
restore <mode> [options] -f <dump-file>
```

### Modes (one required)

| Mode | Flag | Description |
|------|------|-------------|
| Interactive | `-i` | Browse dump archive interactively, select files to restore |
| Full restore | `-r` | Restore entire dump to current directory (for full filesystem recovery) |
| Incremental | `-R` | Resume interrupted full restore |
| Extract | `-x` | Extract specified files/directories |
| Table of contents | `-t` | List contents of dump archive |
| Compare | `-C` | Compare dump with live filesystem |

### Options

| Option | Description |
|--------|-------------|
| `-f <file>` | Source dump file, device, or `host:file` |
| `-v` | Verbose output |
| `-y` | Answer yes to all prompts (don't ask for confirmation) |
| `-a` | Skip asking for tape volume number |
| `-b <blocksize>` | Block size in kilobytes |
| `-D <filesystem>` | Filesystem to compare against (used with `-C`) |
| `-e` | Remove unchanged files during incremental restore |
| `-h` | Restore without recursive directory tree (top-level only) |
| `-l` | List tape contents without restoring |
| `-m` | Restore by inode number rather than filename |
| `-N` | Restore in no-op mode (dry run, don't write anything) |
| `-o` | Restore without asking for owner/mode confirmation |
| `-s <tape#>` | Start from specified tape number in a multi-volume set |
| `-T <tmpdir>` | Specify temporary directory (default `/tmp`) |
| `-u` | Allow restoring over existing newer files |
| `-x` | Restore named files (see extract mode) |

---

## restore — Examples

### List contents of a dump archive
```bash
restore -tf /backup/root.dump
```

### List verbose contents
```bash
restore -tvf /backup/root.dump
```

### Interactive restore (browse and select files)
```bash
restore -if /backup/root.dump
```

### Full filesystem restore (run from target filesystem root)
```bash
cd /mnt/target
restore -rf /backup/root.dump
```

### Restore a specific file
```bash
restore -xvf /backup/root.dump ./etc/passwd
```

### Restore a specific directory
```bash
restore -xvf /backup/root.dump ./home/alice
```

### Restore from remote dump
```bash
restore -xvf user@remotehost:/backup/root.dump ./etc/nginx
```

### Compare dump with live filesystem
```bash
restore -Cvf /backup/root.dump -D /
```

### Restore from tape
```bash
restore -rf /dev/st0
```

### Restore incrementals in order (after full restore)
```bash
restore -rf /backup/root-level0.dump
restore -rf /backup/root-level1.dump
restore -rf /backup/root-level2.dump
```

---

## Interactive Mode Commands

When using `restore -if`, the following commands are available at the `restore >` prompt:

| Command | Description |
|---------|-------------|
| `ls [dir]` | List directory contents |
| `cd <dir>` | Change directory in dump archive |
| `pwd` | Print current directory |
| `add <file>` | Mark file or directory for extraction |
| `delete <file>` | Unmark file or directory |
| `extract` | Extract all marked files |
| `help` | Show available commands |
| `quit` | Exit interactive mode |
| `verbose` | Toggle verbose mode |
| `what` | Show dump header information |
| `setmodes` | Set modes of extracted directories |

### Typical interactive session
```
$ restore -if /backup/root.dump
restore > ls
restore > cd etc
restore > ls
restore > add nginx
restore > cd ..
restore > add home/alice
restore > extract
You have not read any volumes yet.
Specify next volume #: 1
restore > quit
```

---

## Full System Recovery Walkthrough

### Step 1 — Boot from live media and partition the disk
```bash
fdisk /dev/sda         # or gdisk for GPT
mkfs.ext4 /dev/sda1   # recreate filesystem
```

### Step 2 — Mount the target filesystem
```bash
mount /dev/sda1 /mnt/target
cd /mnt/target
```

### Step 3 — Restore the level-0 dump
```bash
restore -rf /backup/root-level0.dump
```

### Step 4 — Restore incrementals in order
```bash
restore -rf /backup/root-level1.dump
restore -rf /backup/root-level2.dump
# continue for each incremental level used
```

### Step 5 — Clean up restore state file
```bash
rm -f /mnt/target/restoresymtable
```

### Step 6 — Reinstall bootloader
```bash
grub-install /dev/sda
update-grub
```

### Step 7 — Unmount and reboot
```bash
cd /
umount /mnt/target
reboot
```

---

## Working with Tape Devices

| Command | Description |
|---------|-------------|
| `mt -f /dev/st0 status` | Show tape drive status |
| `mt -f /dev/st0 rewind` | Rewind tape |
| `mt -f /dev/st0 erase` | Erase tape |
| `mt -f /dev/st0 eject` | Eject tape |
| `mt -f /dev/st0 fsf 1` | Forward space one file |
| `mt -f /dev/st0 bsf 1` | Backward space one file |

### Multiple dumps on one tape
```bash
# First dump
dump -0uf /dev/st0 /dev/sda1

# Second dump (tape automatically advances)
dump -0uf /dev/st0 /dev/sda2

# Restore first dump
mt -f /dev/st0 rewind
restore -tf /dev/st0

# Restore second dump
mt -f /dev/st0 fsf 1
restore -tf /dev/st0
```

---

## Remote dump/restore (rdump / rrestore)

Remote operations use `rsh` by default. Override with `RSH` environment variable to use `ssh`.

```bash
# Remote dump via ssh
RSH=/usr/bin/ssh dump -0uf user@remotehost:/backup/root.dump /dev/sda1

# Remote restore via ssh
RSH=/usr/bin/ssh restore -tf user@remotehost:/backup/root.dump
```

The remote host must have `rmt` (remote magnetic tape) installed:
```bash
apt install dump       # installs rmt on Debian/Ubuntu
dnf install dump       # on RHEL/Fedora
```

---

## Automating with cron

### Example backup script
```bash
#!/bin/bash
# /usr/local/bin/backup.sh

BACKUP_DIR="/backup"
DEVICE="/dev/sda1"
DATE=$(date +%Y%m%d)
LEVEL=$1   # pass 0 for full, 1 for incremental

if [ -z "$LEVEL" ]; then
    echo "Usage: $0 <dump-level>"
    exit 1
fi

dump -${LEVEL}ujf "${BACKUP_DIR}/root-level${LEVEL}-${DATE}.dump" "$DEVICE"

if [ $? -eq 0 ]; then
    echo "Dump level ${LEVEL} completed: ${DATE}"
else
    echo "Dump FAILED!" >&2
    exit 1
fi
```

### Crontab entries
```cron
# Full backup every Sunday at 2am
0 2 * * 0 /usr/local/bin/backup.sh 0

# Incremental level-1 Mon-Sat at 2am
0 2 * * 1-6 /usr/local/bin/backup.sh 1
```

---

## Installation

### Debian / Ubuntu
```bash
apt install dump
```

### RHEL / CentOS / Fedora
```bash
dnf install dump
# or
yum install dump
```

### Arch Linux
```bash
pacman -S dump
```

---

## Key Files

| File | Description |
|------|-------------|
| `/etc/dumpdates` | Log of dump dates per device and level |
| `/usr/sbin/dump` | The dump binary |
| `/usr/sbin/restore` | The restore binary |
| `/usr/sbin/rdump` | Remote dump binary |
| `/usr/sbin/rrestore` | Remote restore binary |
| `/usr/sbin/rmt` | Remote magnetic tape server |
| `restoresymtable` | Temporary file created in target dir during restore — delete after recovery |

---

## Common Pitfalls

1. **Always unmount or freeze the filesystem** before dumping to ensure consistency. Use LVM snapshots or `fsfreeze` for live systems:
   ```bash
   fsfreeze -f /mountpoint
   dump -0uf /backup/root.dump /dev/sda1
   fsfreeze -u /mountpoint
   ```

2. **Restore incrementals in strict order** — level 0, then 1, then 2, etc. Skipping a level will leave gaps.

3. **Delete `restoresymtable`** after a complete restore — leaving it wastes space and confuses subsequent operations.

4. **`dump` only works with ext2/ext3/ext4** — for XFS use `xfsdump`/`xfsrestore`; for Btrfs use `btrfs send`/`btrfs receive`.

5. **Use `-u` consistently** — if you skip updating `/etc/dumpdates`, your incremental dumps will be based on the wrong reference time.

6. **Check dump exit codes** in scripts — `0` = success, `1` = some files skipped, `>1` = failure.

---

## See Also

| Tool | Use case |
|------|---------|
| `xfsdump` / `xfsrestore` | XFS filesystem backup |
| `btrfs send` / `btrfs receive` | Btrfs filesystem backup |
| `tar` | Archive files across any filesystem |
| `rsync` | Incremental file synchronisation |
| `dd` | Raw block-level disk imaging |
| `bacula` / `amanda` | Enterprise network backup |

---

*`dump` and `restore` are part of the `dump` package. Always verify backup integrity with `restore -C` before depending on an archive.*
