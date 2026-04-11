# systemd Cheatsheet

## Table of Contents
1. [Unit Types](#unit-types)
2. [systemctl — Service Management](#systemctl--service-management)
3. [systemctl — System State](#systemctl--system-state)
4. [systemctl — Unit Inspection](#systemctl--unit-inspection)
5. [Unit File Structure](#unit-file-structure)
6. [Service Unit Options](#service-unit-options)
7. [Timer Units](#timer-units)
8. [Target Units](#target-units)
9. [Mount & Automount Units](#mount--automount-units)
10. [Socket Units](#socket-units)
11. [Path Units](#path-units)
12. [journalctl — Log Management](#journalctl--log-management)
13. [systemd-analyze — Boot Performance](#systemd-analyze--boot-performance)
14. [loginctl — User Sessions](#loginctl--user-sessions)
15. [hostnamectl / timedatectl / localectl](#hostnamectl--timedatectl--localectl)
16. [User Systemd (--user)](#user-systemd---user)
17. [Override / Drop-in Files](#override--drop-in-files)
18. [Environment & Security Hardening](#environment--security-hardening)
19. [Useful Patterns](#useful-patterns)
20. [Quick Reference Card](#quick-reference-card)

---

## Unit Types

| Extension | Purpose |
|-----------|---------|
| `.service` | Manages a daemon or one-shot process |
| `.timer` | Cron-like scheduled activation |
| `.socket` | Socket-based activation |
| `.target` | Groups units; acts as a synchronisation point |
| `.mount` | Manages a mount point |
| `.automount` | On-demand mounting |
| `.path` | Monitors a file/directory for changes |
| `.device` | Represents a kernel device |
| `.scope` | Manages externally-created processes |
| `.slice` | Hierarchical group for resource management (cgroups) |
| `.swap` | Manages swap space |

---

## systemctl — Service Management

```bash
# Start / Stop / Restart
systemctl start   <unit>
systemctl stop    <unit>
systemctl restart <unit>
systemctl reload  <unit>          # Reload config without stopping
systemctl reload-or-restart <unit>

# Enable / Disable (persist across reboots)
systemctl enable  <unit>          # Create symlink to start at boot
systemctl disable <unit>          # Remove symlink
systemctl enable --now <unit>     # Enable AND start immediately
systemctl disable --now <unit>    # Disable AND stop immediately

# Mask / Unmask (prevent any start)
systemctl mask   <unit>           # Symlink to /dev/null — cannot start
systemctl unmask <unit>

# Re-read unit files after changes
systemctl daemon-reload

# Reset a failed unit
systemctl reset-failed <unit>
systemctl reset-failed            # Reset all failed units
```

---

## systemctl — System State

```bash
# Power management
systemctl poweroff
systemctl reboot
systemctl suspend
systemctl hibernate
systemctl hybrid-sleep
systemctl halt

# Rescue / emergency modes
systemctl rescue
systemctl emergency

# List all targets
systemctl list-units --type=target

# Switch target (runlevel equivalent)
systemctl isolate multi-user.target
systemctl isolate graphical.target

# Set default boot target
systemctl set-default multi-user.target
systemctl get-default
```

---

## systemctl — Unit Inspection

```bash
# Status (with recent logs)
systemctl status <unit>
systemctl status <unit> -l        # Full (untruncated) output
systemctl status                  # Overall system status

# Check if unit is active / enabled / failed
systemctl is-active  <unit>
systemctl is-enabled <unit>
systemctl is-failed  <unit>

# List units
systemctl list-units                          # All active units
systemctl list-units --all                    # Including inactive
systemctl list-units --type=service           # Only services
systemctl list-units --state=failed           # Only failed
systemctl list-units --state=running          # Only running

# List unit files
systemctl list-unit-files                     # All installed unit files
systemctl list-unit-files --type=service
systemctl list-unit-files --state=enabled

# Show unit file
systemctl cat <unit>

# Show all properties of a unit
systemctl show <unit>
systemctl show <unit> -p MainPID,ActiveState  # Specific properties

# List dependencies
systemctl list-dependencies <unit>
systemctl list-dependencies <unit> --reverse  # Who depends on this unit

# Edit a unit (opens drop-in override)
systemctl edit <unit>
systemctl edit --full <unit>      # Edit the full unit file copy
```

---

## Unit File Structure

Unit files live in:

| Path | Purpose |
|------|---------|
| `/lib/systemd/system/` | Package-installed units (don't edit) |
| `/usr/lib/systemd/system/` | Same as above on some distros |
| `/etc/systemd/system/` | Local/admin overrides (highest priority) |
| `/run/systemd/system/` | Runtime-generated units |
| `~/.config/systemd/user/` | User units |

### Minimal Service Unit

```ini
[Unit]
Description=My Application
After=network.target

[Service]
ExecStart=/usr/bin/myapp --config /etc/myapp/config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## Service Unit Options

### [Unit] Section

```ini
[Unit]
Description=Human-readable name
Documentation=https://example.com/docs  # or man:sshd(8)

# Ordering (does NOT imply dependency)
After=network.target postgresql.service
Before=nginx.service

# Dependencies
Requires=postgresql.service       # Hard dependency — fails together
Wants=redis.service               # Soft dependency — starts if possible
BindsTo=device.service            # Stop this unit if dependency stops
PartOf=parent.service             # Stop/restart with parent

# Conditions (skip unit silently if false)
ConditionPathExists=/etc/myapp/config.yml
ConditionFileNotEmpty=/etc/myapp/config.yml
ConditionHost=myserver
ConditionVirtualization=no        # Skip inside containers/VMs

# Assertions (fail unit hard if false)
AssertPathExists=/usr/bin/myapp

OnFailure=notify-failure@%n.service   # Unit to activate on failure
```

### [Service] Section

```ini
[Service]
# Service type
Type=simple       # Default: process stays in foreground
Type=forking      # Process daemonises (forks to background)
Type=oneshot      # Runs and exits; unit stays active until done
Type=notify       # Like simple but sends sd_notify() when ready
Type=dbus         # Ready when D-Bus name acquired
Type=idle         # Like simple but waits for jobs to complete

# Commands
ExecStart=/usr/bin/myapp arg1 arg2
ExecStartPre=/usr/bin/setup-script     # Before ExecStart
ExecStartPost=/usr/bin/post-script     # After ExecStart
ExecReload=/bin/kill -HUP $MAINPID     # On systemctl reload
ExecStop=/usr/bin/myapp --stop         # On systemctl stop
ExecStopPost=/usr/bin/cleanup          # Always runs after stop

# User / group
User=myuser
Group=mygroup
SupplementaryGroups=docker audio

# Working directory & environment
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/env         # Key=value file
Environment="KEY=value" "FOO=bar"

# Restart policy
Restart=no                 # Never restart (default)
Restart=on-success         # Only on clean exit
Restart=on-failure         # On non-zero exit, signal, timeout
Restart=on-abnormal        # On signal, timeout, watchdog
Restart=always             # Always restart
RestartSec=5               # Wait before restarting (seconds)
StartLimitIntervalSec=60   # Interval for start limit counting
StartLimitBurst=3          # Max starts in interval before giving up

# Timeouts
TimeoutStartSec=30
TimeoutStopSec=30
TimeoutSec=30              # Sets both start and stop
TimeoutStartSec=infinity   # Disable timeout

# Standard streams
StandardOutput=journal
StandardError=journal
StandardInput=null
SyslogIdentifier=myapp     # Tag in journal

# Misc
PIDFile=/run/myapp.pid     # For Type=forking
RemainAfterExit=yes        # Keep unit active after process exits
KillMode=control-group     # How to kill: control-group|process|mixed|none
KillSignal=SIGTERM
FinalKillSignal=SIGKILL
```

### [Install] Section

```ini
[Install]
WantedBy=multi-user.target    # Most services use this
WantedBy=graphical.target     # GUI-dependent services
RequiredBy=other.service
Also=myapp-helper.service     # Also enable/disable this unit
Alias=myapp-compat.service    # Alternative name
```

---

## Timer Units

Timers replace cron jobs. A `foo.timer` activates `foo.service` by default.

### Monotonic Timers (relative to system events)

```ini
[Timer]
OnBootSec=5min          # 5 minutes after boot
OnActiveSec=10min       # 10 minutes after timer activation
OnStartupSec=1min       # 1 minute after systemd startup
OnUnitActiveSec=1h      # 1 hour after the service last ran
OnUnitInactiveSec=30min # 30 minutes after service last stopped
```

### Realtime / Calendar Timers (like cron)

```ini
[Timer]
OnCalendar=daily                   # Every day at midnight
OnCalendar=weekly                  # Every Monday at midnight
OnCalendar=monthly
OnCalendar=hourly
OnCalendar=*:0/15                  # Every 15 minutes
OnCalendar=Mon..Fri 09:00:00       # Weekdays at 9 AM
OnCalendar=*-*-* 02:30:00          # Every day at 02:30
OnCalendar=Sat *-*-1..7 18:00:00   # First Saturday of month
```

### Full Timer Unit Example

```ini
[Unit]
Description=Run myapp daily backup

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=300            # Randomise up to 5 minutes
Persistent=true                   # Run missed jobs after downtime
AccuracySec=1s                    # How precise the timer is

[Install]
WantedBy=timers.target
```

```bash
# Useful timer commands
systemctl list-timers             # All active timers with next run time
systemctl list-timers --all       # Including inactive
systemd-run --on-calendar="*:0/10" /path/to/script   # Transient timer
```

---

## Target Units

Targets are synchronisation points, similar to SysV runlevels.

| Target | SysV Equivalent | Description |
|--------|----------------|-------------|
| `poweroff.target` | 0 | Shut down |
| `rescue.target` | 1 | Single-user / rescue |
| `multi-user.target` | 3 | Multi-user, no GUI |
| `graphical.target` | 5 | Multi-user with GUI |
| `reboot.target` | 6 | Reboot |
| `emergency.target` | — | Minimal emergency shell |
| `network.target` | — | Network available |
| `network-online.target` | — | Network fully configured |
| `sleep.target` | — | System sleeping |
| `default.target` | — | Symlink to boot target |

```ini
# Custom target example
[Unit]
Description=My App Stack
Requires=multi-user.target
After=multi-user.target
AllowIsolate=yes

[Install]
WantedBy=multi-user.target
```

---

## Mount & Automount Units

Unit names must correspond to the mount path (encode `/` as `-`).
`/mnt/data` → `mnt-data.mount`

```ini
# /etc/systemd/system/mnt-data.mount
[Unit]
Description=Data Volume

[Mount]
What=/dev/sdb1           # Or UUID=... or LABEL=...
Where=/mnt/data
Type=ext4
Options=defaults,noatime

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/mnt-data.automount
[Unit]
Description=Automount Data Volume

[Automount]
Where=/mnt/data
TimeoutIdleSec=60        # Unmount after 60s idle

[Install]
WantedBy=multi-user.target
```

```bash
# Generate unit name from path
systemd-escape -p /mnt/my data    # → mnt-my\x20data.mount
```

---

## Socket Units

Socket activation: systemd listens on a socket and starts the service on first connection.

```ini
# myapp.socket
[Unit]
Description=MyApp Socket

[Socket]
ListenStream=8080              # TCP port
ListenStream=/run/myapp.sock   # Unix socket
Accept=no                      # One instance handles all connections

[Install]
WantedBy=sockets.target
```

```ini
# myapp.service (activated by socket)
[Unit]
Description=MyApp Service

[Service]
ExecStart=/usr/bin/myapp
StandardInput=socket           # Receive connection on stdin
```

---

## Path Units

Trigger a service when a file or directory changes.

```ini
# myapp-watch.path
[Unit]
Description=Watch /etc/myapp for changes

[Path]
PathChanged=/etc/myapp/config.yml    # Trigger on modification
PathExists=/tmp/trigger-file         # Trigger when file appears
PathExistsGlob=/spool/*.job          # Trigger on glob match
DirectoryNotEmpty=/var/spool/myapp   # Trigger when dir non-empty
MakeDirectory=yes                    # Create directory if missing
Unit=myapp-reload.service            # Unit to activate (default: same name .service)

[Install]
WantedBy=multi-user.target
```

---

## journalctl — Log Management

```bash
# View logs
journalctl                           # All logs (oldest first)
journalctl -r                        # Reverse order (newest first)
journalctl -f                        # Follow (like tail -f)
journalctl -n 50                     # Last 50 lines
journalctl -e                        # Jump to end

# Filter by unit
journalctl -u nginx.service
journalctl -u nginx -u php-fpm       # Multiple units
journalctl -u nginx -f               # Follow a unit

# Filter by time
journalctl --since "2024-01-15 10:00:00"
journalctl --since "1 hour ago"
journalctl --until "2024-01-15 11:00:00"
journalctl --since today
journalctl --since yesterday
journalctl -b                        # Current boot only
journalctl -b -1                     # Previous boot
journalctl --list-boots              # List all recorded boots

# Filter by priority
journalctl -p err                    # Errors and above
journalctl -p warning..err           # Warning to error range
# Priorities: emerg alert crit err warning notice info debug

# Filter by process / PID / UID
journalctl _PID=1234
journalctl _UID=1000
journalctl _COMM=nginx               # By executable name

# Output formats
journalctl -u nginx -o json          # JSON output
journalctl -u nginx -o json-pretty
journalctl -u nginx -o short-iso     # ISO timestamps
journalctl -u nginx -o cat           # Message text only
journalctl -u nginx -o verbose       # All metadata fields

# Kernel messages only
journalctl -k
journalctl -k -b                     # Kernel messages this boot

# Search / grep
journalctl -u nginx -g "error"       # Grep within unit logs
journalctl | grep "pattern"

# Disk usage
journalctl --disk-usage

# Vacuum old logs
journalctl --vacuum-size=500M        # Keep only 500 MB
journalctl --vacuum-time=2weeks      # Keep only last 2 weeks
journalctl --vacuum-files=5          # Keep only 5 archive files

# Verify journal integrity
journalctl --verify
```

### /etc/systemd/journald.conf

```ini
[Journal]
Storage=persistent         # persistent|volatile|auto|none
Compress=yes
SystemMaxUse=2G            # Max disk space for system journal
SystemKeepFree=1G          # Min free space to keep
SystemMaxFileSize=200M     # Max size per journal file
MaxRetentionSec=1month     # Delete logs older than this
MaxFileSec=1week           # Rotate files older than this
RateLimitIntervalSec=30s
RateLimitBurst=10000
ForwardToSyslog=no
```

---

## systemd-analyze — Boot Performance

```bash
# Total boot time
systemd-analyze

# Time spent in each unit
systemd-analyze blame

# Critical path of the boot
systemd-analyze critical-chain
systemd-analyze critical-chain nginx.service

# Generate SVG boot chart
systemd-analyze plot > boot.svg

# Check unit file syntax
systemd-analyze verify /etc/systemd/system/myapp.service

# Security scoring for a service
systemd-analyze security nginx.service

# Inspect calendar expressions
systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
systemd-analyze calendar --iterations=5 "daily"

# Inspect timespan strings
systemd-analyze timespan "1h 30min"
```

---

## loginctl — User Sessions

```bash
loginctl list-sessions          # Active login sessions
loginctl list-users             # Logged-in users
loginctl list-seats             # Hardware seats

loginctl show-session <id>      # Session properties
loginctl show-user <user>       # User properties

loginctl terminate-session <id> # Kill a session
loginctl terminate-user <user>  # Kill all user sessions
loginctl lock-session <id>      # Lock a session
loginctl unlock-session <id>
loginctl lock-sessions          # Lock all sessions

loginctl enable-linger <user>   # Allow user services to run without login
loginctl disable-linger <user>
```

---

## hostnamectl / timedatectl / localectl

```bash
# Hostname
hostnamectl                          # Show current hostname info
hostnamectl set-hostname myserver    # Set hostname
hostnamectl set-hostname "My Server" --pretty

# Time & Date
timedatectl                          # Show time/date/timezone info
timedatectl set-time "2024-06-01 12:00:00"
timedatectl set-timezone Europe/London
timedatectl list-timezones
timedatectl set-ntp true             # Enable NTP sync

# Locale & Keyboard
localectl                            # Show locale/keyboard info
localectl set-locale LANG=en_GB.UTF-8
localectl set-keymap uk              # Console keymap
localectl list-locales
localectl list-keymaps
```

---

## User Systemd (--user)

Each user has their own systemd instance. Units live in `~/.config/systemd/user/`.

```bash
# All standard commands work with --user
systemctl --user start   myapp.service
systemctl --user stop    myapp.service
systemctl --user enable  myapp.service
systemctl --user status  myapp.service
systemctl --user daemon-reload

# View user journal
journalctl --user
journalctl --user -u myapp.service -f

# Enable linger so user services run without being logged in
loginctl enable-linger $USER
```

**Example user service** (`~/.config/systemd/user/myapp.service`):

```ini
[Unit]
Description=My User Application
After=default.target

[Service]
ExecStart=%h/bin/myapp
Restart=on-failure
Environment=DISPLAY=:0

[Install]
WantedBy=default.target
```

---

## Override / Drop-in Files

Override vendor unit files without modifying originals.

```bash
# Method 1: systemctl edit (creates drop-in automatically)
systemctl edit nginx.service
# Creates: /etc/systemd/system/nginx.service.d/override.conf

# Method 2: systemctl edit --full (full copy to edit)
systemctl edit --full nginx.service
# Creates: /etc/systemd/system/nginx.service

# Method 3: Manual drop-in
mkdir -p /etc/systemd/system/nginx.service.d/
vim /etc/systemd/system/nginx.service.d/override.conf
systemctl daemon-reload
```

**Example drop-in — add environment variable:**

```ini
# /etc/systemd/system/myapp.service.d/override.conf
[Service]
Environment="DEBUG=true"
Environment="LOG_LEVEL=verbose"
```

**Example drop-in — change restart policy:**

```ini
[Service]
Restart=always
RestartSec=10
```

**Example drop-in — add dependency:**

```ini
[Unit]
After=redis.service
Wants=redis.service
```

> **Note:** To clear a list directive (e.g. `ExecStart`), set it to empty first, then redefine:
> ```ini
> [Service]
> ExecStart=
> ExecStart=/usr/bin/myapp --new-flag
> ```

---

## Environment & Security Hardening

### Environment

```ini
[Service]
Environment="KEY=value"
EnvironmentFile=/etc/myapp/env          # file with KEY=value pairs
EnvironmentFile=-/etc/optional.env      # prefix - = ignore if missing
PassEnvironment=HOME USER               # Pass from systemd's environment
UnsetEnvironment=SENSITIVE_VAR
```

### Security Directives

```ini
[Service]
# Filesystem isolation
PrivateTmp=yes                  # Isolated /tmp and /var/tmp
ProtectSystem=strict            # /usr, /boot, /etc read-only
ProtectSystem=full              # /usr, /boot read-only
ProtectHome=yes                 # /home, /root, /run/user inaccessible
ReadWritePaths=/var/lib/myapp   # Allow write to specific path
ReadOnlyPaths=/etc/myapp
InaccessiblePaths=/home /root

# User/group namespacing
DynamicUser=yes                 # Create ephemeral user at start

# Capabilities
CapabilityBoundingSet=CAP_NET_BIND_SERVICE   # Only allow binding ports <1024
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes             # Prevent privilege escalation

# Network isolation
PrivateNetwork=yes              # Isolated network namespace
RestrictAddressFamilies=AF_INET AF_INET6   # Limit address families

# System call filtering
SystemCallFilter=@system-service     # Allow common service syscalls
SystemCallFilter=~@privileged        # Deny privileged syscalls
SystemCallArchitectures=native

# Misc
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
ProtectClock=yes
ProtectHostname=yes
ProtectKernelLogs=yes
```

```bash
# Check security score and suggestions
systemd-analyze security myapp.service
```

### Resource Control (cgroups)

```ini
[Service]
# CPU
CPUQuota=50%                    # Max 50% of one CPU
CPUWeight=100                   # Relative CPU priority (default 100)

# Memory
MemoryMax=512M                  # Hard memory limit
MemoryHigh=400M                 # Soft limit (start throttling)
MemorySwapMax=0                 # Disable swap

# IO
IOWeight=100                    # Relative IO priority
IOReadBandwidthMax=/dev/sda 50M # Limit read bandwidth
IOWriteBandwidthMax=/dev/sda 50M

# Tasks (processes/threads)
TasksMax=50
```

---

## Useful Patterns

### One-shot Service (run-once task)

```ini
[Unit]
Description=Database Migration
After=postgresql.service
ConditionPathExists=!/var/lib/myapp/.migrated

[Service]
Type=oneshot
ExecStart=/usr/bin/myapp migrate
ExecStartPost=/usr/bin/touch /var/lib/myapp/.migrated
RemainAfterExit=yes
User=myapp

[Install]
WantedBy=multi-user.target
```

### Notify on Service Failure

```ini
# /etc/systemd/system/notify-failure@.service
[Unit]
Description=Notify on failure of %i

[Service]
Type=oneshot
ExecStart=/usr/bin/send-alert "Service %i failed on %H"
```

```ini
# In your service unit
[Unit]
OnFailure=notify-failure@%n.service
```

### Templated Units (`@`)

```ini
# /etc/systemd/system/myapp@.service
[Unit]
Description=MyApp instance %i

[Service]
ExecStart=/usr/bin/myapp --port=%i
User=myapp

[Install]
WantedBy=multi-user.target
```

```bash
# Start multiple instances
systemctl start myapp@8080.service
systemctl start myapp@8081.service
systemctl enable myapp@8080.service
```

### Transient Units (run without unit file)

```bash
# Run a command as a transient service
systemd-run --unit=mytask /usr/bin/myscript

# With options
systemd-run --uid=myuser --working-directory=/opt \
  --property=MemoryMax=512M \
  /usr/bin/myapp

# Transient timer
systemd-run --on-calendar="*:0/15" /opt/myapp/poll.sh
systemd-run --on-active=30min --unit=delayed-task /bin/mycommand
```

### Specifier Variables (use in unit files)

| Specifier | Meaning |
|-----------|---------|
| `%n` | Full unit name |
| `%p` | Prefix (name without suffix) |
| `%i` | Instance name (from `unit@instance.service`) |
| `%H` | Hostname |
| `%u` | Username |
| `%h` | Home directory |
| `%t` | Runtime directory (`/run` or `$XDG_RUNTIME_DIR`) |
| `%S` | State directory |
| `%C` | Cache directory |
| `%L` | Log directory |
| `%f` | Full path from `%i` (unescaped) |

---

## Quick Reference Card

```
SERVICE LIFECYCLE
  start / stop / restart / reload    Manage a unit
  enable / disable                   Persist across reboots
  enable --now / disable --now       Combined start+enable
  mask / unmask                      Prevent / allow starting
  daemon-reload                      Re-read unit files after edits
  reset-failed                       Clear failed state

INSPECTION
  status <unit>                      Status + recent logs
  is-active / is-enabled / is-failed Quick state checks
  list-units --type=service          List all services
  list-units --state=failed          Show failed units
  cat <unit>                         Print unit file
  show <unit>                        All unit properties
  list-dependencies <unit>           Dependency tree

JOURNAL
  journalctl -u <unit> -f            Follow unit logs
  journalctl -u <unit> -n 100        Last 100 lines
  journalctl -b                      This boot only
  journalctl -p err                  Errors and above
  journalctl --since "1 hour ago"    Time-based filter
  journalctl --disk-usage            Journal size
  journalctl --vacuum-size=500M      Trim journal

BOOT ANALYSIS
  systemd-analyze blame              Per-unit boot times
  systemd-analyze critical-chain     Boot critical path
  systemd-analyze security <unit>    Security score

OVERRIDES
  systemctl edit <unit>              Drop-in override
  systemctl edit --full <unit>       Full copy override
  /etc/systemd/system/<unit>.d/      Drop-in directory

TIMERS
  list-timers                        Active timers + next run
  OnCalendar=daily / hourly          Common schedules
  Persistent=true                    Catch up missed runs
  systemd-analyze calendar <expr>    Test calendar expression

SPECIFIERS
  %n  unit name    %i  instance    %H  hostname
  %u  user         %h  home dir    %t  runtime dir

KEY UNIT SECTIONS
  [Unit]    Description, After, Requires, Wants, OnFailure
  [Service] Type, ExecStart, Restart, User, Environment
  [Install] WantedBy, RequiredBy, Alias
  [Timer]   OnCalendar, OnBootSec, Persistent
```

---

*See `man systemd`, `man systemctl`, `man journalctl`, `man systemd.service`, `man systemd.timer` or https://systemd.io for full documentation.*
