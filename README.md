# NetworkManager Cheatsheet

> **`nmcli`** — command-line interface for NetworkManager

---

## Table of Contents

- [General Status & Devices](#general-status--devices)
- [Connection Profiles](#connection-profiles)
- [WiFi (Wireless)](#wifi-wireless)
- [Ethernet (Wired)](#ethernet-wired)
- [Bonding (Link Aggregation)](#bonding-link-aggregation)
- [VLANs](#vlans)
- [Bridges](#bridges)
- [Tunnels (GRE, VXLAN, WireGuard)](#tunnels-gre-vxlan-wireguard)
- [IPv4 / IPv6 Addressing & Routing](#ipv4--ipv6-addressing--routing)
- [DNS & Name Resolution](#dns--name-resolution)
- [Connection Flags & Useful Options](#connection-flags--useful-options)
- [Output Formatting & Scripting](#output-formatting--scripting)
- [Troubleshooting & Diagnostics](#troubleshooting--diagnostics)

---

## General Status & Devices

### Status & Overview

```bash
nmcli general status              # Overall NM status, connectivity, and radio state
nmcli general hostname            # Show current system hostname
nmcli general hostname NEW_NAME   # Set hostname persistently
nmcli general logging             # Show current NM log level and domains
nmcli monitor                     # Live monitor NM events
```

### Device Management

```bash
nmcli device status               # List all devices and their state
nmcli device show eth0            # Detailed info for a specific device
nmcli device connect eth0         # Connect a device (auto-picks profile)
nmcli device disconnect eth0      # Disconnect a device
nmcli device reapply eth0         # Reapply config without full reconnect
nmcli device set eth0 managed yes # Enable NM management for a device
```

### Radio / Networking Kill Switches

```bash
nmcli radio all                   # Show all radio switch states
nmcli radio wifi on|off           # Enable or disable Wi-Fi radio
nmcli radio wwan on|off           # Enable or disable mobile broadband
nmcli networking on|off           # Enable or disable all networking
nmcli networking connectivity     # Show internet connectivity state
```

---

## Connection Profiles

### List & Inspect

```bash
nmcli connection show             # List all saved connection profiles
nmcli connection show --active    # Show only active connections
nmcli connection show "My Conn"   # Full details for one connection
nmcli -f all connection show      # Show all fields (no truncation)
```

### Activate / Deactivate

```bash
nmcli connection up "My Conn"                    # Bring up a connection by name
nmcli connection up uuid <UUID>                  # Bring up by UUID
nmcli connection down "My Conn"                  # Bring down a connection
nmcli connection reload                          # Reload all connection files from disk
nmcli connection load /path/file.nmconnection    # Load a specific file
```

### Modify & Delete

```bash
nmcli connection modify "Conn" ipv4.method manual   # Change a property in place
nmcli connection edit "Conn"                        # Open interactive connection editor
nmcli connection clone "Conn" "New"                 # Duplicate a connection profile
nmcli connection delete "Conn"                      # Delete a profile permanently
nmcli connection import type openvpn file vpn.ovpn  # Import VPN config
```

> **Tip:** Use `+property` to append and `-property` to remove individual values without replacing the whole list.
> ```bash
> nmcli connection modify "Conn" +ipv4.dns "1.1.1.1"   # Add a DNS server
> nmcli connection modify "Conn" -ipv4.dns "8.8.8.8"   # Remove a DNS server
> ```

---

## WiFi (Wireless)

### Scan & Connect

```bash
nmcli device wifi list                                          # Scan and list visible APs
nmcli device wifi list ifname wlan0                             # Scan on a specific interface
nmcli device wifi rescan                                        # Force a new scan
nmcli device wifi connect "SSID" password "pass"                # Connect (creates profile)
nmcli device wifi connect "SSID" password "pass" hidden yes     # Connect to a hidden network
```

### Create Wi-Fi Profile Manually

```bash
# WPA2 profile
nmcli connection add type wifi ssid "MyNet" \
  con-name "home-wifi" \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "mypassword" \
  ipv4.method auto

# WPA3 profile (change key-mgmt to sae)
nmcli connection add type wifi ssid "MyNet" \
  con-name "home-wifi-wpa3" \
  wifi-sec.key-mgmt sae \
  wifi-sec.psk "mypassword" \
  ipv4.method auto
```

### Hotspot / Access Point

```bash
# Quick hotspot
nmcli device wifi hotspot \
  ifname wlan0 \
  ssid "MyHotspot" \
  password "hotpass123"

# Manual AP profile
nmcli connection add type wifi ifname wlan0 \
  con-name "ap" ssid "MyAP" \
  mode ap \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "secret"
```

---

## Ethernet (Wired)

### DHCP Connection

```bash
nmcli connection add type ethernet \
  con-name "eth-dhcp" ifname eth0 \
  ipv4.method auto ipv6.method auto
```

### Static IP

```bash
nmcli connection add type ethernet \
  con-name "eth-static" ifname eth0 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 1.1.1.1"
```

### Modify IP on an Existing Connection

```bash
nmcli connection modify "eth-static" ipv4.addresses 192.168.1.50/24   # Change IP
nmcli connection modify "eth-static" ipv4.gateway 192.168.1.254        # Change gateway
nmcli connection modify "eth-static" ipv4.dns "9.9.9.9"               # Replace DNS (overwrites)
nmcli connection modify "eth-static" +ipv4.dns "1.0.0.1"              # Add DNS (appends)
nmcli connection modify "eth-static" -ipv4.dns "8.8.8.8"              # Remove a specific DNS
```

---

## Bonding (Link Aggregation)

### Create Bond Interface

```bash
# Active-backup (failover)
nmcli connection add type bond \
  con-name "bond0" ifname bond0 \
  bond.options "mode=active-backup"

# LACP (802.3ad) — requires switch support
nmcli connection add type bond \
  con-name "bond0" ifname bond0 \
  bond.options "mode=802.3ad,miimon=100"
```

### Add Slaves to Bond

```bash
nmcli connection add type ethernet con-name "bond0-slave1" ifname eth0 master bond0
nmcli connection add type ethernet con-name "bond0-slave2" ifname eth1 master bond0

# Bring up slaves first, then the bond
nmcli connection up bond0-slave1
nmcli connection up bond0-slave2
nmcli connection up bond0
```

### Bond Modes Reference

| Mode | Name | Description |
|------|------|-------------|
| `0` | `balance-rr` | Round-robin across all slaves |
| `1` | `active-backup` | One active slave, others on standby (failover) |
| `2` | `balance-xor` | XOR hash of src/dst MAC |
| `3` | `broadcast` | Transmit on all slaves |
| `4` | `802.3ad` | LACP dynamic link aggregation |
| `5` | `balance-tlb` | Adaptive transmit load balancing |
| `6` | `balance-alb` | Adaptive load balancing (TX + RX) |

---

## VLANs

### Create a VLAN

```bash
# VLAN 100 on top of eth0
nmcli connection add type vlan \
  con-name "vlan100" ifname eth0.100 \
  dev eth0 id 100

# VLAN with static IP
nmcli connection add type vlan \
  con-name "vlan200" ifname eth0.200 \
  dev eth0 id 200 \
  ipv4.method manual \
  ipv4.addresses 10.0.200.1/24
```

### VLAN on a Bond

```bash
nmcli connection add type vlan \
  con-name "vlan-bond" ifname bond0.10 \
  dev bond0 id 10
```

> **Note:** The parent device (`dev`) must be up before the VLAN can activate. VLANs can also be used as bridge ports.

---

## Bridges

### Create a Bridge

```bash
# Bridge with DHCP
nmcli connection add type bridge \
  con-name "br0" ifname br0 \
  ipv4.method auto

# Bridge with static IP and STP
nmcli connection add type bridge \
  con-name "br0" ifname br0 \
  ipv4.method manual \
  ipv4.addresses 192.168.50.1/24 \
  bridge.stp yes
```

### Add Ports to the Bridge

```bash
nmcli connection add type bridge-slave con-name "br0-port1" ifname eth0 master br0
nmcli connection add type bridge-slave con-name "br0-port2" ifname eth1 master br0

# Bring up ports then the bridge
nmcli connection up br0-port1
nmcli connection up br0
```

### Bridge for VMs / Containers

```bash
# Disable STP for faster forwarding (recommended for KVM/libvirt)
nmcli connection modify br0 \
  bridge.forward-delay 0 \
  bridge.stp no
```

---

## Tunnels (GRE, VXLAN, WireGuard)

### GRE Tunnel

```bash
nmcli connection add type ip-tunnel \
  con-name "gre0" ifname gre0 \
  ip-tunnel.mode gre \
  ip-tunnel.local 1.2.3.4 \
  ip-tunnel.remote 5.6.7.8 \
  ipv4.method manual \
  ipv4.addresses 10.10.10.1/30
```

### VXLAN Overlay

```bash
nmcli connection add type vxlan \
  con-name "vxlan10" ifname vxlan10 \
  vxlan.id 10 \
  vxlan.remote 192.168.1.200 \
  vxlan.local 192.168.1.100 \
  vxlan.destination-port 4789
```

### WireGuard

```bash
# Create WireGuard interface
nmcli connection add type wireguard \
  con-name "wg0" ifname wg0 \
  ipv4.method manual \
  ipv4.addresses 10.99.0.1/24 \
  wireguard.private-key "<base64-private-key>"

# Add a peer
nmcli connection modify wg0 \
  +wireguard.peers "public-key=<key>,endpoint=HOST:PORT,allowed-ips=0.0.0.0/0"
```

---

## IPv4 / IPv6 Addressing & Routing

### Multiple IPs on One Interface

```bash
# Add a secondary IP (+ appends)
nmcli connection modify "eth-static" +ipv4.addresses 10.0.0.5/24

# Set multiple IPs at once (replaces existing)
nmcli connection modify "eth-static" ipv4.addresses "192.168.1.10/24,10.0.0.5/24"

# Remove a secondary IP
nmcli connection modify "eth-static" -ipv4.addresses 10.0.0.5/24
```

### Static Routes

```bash
# Add route via gateway
nmcli connection modify "eth-static" +ipv4.routes "10.20.0.0/16 192.168.1.254"

# Add route with metric (lower = preferred)
nmcli connection modify "eth-static" +ipv4.routes "192.168.99.0/24 192.168.1.1 100"

# Remove a static route
nmcli connection modify "eth-static" -ipv4.routes "10.20.0.0/16 192.168.1.254"
```

### IPv6 Configuration

```bash
# Static IPv6 address
nmcli connection modify "eth0" \
  ipv6.method manual \
  ipv6.addresses "2001:db8::1/64" \
  ipv6.gateway "2001:db8::fffe"

nmcli connection modify "eth0" ipv6.method dhcp    # DHCPv6
nmcli connection modify "eth0" ipv6.method ignore  # Disable IPv6 entirely
```

---

## DNS & Name Resolution

### DNS Servers

```bash
nmcli connection modify "conn" ipv4.dns "1.1.1.1 8.8.8.8"   # Set DNS (replaces existing)
nmcli connection modify "conn" +ipv4.dns "9.9.9.9"           # Add a DNS server
nmcli connection modify "conn" -ipv4.dns "8.8.8.8"           # Remove a DNS server
nmcli connection modify "conn" ipv4.ignore-auto-dns yes       # Ignore DHCP-provided DNS
nmcli connection modify "conn" ipv4.dns-priority 10           # Lower value = preferred
```

### Search Domains

```bash
nmcli connection modify "conn" ipv4.dns-search "example.com local.lab"  # Set search domains
nmcli connection modify "conn" +ipv4.dns-search "corp.internal"          # Append a domain
nmcli connection modify "conn" ipv4.dns-search ""                        # Clear all domains
```

---

## Connection Flags & Useful Options

### Autoconnect & Priority

```bash
nmcli connection modify "conn" connection.autoconnect yes         # Enable autoconnect
nmcli connection modify "conn" connection.autoconnect no          # Disable autoconnect
nmcli connection modify "conn" connection.autoconnect-priority 10 # Higher = tried first
nmcli connection modify "conn" connection.autoconnect-retries 3   # Retry limit
```

### Firewall Zone & Permissions

```bash
nmcli connection modify "conn" connection.zone trusted            # Assign firewalld zone
nmcli connection modify "conn" connection.permissions "user:alice"# Restrict to one user
nmcli connection modify "conn" connection.permissions ""          # Allow all users
```

### MTU & MAC Spoofing

```bash
nmcli connection modify "conn" 802-3-ethernet.mtu 9000              # Jumbo frames
nmcli connection modify "conn" wifi.cloned-mac-address random        # Random MAC each connect
nmcli connection modify "conn" wifi.cloned-mac-address stable        # Stable randomised MAC
nmcli connection modify "conn" ethernet.cloned-mac-address AA:BB:CC:DD:EE:FF  # Specific MAC
```

---

## Output Formatting & Scripting

### Terse & Field Selection

```bash
nmcli -t -f NAME,UUID,STATE connection show     # Colon-separated terse output
nmcli -g IP4.ADDRESS device show eth0           # Get only the IPv4 address
nmcli -f GENERAL.HWADDR device show eth0        # Get hardware MAC address
```

### Useful One-liners

```bash
# Print only active connection names
nmcli -t -f NAME con show --active | cut -d: -f1

# Get device state string
nmcli -g GENERAL.STATE device show eth0

# Extract bare IP without prefix length
nmcli -g ip4.address device show eth0 | cut -d/ -f1

# List all connection names
nmcli con show | awk 'NR>1{print $1}'
```

### nmtui & nmstatectl Alternatives

```bash
nmtui                           # Text-based ncurses UI — great for servers
nmtui edit "conn-name"          # Open TUI directly to edit a connection
nmstatectl show                 # YAML state view (requires NetworkManager-config-server)
nmstatectl apply state.yaml     # Apply declarative network state from YAML
```

---

## Troubleshooting & Diagnostics

### Logs

```bash
journalctl -u NetworkManager -f                          # Follow NM logs in real time
journalctl -u NetworkManager -b --no-pager | less        # All logs since last boot
nmcli general logging level DEBUG domains ALL            # Enable verbose debug logging
nmcli general logging level INFO domains ALL             # Restore default log level
```

### Common Fixes

```bash
systemctl restart NetworkManager                         # Restart NM (last resort)
nmcli connection reload                                  # Re-read profiles without restart

# Bounce NM management of a device
nmcli device set eth0 managed no && nmcli device set eth0 managed yes

# Hard reset link layer (outside NM)
ip link set eth0 down && ip link set eth0 up
```

### Config File Locations

| Path | Purpose |
|------|---------|
| `/etc/NetworkManager/NetworkManager.conf` | Main configuration file |
| `/etc/NetworkManager/system-connections/` | Saved connection profiles |
| `/etc/NetworkManager/conf.d/` | Drop-in config snippets |
| `/var/lib/NetworkManager/` | Runtime state and leases |
| `/run/NetworkManager/` | Transient runtime data |

---

## Quick Reference Card

| Task | Command |
|------|---------|
| List devices | `nmcli device status` |
| List connections | `nmcli connection show` |
| Connect to Wi-Fi | `nmcli device wifi connect "SSID" password "pass"` |
| Add static IP | `nmcli connection modify "conn" ipv4.addresses 192.168.1.10/24` |
| Restart a connection | `nmcli connection down "conn" && nmcli connection up "conn"` |
| Check connectivity | `nmcli networking connectivity` |
| Show active IP | `nmcli -g IP4.ADDRESS device show eth0` |
| Reload profiles | `nmcli connection reload` |
| Open text UI | `nmtui` |

---

*Tested on NetworkManager 1.x — most commands apply to NM 1.30+. Refer to `man nmcli` and `man nm-settings` for the full property reference.*
