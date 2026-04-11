# SSH Cheatsheet

> A comprehensive reference for `ssh` and the OpenSSH toolset on Linux — covering connections, keys, tunnelling, config, hardening, and more.

---

## Table of Contents

- [Basic Syntax](#basic-syntax)
- [Connecting to a Host](#connecting-to-a-host)
- [SSH Keys](#ssh-keys)
- [SSH Agent](#ssh-agent)
- [The SSH Config File](#the-ssh-config-file)
- [File Transfer — scp & rsync](#file-transfer--scp--rsync)
- [SFTP](#sftp)
- [Port Forwarding & Tunnelling](#port-forwarding--tunnelling)
- [Jump Hosts & ProxyJump](#jump-hosts--proxyjump)
- [X11 Forwarding](#x11-forwarding)
- [Multiplexing (ControlMaster)](#multiplexing-controlmaster)
- [Remote Commands & Scripting](#remote-commands--scripting)
- [Server Configuration (sshd)](#server-configuration-sshd)
- [Hardening](#hardening)
- [Troubleshooting](#troubleshooting)
- [Quick Reference Card](#quick-reference-card)

---

## Basic Syntax

```
ssh [options] [user@]host [command]
```

```bash
# Connect as current user
ssh host

# Connect as a specific user
ssh user@host

# Connect to a non-standard port
ssh user@host -p 2222

# Run a command remotely and exit
ssh user@host "uptime"
```

---

## Connecting to a Host

| Option | Description |
|---|---|
| `-p port` | Connect to non-default port (default: 22) |
| `-i file` | Identity (private key) file to use |
| `-l user` | Login name (alternative to `user@host`) |
| `-v` | Verbose — show debug output |
| `-vv` / `-vvv` | More verbose |
| `-q` | Quiet — suppress warnings |
| `-C` | Enable compression |
| `-X` | Enable X11 forwarding |
| `-A` | Forward SSH agent |
| `-t` | Force pseudo-terminal allocation |
| `-T` | Disable pseudo-terminal allocation |
| `-4` | Force IPv4 |
| `-6` | Force IPv6 |
| `-o option=value` | Set a config option inline |
| `-F file` | Use alternative config file |
| `-E file` | Log debug output to a file |

```bash
# Connect with a specific key
ssh -i ~/.ssh/id_rsa user@host

# Connect with compression (useful on slow links)
ssh -C user@host

# Force password authentication
ssh -o PubkeyAuthentication=no user@host

# Accept any host key (insecure — testing only)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@host

# Connect with specific cipher
ssh -c aes256-gcm@openssh.com user@host

# Verbose connection debug (diagnose auth failures)
ssh -vvv user@host
```

---

## SSH Keys

### Generating Keys

```bash
# Generate Ed25519 key (recommended — modern, fast, secure)
ssh-keygen -t ed25519 -C "your_comment"

# Generate RSA key (4096-bit — for legacy compatibility)
ssh-keygen -t rsa -b 4096 -C "your_comment"

# Generate ECDSA key
ssh-keygen -t ecdsa -b 521 -C "your_comment"

# Generate key with a custom filename and passphrase prompt
ssh-keygen -t ed25519 -f ~/.ssh/my_key -C "server-name"

# Generate key non-interactively (empty passphrase)
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -N "" -C "deploy"
```

### Key File Locations

| File | Description |
|---|---|
| `~/.ssh/id_ed25519` | Default Ed25519 private key |
| `~/.ssh/id_ed25519.pub` | Default Ed25519 public key |
| `~/.ssh/id_rsa` | Default RSA private key |
| `~/.ssh/id_rsa.pub` | Default RSA public key |
| `~/.ssh/authorized_keys` | Public keys allowed to log in |
| `~/.ssh/known_hosts` | Fingerprints of known remote hosts |
| `~/.ssh/config` | Per-user SSH configuration |
| `/etc/ssh/sshd_config` | Server daemon configuration |
| `/etc/ssh/ssh_config` | System-wide client configuration |

### Copying Keys to a Server

```bash
# Copy public key to remote host (recommended)
ssh-copy-id user@host

# Copy a specific key
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# Copy to non-standard port
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 user@host

# Manual method (if ssh-copy-id is unavailable)
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Key Management

```bash
# View public key fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# View fingerprint in SHA256 format
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E sha256

# View key fingerprint as randomart
ssh-keygen -lv -f ~/.ssh/id_ed25519.pub

# Change passphrase on existing key
ssh-keygen -p -f ~/.ssh/id_rsa

# Remove a host from known_hosts
ssh-keygen -R hostname

# Remove a specific host IP from known_hosts
ssh-keygen -R 192.168.1.1

# Convert OpenSSH key to PEM format
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa

# Show public key from a private key
ssh-keygen -y -f ~/.ssh/id_rsa

# Check if an agent has a key loaded
ssh-add -l
```

### authorized_keys Options

Options can be prepended to a public key entry in `~/.ssh/authorized_keys`:

```
# Restrict to a specific command
command="/usr/bin/rsync --server" ssh-ed25519 AAAA... comment

# Restrict source IP
from="192.168.1.0/24" ssh-ed25519 AAAA... comment

# No port forwarding
no-port-forwarding ssh-ed25519 AAAA... comment

# No agent forwarding, no X11
no-agent-forwarding,no-x11-forwarding ssh-ed25519 AAAA... comment

# Force a command and restrict from specific IP
from="10.0.0.1",command="/usr/bin/backup.sh",no-pty ssh-ed25519 AAAA... backup-key
```

---

## SSH Agent

The SSH agent holds decrypted private keys in memory so you don't re-enter passphrases.

```bash
# Start the agent (outputs eval-able shell commands)
eval "$(ssh-agent -s)"

# Add default key (~/.ssh/id_*)
ssh-add

# Add a specific key
ssh-add ~/.ssh/id_ed25519

# Add key and cache passphrase for 1 hour
ssh-add -t 3600 ~/.ssh/id_ed25519

# List all keys loaded in the agent
ssh-add -l

# List keys with full public key
ssh-add -L

# Remove a specific key from agent
ssh-add -d ~/.ssh/id_ed25519

# Remove all keys from agent
ssh-add -D

# Lock agent with a password
ssh-add -x

# Unlock agent
ssh-add -X

# Automatically start agent in ~/.bashrc / ~/.bash_profile
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
fi
```

---

## The SSH Config File

`~/.ssh/config` lets you define per-host settings, aliases, and defaults.

```
# Permissions must be correct
chmod 600 ~/.ssh/config
```

### Config File Format

```sshconfig
# Global defaults (apply to all hosts)
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentityFile ~/.ssh/id_ed25519

# A simple alias
Host myserver
    HostName 203.0.113.10
    User deploy
    Port 2222
    IdentityFile ~/.ssh/deploy_key

# Jump through a bastion host
Host internal
    HostName 10.0.0.50
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jumpuser
    IdentityFile ~/.ssh/bastion_key

# Wildcard for all dev servers
Host dev-*
    User ubuntu
    IdentityFile ~/.ssh/dev_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

# SOCKS proxy via remote host
Host proxy
    HostName proxy.example.com
    User user
    DynamicForward 1080

# Local port forwarding shortcut
Host db-tunnel
    HostName bastion.example.com
    User user
    LocalForward 5433 db.internal:5432
    ServerAliveInterval 30
```

### Useful Config Options

| Option | Description |
|---|---|
| `HostName` | Actual hostname or IP |
| `User` | Login username |
| `Port` | Remote port |
| `IdentityFile` | Path to private key |
| `ProxyJump` | Jump host(s) |
| `ForwardAgent` | Forward SSH agent |
| `ServerAliveInterval` | Send keepalive every N seconds |
| `ServerAliveCountMax` | Max keepalives before disconnect |
| `AddKeysToAgent` | Auto-add key to agent on use |
| `StrictHostKeyChecking` | `yes`, `no`, or `accept-new` |
| `UserKnownHostsFile` | Path to known_hosts file |
| `Compression` | Enable compression (`yes`/`no`) |
| `LogLevel` | `QUIET`, `INFO`, `VERBOSE`, `DEBUG` |
| `ConnectTimeout` | Timeout in seconds |
| `ControlMaster` | Enable multiplexing |
| `ControlPath` | Socket file for multiplexing |
| `ControlPersist` | Keep master alive after disconnect |
| `DynamicForward` | SOCKS proxy port |
| `LocalForward` | Local port forwarding |
| `RemoteForward` | Remote port forwarding |
| `RequestTTY` | Force TTY: `yes`, `no`, `force`, `auto` |
| `SendEnv` | Send environment variables |
| `SetEnv` | Set environment variables |

---

## File Transfer — scp & rsync

### scp

```bash
# Copy local file to remote
scp file.txt user@host:/remote/path/

# Copy remote file to local
scp user@host:/remote/file.txt /local/path/

# Copy directory recursively
scp -r /local/dir user@host:/remote/dir

# Copy with specific port
scp -P 2222 file.txt user@host:/path/

# Copy with specific key
scp -i ~/.ssh/id_ed25519 file.txt user@host:/path/

# Copy between two remote hosts (via local machine)
scp user1@host1:/path/file user2@host2:/path/

# Preserve timestamps and permissions
scp -p file.txt user@host:/path/

# Compress during transfer
scp -C large_file.tar user@host:/path/

# Limit bandwidth (Kbits/sec)
scp -l 1000 file.txt user@host:/path/
```

### rsync over SSH

```bash
# Sync local dir to remote (recommended over scp for directories)
rsync -avz /local/dir/ user@host:/remote/dir/

# Sync remote dir to local
rsync -avz user@host:/remote/dir/ /local/dir/

# Dry run — show what would be transferred
rsync -avzn /local/dir/ user@host:/remote/dir/

# Delete files on dest that don't exist on source
rsync -avz --delete /local/dir/ user@host:/remote/dir/

# Use a specific SSH port
rsync -avz -e "ssh -p 2222" /local/dir/ user@host:/remote/dir/

# Use a specific key
rsync -avz -e "ssh -i ~/.ssh/deploy_key" /local/dir/ user@host:/remote/dir/

# Exclude files/directories
rsync -avz --exclude ".git" --exclude "node_modules" /local/ user@host:/remote/

# Show progress per file
rsync -avz --progress /local/dir/ user@host:/remote/dir/

# Bandwidth limit (KB/s)
rsync -avz --bwlimit=1000 /local/dir/ user@host:/remote/dir/
```

---

## SFTP

```bash
# Open SFTP session
sftp user@host

# SFTP on non-standard port
sftp -P 2222 user@host

# SFTP with specific key
sftp -i ~/.ssh/id_ed25519 user@host
```

### SFTP Interactive Commands

| Command | Description |
|---|---|
| `ls` | List remote directory |
| `lls` | List local directory |
| `pwd` | Remote working directory |
| `lpwd` | Local working directory |
| `cd path` | Change remote directory |
| `lcd path` | Change local directory |
| `get file` | Download file |
| `get -r dir` | Download directory |
| `put file` | Upload file |
| `put -r dir` | Upload directory |
| `rm file` | Delete remote file |
| `rmdir dir` | Delete remote directory |
| `mkdir dir` | Create remote directory |
| `rename old new` | Rename remote file |
| `chmod mode file` | Change remote permissions |
| `df` | Show remote disk usage |
| `!command` | Run local shell command |
| `bye` / `exit` | Close session |

```bash
# Batch SFTP — non-interactive
sftp user@host <<EOF
cd /remote/path
get important_file.txt
put local_file.txt
bye
EOF
```

---

## Port Forwarding & Tunnelling

### Local Port Forwarding (`-L`)

Forwards a local port to a remote destination through the SSH server.

```
ssh -L [local_addr:]local_port:remote_host:remote_port user@ssh_server
```

```bash
# Access a remote database locally (postgres on remote internal network)
ssh -L 5433:db.internal:5432 user@bastion

# Access a remote web service on localhost:8080
ssh -L 8080:localhost:80 user@webserver

# Bind only on localhost (default; more secure)
ssh -L 127.0.0.1:8080:10.0.0.5:80 user@host

# Keep tunnel open in background
ssh -fNL 5433:db.internal:5432 user@bastion
# -f = background, -N = no command, -L = local forward
```

### Remote Port Forwarding (`-R`)

Forwards a port on the remote server back to a local destination.

```
ssh -R [remote_addr:]remote_port:local_host:local_port user@ssh_server
```

```bash
# Expose local port 3000 as port 8080 on the remote server
ssh -R 8080:localhost:3000 user@remote

# Allow anyone on the remote server to reach your local web server
ssh -R 0.0.0.0:8080:localhost:80 user@remote

# Persistent reverse tunnel in background
ssh -fNR 2222:localhost:22 user@remote
```

### Dynamic Port Forwarding / SOCKS Proxy (`-D`)

Turns SSH into a SOCKS5 proxy for any application.

```bash
# Start SOCKS5 proxy on local port 1080
ssh -D 1080 user@host

# Background SOCKS proxy
ssh -fND 1080 user@host

# Use with curl via SOCKS proxy
curl --socks5 127.0.0.1:1080 https://example.com

# Use with git
GIT_SSH_COMMAND='ssh -o ProxyCommand="nc -x 127.0.0.1:1080 %h %p"' git clone ...
```

### Keeping Tunnels Alive

```bash
# Keep alive — don't exit even with no command
ssh -fNL 5433:db.internal:5432 user@bastion

# Add keepalives to prevent idle timeout
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -fNL 5433:db:5432 user@bastion

# Auto-reconnecting tunnel (with autossh)
autossh -M 20000 -fNL 5433:db.internal:5432 user@bastion
```

---

## Jump Hosts & ProxyJump

ProxyJump connects through one or more intermediate SSH hosts.

```bash
# Jump through a single bastion host
ssh -J jumpuser@bastion target_user@internal_host

# Jump through multiple hosts (chain)
ssh -J user@hop1,user@hop2 user@final_target

# Using config (recommended)
# ~/.ssh/config:
# Host internal
#     ProxyJump bastion

ssh internal

# ProxyCommand (legacy alternative to ProxyJump)
ssh -o ProxyCommand="ssh -W %h:%p jumpuser@bastion" user@internal

# SCP via jump host
scp -J user@bastion file.txt user@internal:/path/

# rsync via jump host
rsync -avz -e "ssh -J user@bastion" /local/ user@internal:/remote/
```

---

## X11 Forwarding

Run graphical applications on a remote server, display locally.

```bash
# Enable X11 forwarding
ssh -X user@host

# Trusted X11 forwarding (faster, less security)
ssh -Y user@host

# Launch a GUI app remotely
ssh -X user@host gedit
ssh -X user@host firefox

# Check DISPLAY is set after connecting
echo $DISPLAY
# Should show something like: localhost:10.0
```

> **Note:** Requires an X server locally (e.g. XQuartz on macOS, VcXsrv on Windows, or built-in on Linux).

---

## Multiplexing (ControlMaster)

Reuse an existing SSH connection for subsequent sessions — eliminates repeated authentication.

```bash
# Manual multiplexing
ssh -M -S /tmp/ssh-control-%r@%h:%p user@host   # open master
ssh -S /tmp/ssh-control-%r@%h:%p user@host       # reuse connection
ssh -S /tmp/ssh-control-%r@%h:%p -O exit user@host  # close master
```

### In `~/.ssh/config` (recommended)

```sshconfig
Host myserver
    HostName 192.168.1.10
    User user
    ControlMaster auto
    ControlPath ~/.ssh/control-%C
    ControlPersist 10m
```

```bash
# Check status of multiplexed connection
ssh -O check myserver

# Stop the master connection
ssh -O stop myserver

# Exit the master connection
ssh -O exit myserver
```

---

## Remote Commands & Scripting

```bash
# Run a single command
ssh user@host "df -h"

# Run multiple commands
ssh user@host "uname -a; uptime; free -m"

# Run a local script on a remote host
ssh user@host 'bash -s' < local_script.sh

# Pass variables to a remote script
VAR="hello" ssh user@host 'echo $VAR'

# Run command as root via sudo
ssh user@host "sudo systemctl restart nginx"

# Force a TTY for interactive commands
ssh -t user@host "sudo bash"

# Pipe local data to a remote command
cat local_file.txt | ssh user@host "cat > /remote/file.txt"

# Pipe remote command output locally
ssh user@host "cat /remote/file.txt" > local_copy.txt

# Run a here-document on the remote host
ssh user@host << 'EOF'
  cd /var/www
  git pull
  systemctl reload nginx
EOF

# Parallel commands on multiple hosts
for host in host1 host2 host3; do
  ssh user@$host "uptime" &
done
wait
```

---

## Server Configuration (sshd)

Main config file: `/etc/ssh/sshd_config`

```bash
# Test config syntax before reloading
sudo sshd -t

# Reload sshd (apply changes without dropping connections)
sudo systemctl reload sshd

# Restart sshd
sudo systemctl restart sshd

# Check sshd status
sudo systemctl status sshd

# View sshd logs
sudo journalctl -u sshd -f
sudo tail -f /var/log/auth.log
```

### Common sshd_config Directives

```sshconfig
# Port to listen on
Port 22

# Listen on specific address
ListenAddress 0.0.0.0

# Protocol version (2 only)
Protocol 2

# Host key files
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

# Authentication
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Login restrictions
AllowUsers alice bob deploy
AllowGroups ssh-users
DenyUsers baduser
MaxAuthTries 3
MaxSessions 10
LoginGraceTime 30

# Keepalive
ClientAliveInterval 300
ClientAliveCountMax 2

# Forwarding
AllowTcpForwarding yes
X11Forwarding no
AllowAgentForwarding yes

# Logging
SyslogFacility AUTH
LogLevel INFO

# SFTP subsystem
Subsystem sftp /usr/lib/openssh/sftp-server
```

---

## Hardening

Best-practice hardening settings for `/etc/ssh/sshd_config`:

```sshconfig
# Use a non-standard port (security through obscurity — minor benefit)
Port 2222

# Disable root login entirely
PermitRootLogin no

# Disable password authentication — keys only
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no

# Disable empty passwords
PermitEmptyPasswords no

# Disable legacy auth methods
KerberosAuthentication no
GSSAPIAuthentication no
HostbasedAuthentication no

# Whitelist allowed users
AllowUsers alice bob

# Reduce attack window
MaxAuthTries 3
LoginGraceTime 20
MaxStartups 10:30:60

# Disable unused features
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no

# Use strong algorithms only
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Preferred host key type
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256

# Logging
LogLevel VERBOSE

# Banner (legal warning)
Banner /etc/ssh/banner.txt
```

### Restrict a User to SFTP Only

```sshconfig
Match User sftponly
    ChrootDirectory /home/sftponly
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

### Fail2ban Integration

```bash
# Install fail2ban
sudo apt install fail2ban

# Enable SSH jail — /etc/fail2ban/jail.local
[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 3600
findtime = 600

sudo systemctl enable --now fail2ban
```

---

## Troubleshooting

```bash
# Verbose connection debug (most useful first step)
ssh -vvv user@host

# Test sshd config syntax
sudo sshd -t

# Check sshd is running and what port it's on
sudo ss -tlnp | grep sshd
sudo netstat -tlnp | grep sshd

# Check firewall allows SSH
sudo ufw status
sudo iptables -L INPUT -n | grep 22

# Check correct permissions (common cause of auth failures)
ls -la ~/.ssh/
# Should be:
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/config

# View live authentication log
sudo tail -f /var/log/auth.log
sudo journalctl -u sshd -f

# Check which key is being offered
ssh -vvv user@host 2>&1 | grep "Offering"

# Test a specific key
ssh -i ~/.ssh/specific_key -o IdentitiesOnly=yes user@host

# Clear a stale known_hosts entry (host key changed)
ssh-keygen -R hostname
ssh-keygen -R 192.168.1.1

# Check server host keys
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

# Verify authorized_keys is correct
cat ~/.ssh/authorized_keys
# Ensure no line breaks in the key, correct format

# Check SELinux / AppArmor context issues
ls -Z ~/.ssh/
restorecon -Rv ~/.ssh/
```

### Common Errors & Fixes

| Error | Likely Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Wrong key / bad permissions | Check `~/.ssh` permissions; use `-vvv` |
| `Connection refused` | sshd not running / wrong port | Check `systemctl status sshd` |
| `Host key verification failed` | Server key changed | Run `ssh-keygen -R hostname` |
| `Too many authentication failures` | Too many keys offered | Use `-o IdentitiesOnly=yes -i key` |
| `Connection timed out` | Firewall blocking | Check firewall; confirm port |
| `WARNING: UNPROTECTED PRIVATE KEY` | Permissions too open | `chmod 600 ~/.ssh/id_*` |
| `No route to host` | Network issue | Check routing; ping host |
| `Broken pipe` | Idle timeout | Add `ServerAliveInterval 60` |
| `channel 0: open failed` | Port forwarding blocked | Check `AllowTcpForwarding` in sshd |

---

## Quick Reference Card

```
ssh [options] user@host [command]

CONNECT                         KEYS
  -p port    custom port          ssh-keygen -t ed25519    generate
  -i file    identity file        ssh-copy-id user@host    deploy key
  -l user    login name           ssh-add ~/.ssh/id_ed25519 add to agent
  -v/-vvv    verbose/debug        ssh-add -l               list agent keys
  -C         compress             ssh-keygen -R host       remove known host
  -A         agent forward
  -t         force TTY          FILE TRANSFER
  -q         quiet                scp file user@host:/path/   send file
  -4 / -6    IPv4 / IPv6          scp user@host:/file .       get file
                                  rsync -avz src/ user@host:dst/

TUNNELS                         SERVER
  -L lport:host:rport  local fwd   sudo sshd -t         test config
  -R rport:host:lport  remote fwd  systemctl reload sshd apply changes
  -D port              SOCKS proxy journalctl -u sshd -f view logs
  -fN                  background

JUMP HOSTS                      HARDENING
  -J user@bastion      jump host   PermitRootLogin no
  ssh -J h1,h2 target  chain       PasswordAuthentication no
                                   MaxAuthTries 3
CONFIG (~/.ssh/config)            AllowUsers alice bob
  Host alias                       X11Forwarding no
    HostName ip                    Ciphers chacha20-poly1305@...
    User username
    Port 22
    IdentityFile ~/.ssh/key
    ProxyJump bastion
    ServerAliveInterval 60
    ControlMaster auto
    ControlPath ~/.ssh/ctrl-%C
    ControlPersist 10m
```

---

> **Further Reading:**
> - `man ssh` / `man sshd` / `man ssh_config` / `man sshd_config`
> - `man ssh-keygen` / `man ssh-agent` / `man ssh-add`
> - [OpenSSH manual pages](https://www.openssh.com/manual.html)
> - [Mozilla SSH hardening guide](https://infosec.mozilla.org/guidelines/openssh)
