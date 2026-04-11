# Rclone Cheatsheet

> A comprehensive reference for `rclone` — the command-line program to manage files on cloud storage, supporting 70+ providers including S3, Google Drive, Dropbox, OneDrive, SFTP, and more.

---

## Table of Contents

- [Basic Syntax](#basic-syntax)
- [Installation & Setup](#installation--setup)
- [Remotes & Configuration](#remotes--configuration)
- [Core Commands](#core-commands)
- [copy vs sync vs move](#copy-vs-sync-vs-move)
- [Filters & Exclusions](#filters--exclusions)
- [Flags & Performance Tuning](#flags--performance-tuning)
- [Listing & Querying](#listing--querying)
- [Mounting](#mounting)
- [Serving](#serving)
- [Encryption (crypt remote)](#encryption-crypt-remote)
- [Bisync](#bisync)
- [Backends & Providers](#backends--providers)
- [Practical Examples](#practical-examples)
- [Automation & Scripting](#automation--scripting)
- [Quick Reference Card](#quick-reference-card)

---

## Basic Syntax

```
rclone command source destination [flags]
```

Paths follow the format:

```
remote:path/to/folder        # cloud remote
/local/path                  # local filesystem
:backend:path                # on-the-fly remote (no config needed)
```

```bash
rclone copy /local/dir remote:bucket/dir
rclone sync remote:bucket /local/backup
rclone ls gdrive:Documents
```

---

## Installation & Setup

```bash
# Linux — install script (official)
curl https://rclone.org/install.sh | sudo bash

# Debian / Ubuntu
sudo apt install rclone

# macOS (Homebrew)
brew install rclone

# Update to latest version
sudo rclone selfupdate

# Check installed version
rclone version

# Check for newer version
rclone version --check
```

---

## Remotes & Configuration

### Interactive Setup

```bash
# Launch interactive config wizard
rclone config

# Options presented:
#   n) New remote
#   e) Edit existing remote
#   d) Delete a remote
#   r) Rename a remote
#   c) Copy a remote
#   q) Quit config
```

### Managing Remotes

```bash
# List all configured remotes
rclone listremotes

# Show full config for all remotes
rclone config show

# Show config for a specific remote
rclone config show myremote:

# Show config file path
rclone config file

# Reconnect / refresh OAuth token for a remote
rclone config reconnect myremote:

# Delete a remote
rclone config delete myremote

# Create a remote non-interactively (example: S3)
rclone config create myS3 s3 \
  provider=AWS \
  access_key_id=AKIAIOSFODNN7 \
  secret_access_key=wJalrXUtnFEMI \
  region=us-east-1
```

### Config File

```bash
# Default config location
~/.config/rclone/rclone.conf   # Linux/macOS
%APPDATA%\rclone\rclone.conf   # Windows

# Use a custom config file
rclone --config /path/to/custom.conf ls remote:

# Set via environment variable
export RCLONE_CONFIG=/path/to/rclone.conf
```

### On-the-Fly Remotes (no config needed)

```bash
# S3 without config
rclone ls :s3:mybucket --s3-access-key-id=KEY --s3-secret-access-key=SECRET

# SFTP without config
rclone ls :sftp:path --sftp-host=server --sftp-user=user --sftp-pass=pass

# HTTP (read-only)
rclone ls :http: --http-url https://example.com/files/
```

---

## Core Commands

| Command | Description |
|---|---|
| `copy` | Copy files from source to dest; skip identical files |
| `sync` | Make dest identical to source; **deletes** from dest |
| `move` | Move files from source to dest |
| `bisync` | Two-way sync between source and dest |
| `ls` | List objects and sizes |
| `lsl` | List with modification time and path |
| `lsd` | List directories only |
| `lsf` | Formatted listing (machine-parsable) |
| `lsjson` | List as JSON |
| `tree` | Directory tree |
| `mkdir` | Create directory/bucket |
| `rmdir` | Remove empty directory/bucket |
| `rmdirs` | Remove empty directories recursively |
| `delete` | Delete files matching filters |
| `purge` | Delete directory and all contents |
| `cat` | Output file contents to stdout |
| `copyto` | Copy single file, rename at destination |
| `moveto` | Move single file, rename at destination |
| `copyurl` | Download a URL to a remote |
| `check` | Compare source and dest for differences |
| `checksum` | Check file checksums |
| `dedupe` | Find and remove duplicate files |
| `about` | Show quota and usage info |
| `size` | Calculate size of a path |
| `md5sum` | Generate MD5 checksums |
| `sha1sum` | Generate SHA1 checksums |
| `touch` | Update file modification time |
| `serve` | Serve remote over HTTP, WebDAV, FTP, SFTP, S3, DLNA |
| `mount` | Mount remote as filesystem |
| `link` | Create a public sharing link |
| `cleanup` | Clean up trash/incomplete uploads |

---

## copy vs sync vs move

| | `copy` | `sync` | `move` |
|---|---|---|---|
| Copies new/changed files | ✅ | ✅ | ✅ |
| Deletes from destination | ❌ | ✅ | ❌ |
| Deletes from source | ❌ | ❌ | ✅ |
| Safe first run | ✅ | ⚠️ use `--dry-run` | ⚠️ use `--dry-run` |

```bash
# copy — add/update files at dest; never delete
rclone copy source: dest:

# sync — make dest mirror source exactly (DELETES from dest)
rclone sync source: dest:

# move — copy then delete from source
rclone move source: dest:

# ALWAYS test destructive operations first
rclone sync source: dest: --dry-run
rclone sync source: dest: --dry-run -v
```

---

## Filters & Exclusions

### Include / Exclude by Name

```bash
# Exclude a directory
rclone copy source: dest: --exclude ".git/**"

# Exclude file types
rclone copy source: dest: --exclude "*.tmp" --exclude "*.log"

# Include only specific types (exclude everything else)
rclone copy source: dest: --include "*.jpg" --include "*.png"

# Exclude hidden files (starting with dot)
rclone copy source: dest: --exclude ".*"

# Exclude hidden directories
rclone copy source: dest: --exclude ".*/**"
```

### Filter Rules File

```bash
# Use a filter file (one rule per line)
rclone copy source: dest: --filter-from filters.txt
```

Example `filters.txt`:
```
- .git/**
- *.tmp
- *.log
- node_modules/**
+ *.py
+ *.md
- **
```

Rules are evaluated top to bottom. `+` = include, `-` = exclude. A final `- **` excludes everything not previously matched.

### Size & Age Filters

```bash
# Only files smaller than 10MB
rclone copy source: dest: --max-size 10M

# Only files larger than 1GB
rclone copy source: dest: --min-size 1G

# Only files modified in the last 7 days
rclone copy source: dest: --max-age 7d

# Only files older than 30 days
rclone copy source: dest: --min-age 30d

# Ignore files modified within the last 2 hours (still being written)
rclone copy source: dest: --min-age 2h

# Size units: b, k, M, G, T, P
# Age units:  ms, s, m, h, d, w, M, y
```

### Depth & Path Filters

```bash
# Limit traversal depth
rclone copy source: dest: --max-depth 2

# Match on full path (anchored with /)
rclone copy source: dest: --exclude "/secret/**"

# Case-insensitive matching
rclone copy source: dest: --exclude-if-present .nobackup
```

---

## Flags & Performance Tuning

### Dry Run & Logging

```bash
# Show what would happen — no changes made
rclone sync source: dest: --dry-run

# Verbose output (level 1)
rclone copy source: dest: -v

# Very verbose (level 2 — shows every file)
rclone copy source: dest: -vv

# Log to file
rclone copy source: dest: --log-file /var/log/rclone.log

# Log level: DEBUG, INFO, NOTICE, ERROR
rclone copy source: dest: --log-level DEBUG

# Show progress stats
rclone copy source: dest: --progress
```

### Transfer Performance

```bash
# Number of parallel file transfers (default: 4)
rclone copy source: dest: --transfers 16

# Number of parallel directory listing threads (default: 10)
rclone copy source: dest: --checkers 32

# Buffer size for transfers (default: 16M)
rclone copy source: dest: --buffer-size 64M

# Bandwidth limit (bytes/sec; M=megabytes, k=kilobytes)
rclone copy source: dest: --bwlimit 10M

# Bandwidth schedule (off-peak only)
rclone copy source: dest: --bwlimit "08:00,512k 18:00,off 23:00,10M"

# Multi-part upload threshold (S3-like backends)
rclone copy source: dest: --s3-upload-cutoff 64M

# Chunk size for multi-part uploads
rclone copy source: dest: --s3-chunk-size 32M

# Retries on failure (default: 3)
rclone copy source: dest: --retries 5

# Low-level retries per file chunk (default: 10)
rclone copy source: dest: --low-level-retries 10

# Timeout for inactivity
rclone copy source: dest: --timeout 30s

# Connect timeout
rclone copy source: dest: --contimeout 15s
```

### Comparison & Integrity

```bash
# Compare by checksum instead of size+time (slower but accurate)
rclone sync source: dest: --checksum

# Ignore modification time — only use size
rclone sync source: dest: --ignore-times

# Skip files that already exist at dest (any size)
rclone copy source: dest: --ignore-existing

# Update only — skip if dest is newer
rclone copy source: dest: --update

# No-traverse — useful when dest has many files
rclone sync source: dest: --no-traverse
```

### Metadata & Attributes

```bash
# Preserve file permissions and metadata (where supported)
rclone copy source: dest: --metadata

# Copy file modification times where possible
rclone copy source: dest: --copy-links    # follow symlinks

# Preserve symlinks instead of following them
rclone copy source: dest: --links
```

---

## Listing & Querying

```bash
# List files with size
rclone ls remote:path

# List with size, modification time, and path
rclone lsl remote:path

# List directories only
rclone lsd remote:

# Formatted listing — one file per line, no header
rclone lsf remote:path

# Recursive listing
rclone lsf remote:path --recursive

# List as JSON (for scripting)
rclone lsjson remote:path

# Tree view
rclone tree remote:path

# Show total size of a path
rclone size remote:path

# Show quota and usage
rclone about remote:

# Calculate checksums
rclone md5sum remote:path
rclone sha1sum remote:path

# Check differences between source and dest
rclone check source: dest:
rclone check source: dest: --one-way   # only check source files exist in dest
```

---

## Mounting

Mount a remote as a local filesystem using FUSE (Linux/macOS) or WinFSP (Windows).

```bash
# Basic mount (foreground — blocks terminal)
rclone mount remote:path /mnt/myremote

# Mount in background (daemon mode)
rclone mount remote:path /mnt/myremote --daemon

# Unmount
fusermount -u /mnt/myremote          # Linux
umount /mnt/myremote                  # macOS

# Mount with read-only access
rclone mount remote:path /mnt/myremote --read-only

# Mount with caching (recommended for most use)
rclone mount remote:path /mnt/myremote \
  --vfs-cache-mode full \
  --vfs-cache-max-size 10G \
  --vfs-cache-max-age 24h

# Allow other users to access the mount
rclone mount remote:path /mnt/myremote --allow-other

# Mount at startup via /etc/fstab (systemd)
# See: rclone mount --daemon-timeout
```

### VFS Cache Modes

| Mode | Description |
|---|---|
| `off` | No caching — reads/writes go directly to remote |
| `minimal` | Cache only files opened for read (default) |
| `writes` | Cache writes; flush on close |
| `full` | Cache everything; best compatibility and performance |

```bash
rclone mount remote: /mnt/point --vfs-cache-mode full
```

### systemd Service Example

```ini
# /etc/systemd/system/rclone-mount.service
[Unit]
Description=RClone mount for myremote
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/rclone mount myremote: /mnt/myremote \
  --vfs-cache-mode full \
  --vfs-cache-max-size 5G \
  --log-file /var/log/rclone-mount.log
ExecStop=/bin/fusermount -u /mnt/myremote
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable rclone-mount
sudo systemctl start rclone-mount
```

---

## Serving

Rclone can serve a remote over several protocols, making it accessible to other tools.

```bash
# Serve over HTTP
rclone serve http remote:path --addr :8080

# Serve over WebDAV (mount in Finder, Windows Explorer, etc.)
rclone serve webdav remote:path --addr :8080

# Serve over FTP
rclone serve ftp remote:path --addr :2121

# Serve over SFTP
rclone serve sftp remote:path --addr :2022

# Serve as an S3-compatible endpoint
rclone serve s3 remote:path --addr :9000

# Serve DLNA (media server)
rclone serve dlna remote:path

# Add basic auth
rclone serve http remote:path --addr :8080 \
  --user myuser --pass mypassword

# Serve read-only
rclone serve webdav remote:path --read-only
```

---

## Encryption (crypt remote)

Rclone's `crypt` remote transparently encrypts files before uploading to any other remote.

```bash
# Create a crypt remote wrapping an existing remote
rclone config
# n) New remote
# name: mycrypt
# type: crypt
# remote: myremote:encrypted-folder
# filename_encryption: standard
# directory_name_encryption: true
# password: (enter password)
```

```bash
# Use the crypt remote just like any other
rclone copy /local/data mycrypt:
rclone ls mycrypt:
rclone mount mycrypt: /mnt/decrypted

# The underlying remote stores encrypted filenames + content
rclone ls myremote:encrypted-folder    # shows scrambled names

# Encrypt on the fly — no config needed (advanced)
rclone copy source: :crypt,remote=dest:,password=ENCRYPTED_PASS:
```

> **Key points:**
> - Passwords are stored in the config file — protect it with `rclone config encryption`
> - File contents, names, and directory names can all be encrypted
> - Standard encryption uses NaCl secretbox (XSalsa20 + Poly1305)

---

## Bisync

Two-way synchronisation between two remotes (or local paths).

```bash
# First run — must use --resync to initialise
rclone bisync source: dest: --resync

# Regular bisync after initialisation
rclone bisync source: dest:

# Verbose bisync with dry-run
rclone bisync source: dest: --dry-run -v

# Conflict resolution — newer file wins
rclone bisync source: dest: --conflict-resolve newer

# Force resync (resets state — use with care)
rclone bisync source: dest: --resync --force
```

> **Note:** Bisync is still considered experimental. Always keep backups and test with `--dry-run` first.

---

## Backends & Providers

Rclone supports 70+ storage providers. Common ones:

| Provider | Type Name | Notes |
|---|---|---|
| Amazon S3 | `s3` | Also works with MinIO, Wasabi, Backblaze B2 S3, etc. |
| Google Drive | `drive` | Requires OAuth setup |
| Dropbox | `dropbox` | Requires OAuth setup |
| Microsoft OneDrive | `onedrive` | Requires OAuth setup |
| Backblaze B2 | `b2` | Native API or S3-compatible |
| SFTP | `sftp` | Any SSH server |
| FTP | `ftp` | Standard FTP/FTPS |
| WebDAV | `webdav` | Nextcloud, Owncloud, etc. |
| Azure Blob Storage | `azureblob` | |
| Google Cloud Storage | `gcs` | |
| Mega | `mega` | |
| pCloud | `pcloud` | |
| Box | `box` | Requires OAuth setup |
| Jottacloud | `jottacloud` | |
| Local filesystem | `local` | Used implicitly for local paths |

```bash
# List all supported backends
rclone help backends

# Get help for a specific backend
rclone help backend s3
```

---

## Practical Examples

### Backup & Restore

```bash
# Backup local folder to S3
rclone sync /home/user/docs s3:my-backups/docs --progress

# Backup to multiple destinations
rclone copy /data s3:backups/data &
rclone copy /data gdrive:backups/data &
wait

# Restore from remote
rclone copy s3:my-backups/docs /home/user/docs-restored

# Incremental backup (copy only changed files)
rclone copy /data remote:backups --track-renames

# Backup with date-stamped folder
rclone copy /data remote:backups/$(date +%Y-%m-%d)

# Backup excluding large media files
rclone sync /data remote:backups \
  --exclude "*.iso" --exclude "*.mkv" --exclude "*.mp4"
```

### Cloud-to-Cloud Transfer

```bash
# Copy between two cloud providers (server-side where possible)
rclone copy gdrive:Photos s3:my-bucket/photos --progress

# Sync from Dropbox to OneDrive
rclone sync dropbox: onedrive:dropbox-mirror --progress

# Move files from one S3 bucket to another
rclone move s3:old-bucket s3:new-bucket --progress

# Copy with bandwidth limit (useful for metered connections)
rclone copy gdrive: s3:backup --bwlimit 5M
```

### Deduplication

```bash
# Find and list duplicate files
rclone dedupe --dedupe-mode list remote:path

# Interactively remove duplicates
rclone dedupe remote:path

# Auto-rename duplicates
rclone dedupe --dedupe-mode rename remote:path

# Keep the first file, delete all duplicates
rclone dedupe --dedupe-mode first remote:path
```

### Integrity Checking

```bash
# Verify a backup matches the source
rclone check /local/data remote:backup

# Check with MD5 hashes
rclone check /local/data remote:backup --checksum

# One-way check (only verify source files exist in dest)
rclone check /local/data remote:backup --one-way

# Download and verify checksums
rclone md5sum remote:path > remote_checksums.txt
md5sum -c remote_checksums.txt
```

### S3 / Object Storage

```bash
# List all buckets
rclone lsd s3:

# Create a bucket
rclone mkdir s3:my-new-bucket

# Set storage class on upload
rclone copy /data s3:bucket --s3-storage-class STANDARD_IA

# Make a file publicly accessible
rclone copy file.txt s3:bucket/ --s3-acl public-read

# Sync with server-side encryption
rclone sync /data s3:bucket --s3-sse aws:kms

# Empty and delete a bucket
rclone purge s3:old-bucket

# Generate a presigned URL (via AWS CLI, rclone link for supported backends)
rclone link s3:bucket/file.txt
```

---

## Automation & Scripting

### Cron Jobs

```bash
# Edit crontab
crontab -e

# Daily backup at 2am
0 2 * * * rclone sync /home/user/data remote:backup/data \
  --log-file /var/log/rclone-backup.log \
  --log-level INFO

# Hourly sync with bandwidth limit during business hours
0 * * * * rclone copy /data remote:sync \
  --bwlimit "09:00,2M 17:00,off" \
  --log-file /var/log/rclone.log
```

### Exit Codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Syntax or usage error |
| `2` | Error not otherwise categorised |
| `3` | Directory not found |
| `4` | File not found |
| `5` | Temporary error — retry may succeed |
| `6` | Less serious error |
| `7` | Fatal error |
| `8` | Transfer limit exceeded |
| `9` | Operation successful but no files transferred |

```bash
#!/bin/bash
rclone sync /data remote:backup --log-file /tmp/rclone.log

case $? in
  0) echo "Sync completed successfully" ;;
  5) echo "Temporary error — will retry" ;;
  *) echo "Error occurred — check log"; exit 1 ;;
esac
```

### Environment Variables

```bash
# Any rclone flag can be set as an env var: RCLONE_FLAG_NAME
export RCLONE_CONFIG=/custom/path/rclone.conf
export RCLONE_LOG_LEVEL=INFO
export RCLONE_TRANSFERS=16
export RCLONE_BWLIMIT=10M

# Backend-specific env vars
export RCLONE_S3_ACCESS_KEY_ID=KEY
export RCLONE_S3_SECRET_ACCESS_KEY=SECRET
export RCLONE_S3_REGION=eu-west-1
```

### Scripting with lsjson

```bash
# List files as JSON and parse with jq
rclone lsjson remote:path | jq '.[] | select(.Size > 1000000) | .Path'

# Find files modified in last 24h
rclone lsjson remote:path --recursive \
  | jq --arg cutoff "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S)" \
       '.[] | select(.ModTime > $cutoff) | .Path'

# Count files
rclone lsjson remote:path --recursive | jq length

# Total size of all files
rclone lsjson remote:path --recursive | jq '[.[].Size] | add'
```

---

## Quick Reference Card

```
rclone command source: dest: [flags]

CORE COMMANDS                   LISTING
  copy    copy (no delete)        ls      size + path
  sync    mirror (deletes!)       lsl     + modification time
  move    copy + delete source    lsd     directories only
  bisync  two-way sync            lsf     parseable format
  check   compare src and dest    lsjson  JSON output
  dedupe  find & remove dupes     tree    directory tree
  purge   delete dir + contents   size    total size of path
  delete  delete matching files   about   quota info

FILTERS                         PERFORMANCE
  --include "*.jpg"               --transfers N     parallel transfers
  --exclude "*.tmp"               --checkers N      parallel checkers
  --filter-from file.txt          --bwlimit 10M     bandwidth cap
  --max-size 100M                 --buffer-size 64M transfer buffer
  --min-age 7d                    --retries N       retry attempts
  --max-depth 3                   --checksum        verify by hash

DRY RUN & LOGGING               MOUNT
  --dry-run                       rclone mount remote: /mnt/point
  --progress                      --vfs-cache-mode full
  -v / -vv                        --daemon
  --log-file /path/to.log         fusermount -u /mnt/point

SERVE                           ENCRYPTION
  rclone serve http remote: :8080   crypt remote wraps any remote
  rclone serve webdav remote: :8080 encrypts contents + filenames
  rclone serve ftp remote: :2121    rclone config → type: crypt
  rclone serve sftp remote: :2022
  rclone serve s3 remote: :9000
```

---

> **Further Reading:**
> - `rclone help` and `rclone help <command>`
> - [rclone.org documentation](https://rclone.org/docs/)
> - [rclone supported backends](https://rclone.org/overview/)
> - [rclone flags reference](https://rclone.org/flags/)
> - `rclone config` — interactive setup for any provider
