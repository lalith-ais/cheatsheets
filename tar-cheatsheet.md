# tar Cheatsheet

A complete reference for the `tar` (tape archive) utility — the standard tool for creating, extracting, and managing archives on Linux and Unix systems.

---

## Syntax

```bash
tar [operation] [options] [archive] [file(s)]
```

> **Tip:** The leading `-` before options is optional in most implementations. `tar -czf` and `tar czf` are equivalent.

---

## Operations (one required)

| Flag | Long form | Description |
|------|-----------|-------------|
| `-c` | `--create` | Create a new archive |
| `-x` | `--extract` | Extract files from an archive |
| `-t` | `--list` | List contents of an archive |
| `-r` | `--append` | Append files to end of an existing archive |
| `-u` | `--update` | Append files newer than those in archive |
| `-d` | `--diff` / `--compare` | Compare archive with filesystem |
| `-A` | `--catenate` | Append one archive to another |
| `--delete` | | Delete files from archive (not for tape devices) |

---

## Most-Used Options

| Flag | Long form | Description |
|------|-----------|-------------|
| `-f <file>` | `--file=<file>` | Archive filename (required; use `-` for stdin/stdout) |
| `-v` | `--verbose` | Verbose — list files as processed |
| `-z` | `--gzip` | Compress/decompress with gzip (`.tar.gz`, `.tgz`) |
| `-j` | `--bzip2` | Compress/decompress with bzip2 (`.tar.bz2`, `.tbz2`) |
| `-J` | `--xz` | Compress/decompress with xz (`.tar.xz`, `.txz`) |
| `--zstd` | | Compress/decompress with zstd (`.tar.zst`) |
| `-Z` | `--compress` | Compress/decompress with compress (`.tar.Z`) |
| `-a` | `--auto-compress` | Infer compression from archive filename extension |
| `-C <dir>` | `--directory=<dir>` | Change to directory before operation |
| `-p` | `--preserve-permissions` | Preserve file permissions (default for root) |
| `--numeric-owner` | | Use numeric UID/GID instead of names |
| `-P` | `--absolute-names` | Don't strip leading `/` from filenames |
| `--strip-components=N` | | Strip N leading path components on extract |
| `-k` | `--keep-old-files` | Don't overwrite existing files on extract |
| `--overwrite` | | Overwrite existing files on extract |
| `-m` | `--touch` | Don't restore modification times |
| `-h` | `--dereference` | Follow symlinks — archive the files they point to |
| `--hard-dereference` | | Dereference hard links |
| `-X <file>` | `--exclude-from=<file>` | Exclude files matching patterns in file |
| `--exclude=<pattern>` | | Exclude files matching pattern |
| `--exclude-vcs` | | Exclude version control directories (`.git`, `.svn`, etc.) |
| `--exclude-vcs-ignores` | | Read exclusions from VCS ignore files (`.gitignore`) |
| `-n` | `--seek` | Archive is seekable (enables random access) |
| `--sparse` | `-S` | Handle sparse files efficiently |
| `--acls` | | Preserve ACLs |
| `--xattrs` | | Preserve extended attributes |
| `--selinux` | | Preserve SELinux context |
| `-I <prog>` | `--use-compress-program=<prog>` | Use custom compression program |
| `--checkpoint[=N]` | | Show progress checkpoint every N records |
| `--checkpoint-action=<act>` | | Action at each checkpoint (e.g., `dot`, `echo`) |
| `-b <N>` | `--blocking-factor=N` | Set block size to N × 512 bytes |
| `--record-size=N` | | Set record size in bytes |

---

## Compression Comparison

| Format | Flag | Extension | Speed | Ratio | Best for |
|--------|------|-----------|-------|-------|---------|
| gzip | `-z` | `.tar.gz` / `.tgz` | Fast | Good | General use |
| bzip2 | `-j` | `.tar.bz2` / `.tbz2` | Moderate | Better | Slightly smaller archives |
| xz | `-J` | `.tar.xz` / `.txz` | Slow | Best | Maximum compression |
| zstd | `--zstd` | `.tar.zst` | Very fast | Good | Modern, balanced |
| lz4 | `-I lz4` | `.tar.lz4` | Fastest | Low | Speed-critical transfers |
| compress | `-Z` | `.tar.Z` | Fast | Low | Legacy systems |

---

## Creating Archives

### Basic archive (no compression)
```bash
tar -cf archive.tar file1 file2 dir/
```

### With gzip compression
```bash
tar -czf archive.tar.gz dir/
```

### With bzip2 compression
```bash
tar -cjf archive.tar.bz2 dir/
```

### With xz compression
```bash
tar -cJf archive.tar.xz dir/
```

### With zstd compression
```bash
tar --zstd -cf archive.tar.zst dir/
```

### Auto-detect compression from filename
```bash
tar -caf archive.tar.xz dir/
```

### Verbose (show files as added)
```bash
tar -czvf archive.tar.gz dir/
```

### Archive multiple sources
```bash
tar -czf archive.tar.gz /etc /home /var/log
```

### Archive from a specific directory
```bash
tar -czf archive.tar.gz -C /var/www html/
```

### Archive current directory contents
```bash
tar -czf archive.tar.gz -C /path/to/dir .
```

### Preserve permissions and ownership
```bash
tar -czpf archive.tar.gz dir/
```

### Archive with ACLs and extended attributes
```bash
tar --acls --xattrs -czf archive.tar.gz dir/
```

### Follow symlinks (archive targets, not links)
```bash
tar -czhf archive.tar.gz dir/
```

### Handle sparse files efficiently
```bash
tar -cSzf archive.tar.gz sparse-file
```

---

## Listing Archive Contents

### List contents
```bash
tar -tf archive.tar.gz
```

### Verbose listing (permissions, owner, size, date)
```bash
tar -tvf archive.tar.gz
```

### List contents of a remote archive via ssh
```bash
ssh user@host "cat /backup/archive.tar.gz" | tar -tzvf -
```

### Search for a specific file in archive
```bash
tar -tf archive.tar.gz | grep filename
```

---

## Extracting Archives

### Extract to current directory
```bash
tar -xf archive.tar.gz
```

### Extract with verbose output
```bash
tar -xvf archive.tar.gz
```

### Extract to a specific directory
```bash
tar -xf archive.tar.gz -C /target/dir
```

### Extract a specific file
```bash
tar -xf archive.tar.gz path/in/archive/file.txt
```

### Extract a specific directory from archive
```bash
tar -xf archive.tar.gz path/in/archive/subdir/
```

### Extract using wildcards
```bash
tar -xf archive.tar.gz --wildcards '*.conf'
```

### Extract without top-level directory (strip 1 component)
```bash
tar -xf archive.tar.gz --strip-components=1
```

### Extract without overwriting existing files
```bash
tar -xkf archive.tar.gz
```

### Extract and preserve permissions
```bash
tar -xpf archive.tar.gz
```

### Extract preserving ACLs and xattrs
```bash
tar --acls --xattrs -xf archive.tar.gz
```

### Extract to stdout (pipe to another command)
```bash
tar -xOf archive.tar.gz path/to/file | less
```

---

## Excluding Files & Directories

### Exclude a single path
```bash
tar -czf archive.tar.gz dir/ --exclude=dir/cache
```

### Exclude by pattern
```bash
tar -czf archive.tar.gz dir/ --exclude='*.log' --exclude='*.tmp'
```

### Exclude from a file
```bash
cat exclude.txt
*.log
*.tmp
cache/
node_modules/

tar -czf archive.tar.gz dir/ -X exclude.txt
```

### Exclude VCS directories
```bash
tar -czf archive.tar.gz dir/ --exclude-vcs
```

### Exclude VCS-ignored files (honours .gitignore)
```bash
tar -czf archive.tar.gz dir/ --exclude-vcs-ignores
```

### Exclude node_modules and .git
```bash
tar -czf archive.tar.gz project/ \
    --exclude=project/node_modules \
    --exclude=project/.git
```

---

## Appending & Updating

### Append files to an existing uncompressed archive
```bash
tar -rf archive.tar newfile.txt
```

### Append only files newer than those in archive
```bash
tar -uf archive.tar dir/
```

### Concatenate two archives
```bash
tar -Af combined.tar archive1.tar archive2.tar
```

> **Note:** Appending and updating only work on uncompressed (`.tar`) archives, not compressed ones.

---

## Comparing Archives

### Compare archive against filesystem
```bash
tar -df archive.tar.gz
```

### Compare with verbose output
```bash
tar -dvf archive.tar.gz
```

---

## Deleting from Archives

### Delete a file from an uncompressed archive
```bash
tar --delete -f archive.tar path/to/file.txt
```

### Delete a directory from an archive
```bash
tar --delete -f archive.tar path/to/dir/
```

> **Note:** `--delete` does not work on compressed archives or tape devices.

---

## Working with stdin / stdout (Pipelines)

### Create archive and write to stdout
```bash
tar -czf - dir/ > archive.tar.gz
```

### Pipe archive through ssh to remote host
```bash
tar -czf - dir/ | ssh user@host "cat > /backup/archive.tar.gz"
```

### Extract archive received from stdin
```bash
cat archive.tar.gz | tar -xzf -
```

### Copy a directory tree to another location
```bash
tar -cf - source/ | tar -xf - -C /destination/
```

### Copy to remote host preserving permissions
```bash
tar -cpf - /source/ | ssh user@host "tar -xpf - -C /destination/"
```

### Create compressed archive over ssh from remote files
```bash
ssh user@host "tar -czf - /remote/dir" > local-backup.tar.gz
```

---

## Multi-Volume Archives (Tape)

### Create multi-volume archive (prompt for next tape)
```bash
tar -cMf /dev/st0 dir/
```

### Create multi-volume archive split to file size
```bash
tar -cMf 'archive-vol-%d.tar' --tape-length=700M dir/
```

### Extract from multi-volume archive
```bash
tar -xMf /dev/st0
```

---

## Progress & Checkpoints

### Show a dot for every 10 blocks processed
```bash
tar -czf archive.tar.gz dir/ \
    --checkpoint=1 \
    --checkpoint-action=dot
```

### Show progress with `pv` (pipe viewer)
```bash
tar -czf - dir/ | pv > archive.tar.gz
```

### Show progress with `pv` when extracting
```bash
pv archive.tar.gz | tar -xzf -
```

---

## Custom Compression Programs

### Use pigz (parallel gzip) for faster compression
```bash
tar -cf - dir/ | pigz > archive.tar.gz
# or
tar -I pigz -cf archive.tar.gz dir/
```

### Use lbzip2 (parallel bzip2)
```bash
tar -I lbzip2 -cf archive.tar.bz2 dir/
```

### Use pixz (parallel xz)
```bash
tar -I pixz -cf archive.tar.xz dir/
```

### Use lz4
```bash
tar -I lz4 -cf archive.tar.lz4 dir/
```

### Use zstd with compression level
```bash
tar -I 'zstd -19' -cf archive.tar.zst dir/
```

---

## Incremental Backups

tar supports incremental backups via snapshot files.

### Create initial (level-0) backup
```bash
tar -czf backup-full.tar.gz \
    --listed-incremental=/var/backup/snapshot.snar \
    /home/
```

### Create incremental backup (run again — only changed files)
```bash
tar -czf backup-inc-$(date +%Y%m%d).tar.gz \
    --listed-incremental=/var/backup/snapshot.snar \
    /home/
```

### Restore full backup
```bash
tar -xzf backup-full.tar.gz \
    --listed-incremental=/dev/null \
    -C /restore/
```

### Restore incremental on top
```bash
tar -xzf backup-inc-20250407.tar.gz \
    --listed-incremental=/dev/null \
    -C /restore/
```

> The `--listed-incremental` file tracks metadata between runs. Always use the **same** snapshot file across a backup set.

---

## Common Recipes

### Backup home directory with datestamp
```bash
tar -czf ~/backup-home-$(date +%Y%m%d).tar.gz \
    --exclude=~/.cache \
    --exclude=~/.local/share/Trash \
    ~/
```

### Backup /etc configuration
```bash
tar -czpf /backup/etc-$(date +%Y%m%d).tar.gz /etc/
```

### Clone a directory to a remote server
```bash
tar -cpf - /var/www/html | \
    ssh user@server "tar -xpf - -C /var/www/html"
```

### Create a reproducible archive (fixed timestamps)
```bash
tar -czf archive.tar.gz \
    --mtime='2025-01-01 00:00:00' \
    --sort=name \
    --owner=0 --group=0 \
    dir/
```

### Split large archive across multiple files
```bash
tar -czf - dir/ | split -b 4G - archive.tar.gz.part
# Reassemble and extract
cat archive.tar.gz.part* | tar -xzf -
```

### Find which archive contains a file
```bash
for f in /backup/*.tar.gz; do
    tar -tf "$f" | grep -q "filename.txt" && echo "$f"
done
```

### Verify archive integrity
```bash
tar -tzf archive.tar.gz > /dev/null && echo "OK" || echo "CORRUPT"
```

### Extract newest version of a file from multiple archives
```bash
tar -xOf latest.tar.gz path/to/file.conf > restored.conf
```

---

## File Extension Reference

| Extension | Compression |
|-----------|------------|
| `.tar` | None |
| `.tar.gz` / `.tgz` | gzip |
| `.tar.bz2` / `.tbz2` / `.tbz` | bzip2 |
| `.tar.xz` / `.txz` | xz |
| `.tar.zst` | zstd |
| `.tar.lz4` | lz4 |
| `.tar.lzma` | lzma |
| `.tar.Z` | compress |

---

## Permissions & Ownership

| Scenario | Flags to use |
|----------|-------------|
| Preserve all permissions | `-p` / `--preserve-permissions` |
| Preserve numeric UID/GID | `--numeric-owner` |
| Restore as different user | run as that user, or use `--owner` / `--group` |
| Archive with ACLs | `--acls` |
| Archive with xattrs | `--xattrs` |
| Archive with SELinux labels | `--selinux` |
| Archive with all security attrs | `--acls --xattrs --selinux` |

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Some files differ (when using `--compare`) or non-fatal warnings |
| `2` | Fatal error — archive may be incomplete |

---

## Tips & Best Practices

1. **Always use `-f`** — without it, tar defaults to the tape device, which is rarely what you want.
2. **Test before you trust** — verify archives with `tar -tzf archive.tar.gz` or `tar -df archive.tar.gz`.
3. **Avoid absolute paths** — tar strips leading `/` by default. Use `-P` only if you are sure, and only restore to the exact same system layout.
4. **Use `--strip-components=1`** when extracting archives that have a top-level wrapper directory.
5. **Pipe through `pv`** for progress visibility on large archives: `tar -czf - dir/ | pv > archive.tar.gz`.
6. **Use parallel compression** (`pigz`, `lbzip2`, `pixz`) to dramatically speed up creation of large archives on multi-core systems.
7. **Reproducible builds** — use `--sort=name`, `--mtime`, `--owner=0`, and `--group=0` to create byte-for-byte identical archives.
8. **Snapshot-based incrementals** — use `--listed-incremental` for proper incremental backups rather than relying on `--newer`.
9. **Don't compress already-compressed data** — skip `-z`/`-j`/`-J` when archiving JPEG, MP4, ZIP, or other already-compressed formats; it wastes CPU.
10. **Prefer `.tar.zst`** for modern use cases — zstd offers near-gzip speed with near-xz compression ratios.

---

## See Also

| Tool | Purpose |
|------|---------|
| `gzip` / `gunzip` | Compress/decompress `.gz` files |
| `bzip2` / `bunzip2` | Compress/decompress `.bz2` files |
| `xz` / `unxz` | Compress/decompress `.xz` files |
| `zstd` / `unzstd` | Compress/decompress `.zst` files |
| `pv` | Monitor pipeline throughput |
| `pigz` | Parallel gzip |
| `rsync` | Incremental file synchronisation |
| `dump` / `restore` | Filesystem-level backup (ext2/3/4) |
| `cpio` | Alternative archive format |
| `7z` | High-compression archiver (cross-platform) |

---

*GNU tar is the reference implementation on Linux. BSD tar (macOS default) is largely compatible but differs in some flags. Always check `man tar` for your specific version.*
