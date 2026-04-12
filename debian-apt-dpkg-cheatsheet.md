# Debian apt & dpkg Cheatsheet

A complete reference for managing packages on Debian, Ubuntu, and derivatives using `apt`, `apt-get`, `apt-cache`, and `dpkg`.

---

## Quick Reference — Most Common Commands

```bash
apt update                        # refresh package index
apt upgrade                       # upgrade all installed packages
apt install <pkg>                 # install a package
apt remove <pkg>                  # remove a package (keep config)
apt purge <pkg>                   # remove package + config files
apt autoremove                    # remove unused dependencies
apt search <term>                 # search for packages
apt show <pkg>                    # show package details
apt list --installed              # list installed packages
dpkg -l                           # list all installed packages
dpkg -i package.deb               # install a local .deb file
```

---

## apt vs apt-get vs apt-cache

| Tool | Purpose |
|------|---------|
| `apt` | Modern high-level interface — recommended for interactive use |
| `apt-get` | Traditional high-level tool — preferred in scripts (stable output) |
| `apt-cache` | Query the package cache and metadata |
| `apt-file` | Search for files within packages (requires separate install) |
| `dpkg` | Low-level package manager — installs/removes `.deb` files directly |
| `dpkg-query` | Query the dpkg package database |

> **Rule of thumb:** Use `apt` interactively, use `apt-get` in scripts.

---

## Package Index Management

### apt
```bash
apt update                        # update package index from all sources
apt update -q                     # quiet update (minimal output)
```

### apt-get
```bash
apt-get update                    # update package index
apt-get update --allow-releaseinfo-change  # allow suite changes (e.g. stable → stable-updates)
```

---

## Installing Packages

### apt
```bash
apt install <pkg>                 # install a package
apt install <pkg1> <pkg2>         # install multiple packages
apt install <pkg>=<version>       # install specific version
apt install ./package.deb         # install local .deb file
apt install -y <pkg>              # install without confirmation prompt
apt install --no-install-recommends <pkg>  # skip recommended packages
apt install --reinstall <pkg>     # reinstall an already installed package
apt install -f                    # fix broken dependencies
```

### apt-get
```bash
apt-get install <pkg>
apt-get install -y <pkg>
apt-get install --only-upgrade <pkg>    # upgrade a single package only
apt-get install --no-upgrade <pkg>      # install without upgrading if present
apt-get install -f                      # fix broken installs
apt-get install --reinstall <pkg>
apt-get install --install-suggests <pkg>  # also install suggested packages
```

### dpkg (local .deb files)
```bash
dpkg -i package.deb               # install a .deb file
dpkg -i *.deb                     # install all .deb files in current dir
dpkg --unpack package.deb         # unpack without configuring
dpkg --configure <pkg>            # configure an unpacked package
dpkg --configure -a               # configure all unpacked packages
```

---

## Removing Packages

### apt
```bash
apt remove <pkg>                  # remove package, keep config files
apt remove <pkg1> <pkg2>          # remove multiple packages
apt purge <pkg>                   # remove package + all config files
apt autoremove                    # remove orphaned/unused dependencies
apt autoremove --purge            # autoremove + purge config files
```

### apt-get
```bash
apt-get remove <pkg>
apt-get purge <pkg>
apt-get autoremove
apt-get autoremove --purge
apt-get remove --purge <pkg>      # same as purge
```

### dpkg
```bash
dpkg -r <pkg>                     # remove package, keep config
dpkg -P <pkg>                     # purge package + config files
dpkg --remove <pkg>
dpkg --purge <pkg>
```

---

## Upgrading Packages

### apt
```bash
apt upgrade                       # upgrade all upgradeable packages (safe)
apt full-upgrade                  # upgrade + remove/install deps as needed
apt upgrade <pkg>                 # upgrade a specific package
apt-get dist-upgrade              # equivalent to full-upgrade
```

### apt-get
```bash
apt-get upgrade                   # safe upgrade (no removes)
apt-get dist-upgrade              # full upgrade (may add/remove packages)
apt-get upgrade -y                # non-interactive upgrade
```

### Simulating upgrades (dry run)
```bash
apt upgrade --simulate            # show what would be upgraded
apt-get upgrade -s                # simulate upgrade
apt-get dist-upgrade --dry-run    # dry run dist-upgrade
```

---

## Searching & Querying

### apt / apt-cache
```bash
apt search <term>                 # search package names and descriptions
apt show <pkg>                    # show detailed package info
apt list                          # list all available packages
apt list --installed              # list installed packages
apt list --upgradeable            # list packages with available upgrades
apt list --all-versions           # list all versions of all packages
apt depends <pkg>                 # show dependencies
apt rdepends <pkg>                # show reverse dependencies

apt-cache search <term>           # search by name and description
apt-cache search --names-only <term>  # search names only
apt-cache show <pkg>              # show package details
apt-cache showpkg <pkg>           # show full package info including deps
apt-cache policy <pkg>            # show installed/candidate versions and priorities
apt-cache depends <pkg>           # show dependencies
apt-cache rdepends <pkg>          # show reverse dependencies
apt-cache pkgnames                # list all available package names
apt-cache pkgnames <prefix>       # list packages starting with prefix
apt-cache stats                   # show cache statistics
apt-cache dump                    # dump entire package cache
apt-cache unmet                   # show unmet dependencies
```

### dpkg-query
```bash
dpkg -l                           # list all installed packages
dpkg -l <pattern>                 # list packages matching pattern
dpkg -l 'lib*'                    # list all packages starting with lib
dpkg -s <pkg>                     # show package status/details
dpkg -L <pkg>                     # list files installed by package
dpkg -S <file>                    # find which package owns a file
dpkg -S /usr/bin/ssh              # example: who owns /usr/bin/ssh
dpkg --get-selections             # list all packages with state
dpkg --get-selections | grep -v deinstall  # list installed only
dpkg-query -W <pkg>               # show package name and version
dpkg-query -W -f='${Package} ${Version}\n'  # custom output format
dpkg-query -l '*nginx*'           # list packages matching pattern
```

---

## Package Information & Details

```bash
apt show <pkg>                    # name, version, deps, description
apt-cache show <pkg>              # full package metadata
apt-cache showsrc <pkg>           # show source package info
apt-cache policy <pkg>            # installed vs candidate version + priorities

dpkg -s <pkg>                     # installed package status
dpkg -I package.deb               # info about a .deb file (before installing)
dpkg -c package.deb               # list contents of a .deb file
```

### Useful dpkg-query format strings
```bash
# List package name, version, architecture
dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\n'

# List explicitly installed packages (not auto-installed)
apt-mark showmanual

# List automatically installed packages
apt-mark showauto

# List packages by install size (largest first)
dpkg-query -W -f='${Installed-Size}\t${Package}\n' | sort -rn | head -20
```

---

## Holding & Pinning Packages

### Holding packages (prevent upgrade)
```bash
apt-mark hold <pkg>               # hold a package at current version
apt-mark unhold <pkg>             # release hold
apt-mark showhold                 # list held packages

dpkg --set-selections <<< "<pkg> hold"   # hold via dpkg
dpkg --get-selections | grep hold        # show held packages
```

### Pinning with apt preferences
Create `/etc/apt/preferences.d/mypins`:
```
# Pin a package to a specific version
Package: nginx
Pin: version 1.24.*
Pin-Priority: 1001

# Prefer packages from a specific release
Package: *
Pin: release a=stable
Pin-Priority: 900

# Block a package from being installed
Package: snapd
Pin: release *
Pin-Priority: -1

# Pin to a specific origin
Package: docker-ce
Pin: origin download.docker.com
Pin-Priority: 500
```

Pin priorities:
| Priority | Behaviour |
|----------|-----------|
| `< 0` | Prevents installation |
| `0–99` | Can only install if no other version available |
| `100` | Installed version (default) |
| `500` | Default for packages in `/etc/apt/sources.list` |
| `990` | Preferred — used by apt upgrade |
| `≥ 1000` | Installed even if downgrade is required |
| `1001` | Downgrade if needed |

```bash
apt-cache policy                  # show priorities for all repos
apt-cache policy <pkg>            # show priorities for a specific package
```

---

## Repository Management

### Sources list formats

**One-line format** (`/etc/apt/sources.list`):
```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm main
deb http://security.debian.org/debian-security bookworm-security main
deb http://deb.debian.org/debian bookworm-updates main
```

**DEB822 format** (`/etc/apt/sources.list.d/<name>.sources`):
```
Types: deb
URIs: http://deb.debian.org/debian
Suites: bookworm bookworm-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

### Managing repositories
```bash
# Add a repository (Ubuntu/derivatives)
add-apt-repository ppa:user/ppa-name
add-apt-repository "deb http://repo.example.com/apt stable main"
add-apt-repository --remove ppa:user/ppa-name

# Manually add a third-party repo key (modern method)
curl -fsSL https://repo.example.com/key.gpg | \
    gpg --dearmor -o /usr/share/keyrings/example-archive-keyring.gpg

# Add repo using the key
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/example-archive-keyring.gpg] \
    https://repo.example.com/apt stable main" | \
    tee /etc/apt/sources.list.d/example.list

# List configured repositories
apt-cache policy
grep -r "^deb" /etc/apt/sources.list /etc/apt/sources.list.d/
```

### Repository options in sources
```
deb [arch=amd64]                  # limit to architecture
deb [arch=amd64,arm64]            # multiple architectures
deb [signed-by=/path/to/key.gpg]  # specify signing key
deb [trusted=yes]                 # skip signature check (not recommended)
deb [check-valid-until=no]        # ignore expiry (for old snapshots)
```

---

## APT Configuration

### Key configuration files
| File/Directory | Purpose |
|----------------|---------|
| `/etc/apt/sources.list` | Primary repository list |
| `/etc/apt/sources.list.d/` | Drop-in repository files |
| `/etc/apt/preferences` | Global pinning preferences |
| `/etc/apt/preferences.d/` | Drop-in pinning files |
| `/etc/apt/apt.conf` | Main APT configuration |
| `/etc/apt/apt.conf.d/` | Drop-in configuration files |
| `/etc/apt/trusted.gpg.d/` | Trusted signing keys |
| `/var/cache/apt/archives/` | Downloaded `.deb` package cache |
| `/var/lib/apt/lists/` | Package index files |
| `/var/lib/dpkg/` | dpkg package database |
| `/var/log/apt/history.log` | APT install/remove history |
| `/var/log/apt/term.log` | Terminal output log |
| `/var/log/dpkg.log` | dpkg operation log |

### Useful apt.conf.d snippets

**Disable recommended packages globally** (`/etc/apt/apt.conf.d/99no-recommends`):
```
APT::Install-Recommends "false";
APT::Install-Suggests "false";
```

**Set a proxy** (`/etc/apt/apt.conf.d/99proxy`):
```
Acquire::http::Proxy "http://proxy.example.com:3128";
Acquire::https::Proxy "http://proxy.example.com:3128";
```

**Keep downloaded packages** (`/etc/apt/apt.conf.d/99keep-cache`):
```
Binary::apt::APT::Keep-Downloaded-Packages "true";
```

**Periodic automatic updates** (`/etc/apt/apt.conf.d/20auto-upgrades`):
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

---

## Cache Management

```bash
apt clean                         # remove all cached .deb files
apt autoclean                     # remove cached .debs that can no longer be downloaded
apt-get clean
apt-get autoclean

du -sh /var/cache/apt/archives/   # check cache size

# List cached packages
ls /var/cache/apt/archives/*.deb

# Reinstall from cache (if still present)
dpkg -i /var/cache/apt/archives/package_version_arch.deb
```

---

## Dealing with Broken Packages

```bash
apt install -f                    # fix broken dependencies
apt-get install -f
dpkg --configure -a               # configure any unconfigured packages
apt --fix-broken install

# Force remove a stubborn package
dpkg --remove --force-remove-reinstreq <pkg>
dpkg --purge --force-all <pkg>

# Reconfigure an installed package
dpkg-reconfigure <pkg>
dpkg-reconfigure tzdata            # example: reconfigure timezone
dpkg-reconfigure locales           # example: reconfigure locales

# Rebuild dpkg database
dpkg --audit                      # check for partially installed packages
```

---

## Package States (dpkg)

The `dpkg -l` output columns:

```
||/ Name        Version      Architecture Description
ii  bash        5.2.15       amd64        GNU Bourne Again SHell
```

| Column 1 (desired) | Meaning |
|--------------------|---------|
| `u` | Unknown |
| `i` | Install |
| `h` | Hold |
| `r` | Remove |
| `p` | Purge |

| Column 2 (status) | Meaning |
|-------------------|---------|
| `n` | Not installed |
| `i` | Installed |
| `c` | Config files only (removed but config remains) |
| `u` | Unpacked |
| `f` | Failed config |
| `h` | Half installed |
| `w` | Awaiting triggers |
| `t` | Triggers pending |

| Column 3 (error) | Meaning |
|------------------|---------|
| (blank) | No error |
| `h` | Hold |
| `r` | Reinstall required |
| `x` | Both hold and reinstall |

---

## Automating & Scripting

```bash
# Non-interactive (suppress prompts)
DEBIAN_FRONTEND=noninteractive apt-get install -y <pkg>

# Prevent any interactive dialogs
DEBIAN_FRONTEND=noninteractive apt-get -yq install <pkg>

# Preseed debconf answers before install
echo "<pkg> <pkg>/key select value" | debconf-set-selections
apt-get install -y <pkg>

# Export and restore package selections
dpkg --get-selections > packages.list
dpkg --set-selections < packages.list
apt-get dselect-upgrade

# Export installed packages list (name only)
apt list --installed 2>/dev/null | cut -d/ -f1 > installed.txt

# Reinstall from list on new system
xargs apt-get install -y < installed.txt

# Run apt in a script safely
apt-get -qq update                # quiet, machine-readable
apt-get -qq install -y <pkg>
```

---

## apt-file — Search File Contents of Packages

```bash
# Install apt-file
apt install apt-file

# Update apt-file cache
apt-file update

# Search which package provides a file
apt-file search /usr/bin/convert
apt-file search libssl.so

# Search by pattern
apt-file search --regexp 'bin/py.*3'

# List all files in a package (even if not installed)
apt-file list <pkg>
apt-file list nginx
```

---

## unattended-upgrades

```bash
# Install
apt install unattended-upgrades

# Enable
dpkg-reconfigure --priority=low unattended-upgrades

# Manual test run
unattended-upgrade --dry-run --debug

# Force run now
unattended-upgrade -d

# Configuration file
/etc/apt/apt.conf.d/50unattended-upgrades
```

---

## Useful One-Liners

```bash
# List recently installed packages
grep " install " /var/log/dpkg.log | tail -20

# List packages installed today
grep "$(date +%Y-%m-%d)" /var/log/dpkg.log | grep " install "

# Find the 20 largest installed packages
dpkg-query -W -f='${Installed-Size}\t${Package}\n' | \
    sort -rn | head -20 | awk '{printf "%s MB\t%s\n", $1/1024, $2}'

# Count total installed packages
dpkg -l | grep -c '^ii'

# List packages not from official repos
apt-show-versions | grep -v "^.*:.*debian"

# Check if a package is installed
dpkg -l <pkg> | grep -q '^ii' && echo "installed" || echo "not installed"

# Download package without installing
apt-get download <pkg>

# Download source package
apt-get source <pkg>

# Show changelog
apt-get changelog <pkg>

# Show package dependencies as a tree
apt-cache depends --recurse --no-recommends --no-suggests \
    --no-conflicts --no-breaks --no-replaces --no-enhances <pkg>

# Simulate install (show what would happen)
apt-get install -s <pkg>

# Find which package provides a command
dpkg -S $(which ssh)
```

---

## Debian Release Codenames

| Release | Codename | Status |
|---------|----------|--------|
| Debian 13 | Trixie | Testing |
| Debian 12 | Bookworm | Stable |
| Debian 11 | Bullseye | Oldstable |
| Debian 10 | Buster | LTS |
| Debian 9 | Stretch | ELTS |

### Repository suites
| Suite | Description |
|-------|-------------|
| `stable` | Current stable release |
| `testing` | Next stable (rolling) |
| `unstable` (sid) | Development, always latest |
| `experimental` | Pre-unstable, may be broken |
| `stable-updates` | Non-security stable updates |
| `stable-backports` | Newer packages backported to stable |
| `stable-security` | Security updates |

---

## Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `E: Unable to acquire the dpkg frontend lock` | Another apt process is running; wait or kill it |
| `E: dpkg was interrupted` | Run `dpkg --configure -a` |
| `E: Unmet dependencies` | Run `apt install -f` |
| `W: GPG error: NO_PUBKEY` | Import missing key: `apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <KEY>` |
| `E: Repository ... changed its 'Suite'` | Run `apt update --allow-releaseinfo-change` |
| `dpkg: error: ... is already installed and configured` | Run `apt install --reinstall <pkg>` |
| `Sub-process /usr/bin/dpkg returned an error code (1)` | Run `dpkg --configure -a` then `apt install -f` |
| `E: Package has no installation candidate` | Package not in sources; check `apt-cache policy <pkg>` |
| `Hash Sum mismatch` | Stale cache; run `apt clean` then `apt update` |

---

## Security Best Practices

1. **Always run `apt update` before installing** to ensure you get the latest security patches.
2. **Use `apt upgrade` regularly** — set up `unattended-upgrades` for automatic security updates.
3. **Verify repository signatures** — never use `trusted=yes` in production.
4. **Store GPG keys in `/usr/share/keyrings/`** — the deprecated `apt-key` stores keys globally in the trusted keyring.
5. **Use `--no-install-recommends`** to minimise attack surface on servers.
6. **Audit installed packages periodically** — remove packages you no longer need with `apt autoremove --purge`.
7. **Pin critical packages** to prevent unintended upgrades during `dist-upgrade`.

---

*This cheatsheet targets Debian and Ubuntu. Some commands (e.g. `add-apt-repository`, PPA support) are Ubuntu/derivatives only. Always check `man apt`, `man apt-get`, and `man dpkg` for your specific version.*
