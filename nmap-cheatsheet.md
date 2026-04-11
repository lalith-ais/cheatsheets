# Nmap Cheatsheet

> A comprehensive reference for `nmap` (Network Mapper) — the industry-standard open-source tool for network discovery, port scanning, service detection, and security auditing.

> ⚠️ **Legal Warning:** Only scan networks and systems you own or have explicit written permission to test. Unauthorised scanning may be illegal in your jurisdiction.

---

## Table of Contents

- [Basic Syntax](#basic-syntax)
- [Target Specification](#target-specification)
- [Scan Types](#scan-types)
- [Port Specification](#port-specification)
- [Service & Version Detection](#service--version-detection)
- [OS Detection](#os-detection)
- [Timing & Performance](#timing--performance)
- [Firewall & IDS Evasion](#firewall--ids-evasion)
- [Output Formats](#output-formats)
- [Nmap Scripting Engine (NSE)](#nmap-scripting-engine-nse)
- [Common NSE Scripts](#common-nse-scripts)
- [IPv6 Scanning](#ipv6-scanning)
- [Practical Examples](#practical-examples)
- [Quick Reference Card](#quick-reference-card)

---

## Basic Syntax

```
nmap [scan type] [options] [target]
```

```bash
# Scan a single host
nmap 192.168.1.1

# Scan a domain
nmap example.com

# Scan with version and OS detection (common starting point)
nmap -sV -O 192.168.1.1

# Aggressive scan (OS, version, scripts, traceroute)
nmap -A 192.168.1.1

# Run as root for best results (required for SYN scans)
sudo nmap -sS 192.168.1.1
```

---

## Target Specification

| Format | Example | Description |
|---|---|---|
| Single host | `192.168.1.1` | One IP address |
| Hostname | `example.com` | Resolved via DNS |
| CIDR range | `192.168.1.0/24` | 256 hosts |
| Range | `192.168.1.1-50` | 50 hosts |
| Wildcard | `192.168.1.*` | Same as `/24` |
| List | `192.168.1.1,5,10` | Specific IPs |
| File | `-iL hosts.txt` | One target per line |
| Random | `-iR 100` | 100 random hosts |
| Exclude | `--exclude 192.168.1.5` | Skip a host |
| Exclude file | `--excludefile skip.txt` | Skip from file |

```bash
# Scan entire subnet
nmap 192.168.1.0/24

# Scan a range
nmap 10.0.0.1-254

# Scan from a file
nmap -iL targets.txt

# Scan multiple specific hosts
nmap 192.168.1.1 192.168.1.10 192.168.1.20

# Exclude hosts from a range
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.254

# Random internet hosts (use responsibly)
nmap -iR 10 --open -p 80
```

---

## Scan Types

### Host Discovery

| Flag | Description |
|---|---|
| `-sn` | Ping scan only — no port scan (host discovery) |
| `-Pn` | Skip host discovery — treat all hosts as online |
| `-PS [ports]` | TCP SYN ping to specified ports |
| `-PA [ports]` | TCP ACK ping |
| `-PU [ports]` | UDP ping |
| `-PE` | ICMP echo ping |
| `-PP` | ICMP timestamp ping |
| `-PM` | ICMP address mask ping |
| `-PR` | ARP ping (local network only) |
| `-n` | Never resolve DNS |
| `-R` | Always resolve DNS |
| `--dns-servers` | Use specified DNS servers |

```bash
# Discover live hosts without port scanning
nmap -sn 192.168.1.0/24

# Scan all hosts even if they don't respond to ping
nmap -Pn 192.168.1.0/24

# ARP scan local network (very fast, reliable on LAN)
sudo nmap -PR -sn 192.168.1.0/24

# Disable DNS resolution (faster)
nmap -n 192.168.1.0/24
```

### Port Scan Techniques

| Flag | Name | Description | Requires Root |
|---|---|---|---|
| `-sS` | TCP SYN scan | Half-open; fast and stealthy | ✅ |
| `-sT` | TCP Connect scan | Full 3-way handshake; no root needed | ❌ |
| `-sU` | UDP scan | Slower; finds UDP services | ✅ |
| `-sA` | TCP ACK scan | Map firewall rules | ✅ |
| `-sW` | TCP Window scan | Like ACK; detects open vs closed | ✅ |
| `-sM` | TCP Maimon scan | FIN/ACK probe | ✅ |
| `-sN` | TCP Null scan | No flags set; evades some firewalls | ✅ |
| `-sF` | FIN scan | FIN flag only | ✅ |
| `-sX` | Xmas scan | FIN + PSH + URG flags | ✅ |
| `-sI zombie:port` | Idle/Zombie scan | Blind scan via third party | ✅ |
| `-sY` | SCTP INIT scan | SCTP protocol scan | ✅ |
| `-sO` | IP protocol scan | Which IP protocols are supported | ✅ |

```bash
# SYN scan (default with root — recommended)
sudo nmap -sS 192.168.1.1

# Connect scan (no root required)
nmap -sT 192.168.1.1

# UDP scan (slow — common UDP services)
sudo nmap -sU -p 53,67,68,123,161,500 192.168.1.1

# Combined TCP SYN + UDP scan
sudo nmap -sS -sU 192.168.1.1

# Xmas scan (stealthy against some firewalls)
sudo nmap -sX 192.168.1.1

# ACK scan to detect firewall rules
sudo nmap -sA 192.168.1.1

# Idle scan (uses zombie host)
sudo nmap -sI zombie_host:80 target_host
```

---

## Port Specification

| Flag | Description |
|---|---|
| `-p 80` | Scan port 80 only |
| `-p 80,443,8080` | Scan specific ports |
| `-p 1-1024` | Scan port range |
| `-p 1-65535` | Scan all ports |
| `-p-` | Scan all 65535 ports (shorthand) |
| `-p U:53,T:80` | Scan UDP 53 and TCP 80 |
| `--top-ports N` | Scan N most common ports |
| `-F` | Fast scan — top 100 ports |
| `-r` | Scan ports in sequential order |
| `--port-ratio 0.1` | Scan ports with ratio > 0.1 |

```bash
# Scan all ports
nmap -p- 192.168.1.1

# Scan common ports
nmap --top-ports 1000 192.168.1.1

# Fast scan (top 100)
nmap -F 192.168.1.1

# Scan specific ports
nmap -p 22,80,443,3306,5432,6379,8080,8443 192.168.1.1

# Scan port range
nmap -p 1-10000 192.168.1.1

# Scan UDP and TCP on specific ports
sudo nmap -p U:53,161 -p T:80,443 192.168.1.1
```

### Port States

| State | Meaning |
|---|---|
| `open` | Service is actively accepting connections |
| `closed` | Port is accessible but no service is listening |
| `filtered` | Firewall or filter blocking — nmap can't determine state |
| `unfiltered` | Port accessible but nmap can't determine open/closed |
| `open\|filtered` | Can't tell if open or filtered (UDP, some scans) |
| `closed\|filtered` | Can't tell if closed or filtered |

---

## Service & Version Detection

| Flag | Description |
|---|---|
| `-sV` | Detect service version |
| `--version-intensity 0-9` | How aggressively to probe (default: 7) |
| `--version-light` | Less intensive version scan (intensity 2) |
| `--version-all` | Try all probes (intensity 9) |
| `--version-trace` | Detailed debugging of version detection |

```bash
# Basic version detection
nmap -sV 192.168.1.1

# Light version scan (faster)
nmap -sV --version-light 192.168.1.1

# Most aggressive version detection
nmap -sV --version-all 192.168.1.1

# Version + scripts (most thorough)
nmap -sV --script=default 192.168.1.1
```

---

## OS Detection

| Flag | Description |
|---|---|
| `-O` | Enable OS detection |
| `--osscan-limit` | Only try OS detection on promising targets |
| `--osscan-guess` | Guess OS more aggressively |
| `--max-os-tries N` | Max attempts at OS detection |

```bash
# Basic OS detection
sudo nmap -O 192.168.1.1

# Aggressive OS guess
sudo nmap -O --osscan-guess 192.168.1.1

# Combined OS + version detection
sudo nmap -O -sV 192.168.1.1

# Full aggressive scan
sudo nmap -A 192.168.1.1
```

> **Note:** OS detection requires at least one open and one closed port to work reliably. `-A` enables OS detection, version detection, script scanning, and traceroute.

---

## Timing & Performance

### Timing Templates

| Flag | Name | Speed | Use Case |
|---|---|---|---|
| `-T0` | Paranoid | Extremely slow | IDS evasion |
| `-T1` | Sneaky | Very slow | IDS evasion |
| `-T2` | Polite | Slow | Reduce bandwidth |
| `-T3` | Normal | Default | Balanced |
| `-T4` | Aggressive | Fast | Fast networks / CTFs |
| `-T5` | Insane | Very fast | Sacrifice accuracy |

```bash
# Default (T3 is implicit)
nmap 192.168.1.1

# Fast aggressive scan
nmap -T4 192.168.1.1

# Stealthy slow scan (IDS evasion)
nmap -T1 192.168.1.1
```

### Fine-Grained Timing

| Flag | Description |
|---|---|
| `--min-rate N` | Send at least N packets/sec |
| `--max-rate N` | Send at most N packets/sec |
| `--min-parallelism N` | Min parallel probes |
| `--max-parallelism N` | Max parallel probes |
| `--min-hostgroup N` | Min hosts scanned in parallel |
| `--max-hostgroup N` | Max hosts scanned in parallel |
| `--host-timeout ms` | Give up on slow hosts after this time |
| `--scan-delay ms` | Delay between probes to a host |
| `--max-scan-delay ms` | Max delay between probes |
| `--min-rtt-timeout ms` | Min round-trip time |
| `--max-rtt-timeout ms` | Max round-trip time |
| `--initial-rtt-timeout ms` | Initial RTT estimate |
| `--max-retries N` | Cap probe retransmissions |

```bash
# Fast scan with rate limit
nmap --min-rate 1000 --max-rate 5000 192.168.1.0/24

# Slow scan to avoid detection
nmap --scan-delay 1s --max-rate 10 192.168.1.1

# Skip hosts taking too long
nmap --host-timeout 30s 192.168.1.0/24

# High parallelism for large network
nmap --min-parallelism 50 --max-parallelism 256 192.168.1.0/24
```

---

## Firewall & IDS Evasion

| Flag | Description |
|---|---|
| `-f` | Fragment packets (8-byte fragments) |
| `-f -f` | 16-byte fragments |
| `--mtu N` | Set custom MTU (multiple of 8) |
| `-D decoy1,decoy2` | Use decoy IP addresses |
| `-D RND:10` | Use 10 random decoys |
| `-S addr` | Spoof source address |
| `-e interface` | Use specific network interface |
| `-g port` | Use given source port |
| `--source-port port` | Alias for `-g` |
| `--data-length N` | Append random data to packets |
| `--ip-options opts` | Set IP options |
| `--ttl value` | Set TTL |
| `--randomize-hosts` | Scan hosts in random order |
| `--spoof-mac addr` | Spoof MAC address |
| `--badsum` | Send packets with bad checksums |
| `--proxies url` | Relay via HTTP/SOCKS4 proxies |

```bash
# Fragment packets to bypass simple firewalls
sudo nmap -f 192.168.1.1

# Use decoy IPs to obscure the real scanner
sudo nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.1

# Use common source port (looks like legit traffic)
sudo nmap --source-port 53 192.168.1.1

# Spoof MAC address
sudo nmap --spoof-mac 00:11:22:33:44:55 192.168.1.1

# Randomise scan order to avoid pattern detection
nmap --randomize-hosts 192.168.1.0/24

# Append random data to packets
sudo nmap --data-length 25 192.168.1.1

# Slow timing + fragmentation + decoys
sudo nmap -T1 -f -D RND:5 192.168.1.1
```

---

## Output Formats

| Flag | Format | Description |
|---|---|---|
| `-oN file` | Normal | Human-readable text |
| `-oX file` | XML | Machine-parseable |
| `-oG file` | Grepable | One line per host |
| `-oS file` | Script kiddie | l33t speak (novelty) |
| `-oA basename` | All formats | Saves `.nmap`, `.xml`, `.gnmap` |
| `-v` | | Increase verbosity |
| `-vv` | | Very verbose |
| `-d` | | Increase debug level |
| `-dd` | | More debug |
| `--reason` | | Show why port is in that state |
| `--open` | | Only show open ports |
| `--packet-trace` | | Show all packets sent/received |
| `--iflist` | | List interfaces and routes |
| `--append-output` | | Append to output files |
| `--resume file` | | Resume aborted scan |

```bash
# Save in all formats
nmap -oA scan_results 192.168.1.0/24

# Normal output to file
nmap -oN output.txt 192.168.1.1

# XML output (for tools like Metasploit)
nmap -oX output.xml 192.168.1.1

# Grepable output
nmap -oG output.gnmap 192.168.1.1

# Show only open ports
nmap --open 192.168.1.0/24

# Show reason for each port state
nmap --reason 192.168.1.1

# Resume an interrupted scan
nmap --resume output.gnmap

# Parse grepable output for open ports
grep "open" output.gnmap | awk '{print $2}'

# Extract open ports from XML with grep
grep "portid" output.xml | grep "open"
```

---

## Nmap Scripting Engine (NSE)

NSE allows nmap to run Lua scripts for advanced detection, exploitation, and information gathering.

### Script Categories

| Category | Description |
|---|---|
| `auth` | Authentication bypass / credential checks |
| `broadcast` | Host discovery via broadcast |
| `brute` | Brute-force credential attacks |
| `default` | Run with `-sC` or `-A`; safe and useful |
| `discovery` | Service / host information gathering |
| `dos` | Denial-of-service tests (use with care) |
| `exploit` | Exploit vulnerabilities |
| `external` | Uses external resources (DNS, Whois) |
| `fuzzer` | Send unexpected / malformed data |
| `intrusive` | Likely to crash or affect target systems |
| `malware` | Detect malware / backdoors |
| `safe` | Unlikely to crash or harm target |
| `version` | Supplement version detection |
| `vuln` | Detect known vulnerabilities |

### Running Scripts

```bash
# Run default scripts (equivalent to --script=default)
nmap -sC 192.168.1.1

# Run a specific script
nmap --script=http-title 192.168.1.1

# Run multiple scripts
nmap --script=http-title,http-headers 192.168.1.1

# Run all scripts in a category
nmap --script=vuln 192.168.1.1

# Run all safe scripts
nmap --script=safe 192.168.1.1

# Run scripts matching a pattern
nmap --script="http-*" 192.168.1.1

# Pass arguments to a script
nmap --script=http-brute --script-args userdb=users.txt,passdb=pass.txt 192.168.1.1

# Run a script with verbose output
nmap --script=smb-vuln-ms17-010 -v 192.168.1.1

# Update the NSE script database
sudo nmap --script-updatedb

# List all available scripts
ls /usr/share/nmap/scripts/

# Get help on a script
nmap --script-help http-title
```

---

## Common NSE Scripts

### HTTP & Web

```bash
# Get HTTP page titles
nmap --script=http-title -p 80,443,8080 192.168.1.0/24

# Enumerate HTTP headers
nmap --script=http-headers 192.168.1.1

# Detect WAF
nmap --script=http-waf-detect 192.168.1.1

# Brute-force HTTP basic auth
nmap --script=http-brute 192.168.1.1

# Find interesting files/dirs
nmap --script=http-enum 192.168.1.1

# Check for open HTTP proxies
nmap --script=http-open-proxy 192.168.1.1

# Detect common web vulnerabilities
nmap --script="http-vuln-*" 192.168.1.1

# SQL injection detection
nmap --script=http-sql-injection 192.168.1.1

# Shellshock detection
nmap --script=http-shellshock 192.168.1.1

# Check CORS headers
nmap --script=http-cors 192.168.1.1
```

### SMB / Windows

```bash
# Enumerate SMB shares
nmap --script=smb-enum-shares 192.168.1.1

# Enumerate SMB users
nmap --script=smb-enum-users 192.168.1.1

# Check for EternalBlue (MS17-010)
nmap --script=smb-vuln-ms17-010 -p 445 192.168.1.1

# Check for MS08-067
nmap --script=smb-vuln-ms08-067 -p 445 192.168.1.1

# SMB OS discovery
nmap --script=smb-os-discovery -p 445 192.168.1.1

# Brute-force SMB credentials
nmap --script=smb-brute -p 445 192.168.1.1

# Check for SMB signing
nmap --script=smb-security-mode -p 445 192.168.1.1
```

### SSH & Remote Access

```bash
# Get SSH host key
nmap --script=ssh-hostkey -p 22 192.168.1.1

# Detect supported SSH auth methods
nmap --script=ssh-auth-methods -p 22 192.168.1.1

# Brute-force SSH
nmap --script=ssh-brute -p 22 192.168.1.1

# Check for default credentials
nmap --script=ssh-default-auth-methods 192.168.1.1
```

### DNS

```bash
# Enumerate DNS records
nmap --script=dns-brute example.com

# Zone transfer attempt
nmap --script=dns-zone-transfer --script-args dns-zone-transfer.domain=example.com -p 53 ns.example.com

# Reverse DNS sweep
nmap -sn --script=dns-brute 192.168.1.0/24
```

### Databases

```bash
# MySQL info
nmap --script=mysql-info -p 3306 192.168.1.1

# MySQL brute-force
nmap --script=mysql-brute -p 3306 192.168.1.1

# MySQL databases
nmap --script=mysql-databases -p 3306 192.168.1.1

# PostgreSQL brute-force
nmap --script=pgsql-brute -p 5432 192.168.1.1

# MongoDB info
nmap --script=mongodb-info -p 27017 192.168.1.1

# Redis info
nmap --script=redis-info -p 6379 192.168.1.1

# MS SQL info
nmap --script=ms-sql-info -p 1433 192.168.1.1
```

### Vulnerability Detection

```bash
# Run all vuln scripts
nmap --script=vuln 192.168.1.1

# Heartbleed (OpenSSL)
nmap --script=ssl-heartbleed -p 443 192.168.1.1

# POODLE (SSL v3)
nmap --script=ssl-poodle -p 443 192.168.1.1

# Shellshock
nmap --script=http-shellshock 192.168.1.1

# Slowloris DoS check
nmap --script=http-slowloris-check 192.168.1.1

# SSL/TLS certificate info
nmap --script=ssl-cert -p 443 192.168.1.1

# SSL/TLS supported ciphers
nmap --script=ssl-enum-ciphers -p 443 192.168.1.1
```

---

## IPv6 Scanning

```bash
# Enable IPv6 scanning
nmap -6 ::1

# Scan an IPv6 address
nmap -6 2001:db8::1

# Scan IPv6 with SYN scan
sudo nmap -6 -sS 2001:db8::1

# IPv6 host discovery
nmap -6 -sn fe80::1%eth0

# Version + script detection on IPv6
nmap -6 -sV -sC 2001:db8::1
```

---

## Practical Examples

### Network Reconnaissance

```bash
# Quick overview of live hosts on LAN
sudo nmap -sn 192.168.1.0/24

# Fast scan of entire subnet — top 100 ports
nmap -F -T4 192.168.1.0/24

# Full port scan + version on a single host
sudo nmap -p- -sV -T4 192.168.1.1

# Comprehensive single host audit
sudo nmap -A -p- -T4 192.168.1.1

# Save results while scanning
sudo nmap -A 192.168.1.0/24 -oA network_scan
```

### Service-Specific Scans

```bash
# Find all web servers on a subnet
nmap -p 80,443,8080,8443 --open 192.168.1.0/24

# Find all SSH servers
nmap -p 22 --open 192.168.1.0/24

# Find all RDP servers
nmap -p 3389 --open 192.168.1.0/24

# Find all DNS servers
sudo nmap -sU -p 53 --open 192.168.1.0/24

# Find all SMB/Windows hosts
nmap -p 445 --open 192.168.1.0/24

# Find database servers
nmap -p 3306,5432,1433,1521,27017,6379 --open 192.168.1.0/24
```

### Security Auditing

```bash
# Full vulnerability scan
sudo nmap -sV --script=vuln 192.168.1.1

# Web application audit
sudo nmap -sV -p 80,443 --script="http-*" 192.168.1.1

# SSL/TLS audit
nmap --script=ssl-enum-ciphers,ssl-cert,ssl-heartbleed -p 443 192.168.1.1

# SMB vulnerability check (EternalBlue etc.)
sudo nmap -sV --script=smb-vuln-* -p 445 192.168.1.0/24

# Check for default credentials on common services
sudo nmap --script=*-default-accounts 192.168.1.0/24

# Firewall ACL mapping
sudo nmap -sA -p 1-1024 192.168.1.1
```

### CTF & Lab Usage

```bash
# Quick all-ports + version scan (CTF starting point)
sudo nmap -sV -p- -T4 target_ip

# Aggressive all-in-one
sudo nmap -A -p- target_ip

# All ports then targeted script scan
sudo nmap -p- --min-rate=5000 target_ip -oG ports.txt
ports=$(grep open ports.txt | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//')
sudo nmap -sV -sC -p $ports target_ip

# UDP top 20 ports
sudo nmap -sU --top-ports 20 target_ip
```

### Parsing & Scripting

```bash
# List only IPs with open port 80
nmap -p 80 --open -oG - 192.168.1.0/24 | grep "/open" | awk '{print $2}'

# Get all open ports from grepable output
grep "Ports:" scan.gnmap | tr ',' '\n' | grep "open" | awk -F/ '{print $1}' | sort -n | uniq

# Convert XML output to HTML report
xsltproc /usr/share/nmap/nmap.xsl output.xml -o report.html

# Parse XML with Python
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('output.xml')
for host in tree.findall('.//host'):
    ip = host.find('.//address').get('addr')
    for port in host.findall('.//port'):
        if port.find('state').get('state') == 'open':
            print(f\"{ip}:{port.get('portid')}\")
"
```

---

## Quick Reference Card

```
nmap [type] [options] target

TARGET                          PORT SELECTION
  192.168.1.1                     -p 80,443       specific ports
  192.168.1.0/24   subnet         -p 1-1024       range
  192.168.1.1-50   range          -p-             all 65535
  -iL hosts.txt    from file      -F              top 100
  --exclude ip     skip host      --top-ports N   top N
  -Pn              skip ping      -r              sequential

SCAN TYPES                      DETECTION
  -sS  SYN (default, root)        -sV  service version
  -sT  Connect (no root)          -O   OS detection
  -sU  UDP                        -A   all: OS+version+scripts+trace
  -sA  ACK (firewall map)         -sC  default scripts
  -sN  Null   -sF FIN  -sX Xmas  --script=name
  -sn  ping only (no ports)

TIMING                          OUTPUT
  -T0  paranoid  -T3  normal      -oN  normal text
  -T1  sneaky    -T4  aggressive  -oX  XML
  -T2  polite    -T5  insane      -oG  grepable
  --min-rate N                    -oA  all formats
  --max-rate N                    -v / -vv  verbose
                                  --open    open only
EVASION                           --reason  show reason
  -f          fragment packets
  -D ip,ip    decoys            NSE SCRIPTS
  -S addr     spoof source        --script=vuln
  --source-port 53                --script=default
  --randomize-hosts               --script=safe
  --spoof-mac                     --script="smb-*"
  --scan-delay                    --script-args k=v
```

---

> **Further Reading:**
> - `man nmap`
> - `nmap -h`
> - [nmap.org reference guide](https://nmap.org/book/man.html)
> - [NSE script library](https://nmap.org/nsedoc/)
> - [nmap network scanning book](https://nmap.org/book/)
