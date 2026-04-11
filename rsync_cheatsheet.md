# rsync Cheatsheet

## Basic Syntax

```
rsync [OPTIONS] SOURCE DESTINATION
```

- **SOURCE** — file, directory, or remote path to copy from
- **DESTINATION** — file, directory, or remote path to copy to
- Trailing slash on source (`src/`) copies **contents**; no slash copies the **directory itself**

---

## Essential Flags

| Flag | Long Form | Description |
|------|-----------|-------------|
| `-a` | `--archive` | Archive mode: preserves permissions, timestamps, symlinks, owner, group, and recurses (`-rlptgoD`) |
| `-v` | `--verbose` | Verbose output |
| `-z` | `--compress` | Compress data during transfer |
| `-r` | `--recursive` | Recurse into directories |
| `-n` | `--dry-run` | Simulate the transfer without making changes |
| `-P` | | Combines `--progress` and `--partial` (show progress, resume partial transfers) |
| `-h` | `--human-readable` | Output numbers in human-readable format |
| `-u` | `--update` | Skip files that are newer on the destination |
| `-e` | `--rsh=COMMAND` | Specify remote shell (e.g. SSH) |
| `--delete` | | Delete files in destination that don't exist in source |
| `--exclude` | | Exclude files/directories matching a pattern |
| `--include` | | Include files/directories matching a pattern |
| `--checksum` | `-c` | Compare files by checksum instead of mod-time and size |
| `--no-perms` | | Do not preserve permissions |
| `--no-owner` | | Do not preserve owner |
| `--no-group` | | Do not preserve group |
| `--times` | `-t` | Preserve modification times |
| `--links` | `-l` | Preserve symlinks |
| `--hard-links` | `-H` | Preserve hard links |
| `-q` | `--quiet` | Suppress non-error output |
| `--stats` | | Show transfer statistics summary |
| `--bwlimit=KB/s` | | Limit I/O bandwidth (e.g. `--bwlimit=1000`) |
| `--timeout=SEC` | | Set I/O timeout in seconds |

---

## Common Use Cases

### Local Copy

```bash
# Copy a file
rsync file.txt /backup/file.txt

# Copy a directory (and its contents)
rsync -av /source/dir/ /destination/dir/

# Copy directory itself (not just contents)
rsync -av /source/dir /destination/
```

### Remote Copy (over SSH)

```bash
# Local → Remote
rsync -avz /local/path/ user@host:/remote/path/

# Remote → Local
rsync -avz user@host:/remote/path/ /local/path/

# Specify SSH port
rsync -avz -e "ssh -p 2222" /local/path/ user@host:/remote/path/

# Use SSH key
rsync -avz -e "ssh -i ~/.ssh/mykey" /local/path/ user@host:/remote/path/
```

### Dry Run (Preview Changes)

```bash
rsync -avzn /source/ /destination/

# Recommended before any destructive sync
rsync -avzn --delete /source/ /destination/
```

### Mirror / Full Sync (with Deletion)

```bash
# Make destination exactly match source (deletes extra files in dest)
rsync -av --delete /source/ /destination/
```

### Show Progress

```bash
rsync -av --progress /source/ /destination/

# Shorthand with partial resume support
rsync -avP /source/ /destination/
```

### Backup with Timestamp

```bash
rsync -av --backup --backup-dir=/backups/$(date +%Y-%m-%d) /source/ /destination/
```

### Limit Bandwidth

```bash
# Limit to ~500 KB/s
rsync -avz --bwlimit=500 /source/ user@host:/destination/
```

---

## Exclude & Include Patterns

### Exclude Files or Directories

```bash
# Exclude a single directory
rsync -av --exclude='node_modules' /source/ /destination/

# Exclude multiple patterns
rsync -av --exclude='*.log' --exclude='*.tmp' /source/ /destination/

# Exclude using a file
rsync -av --exclude-from='exclude.txt' /source/ /destination/
```

**exclude.txt example:**
```
node_modules/
.git/
*.log
*.tmp
.DS_Store
__pycache__/
```

### Include / Exclude Combination

```bash
# Only sync .jpg files (include pattern, exclude everything else)
rsync -av --include='*.jpg' --exclude='*' /source/ /destination/

# Sync only a specific subdirectory
rsync -av --include='subdir/***' --exclude='*' /source/ /destination/
```

> **Order matters:** `--include` and `--exclude` rules are evaluated in the order they appear.

---

## Trailing Slash Rules

| Command | Behaviour |
|---------|-----------|
| `rsync -av src/ dest/` | Copies **contents** of `src` into `dest` |
| `rsync -av src dest/` | Copies **`src` directory** into `dest`, creating `dest/src/` |

---

## Rsync Daemon Mode

```bash
# Sync to a named rsync module
rsync -av rsync://user@host::module/path /local/path/

# Or using the rsync:// URL scheme
rsync -avz rsync://backup.example.com/data /local/backup/
```

---

## Useful Combinations

```bash
# Safe full backup command (dry-run first)
rsync -avzP --delete --dry-run /source/ user@host:/dest/

# Run for real once happy
rsync -avzP --delete /source/ user@host:/dest/

# Sync only newer files (no deletions)
rsync -avzu /source/ /destination/

# Preserve hard links and ACLs
rsync -avHA /source/ /destination/

# Exclude hidden files/dirs
rsync -av --exclude='.*' /source/ /destination/

# Resume a large interrupted transfer
rsync -avP /source/largefile.tar /destination/
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Syntax or usage error |
| `2` | Protocol incompatibility |
| `3` | Errors selecting input/output files |
| `11` | Error in file I/O |
| `12` | Error in rsync protocol data stream |
| `23` | Partial transfer due to error |
| `24` | Partial transfer — some source files vanished |
| `25` | Max delete limit reached |
| `30` | Timeout in data send/receive |

---

## Quick Reference Card

```bash
# The "safe backup" one-liner
rsync -avzP --delete -n /src/ user@host:/dest/   # dry-run
rsync -avzP --delete    /src/ user@host:/dest/   # real run

# Key flag combos to remember
-avz        → archive + verbose + compress       (remote transfers)
-avzP       → above + progress + partial resume
-avzPn      → above but dry-run only
--delete    → mirror mode (removes extra dest files)
--exclude   → skip files matching pattern
--bwlimit   → throttle bandwidth
```

---

*rsync was created by Andrew Tridgell and Paul Mackerras. See `man rsync` or https://rsync.samba.org for full documentation.*
