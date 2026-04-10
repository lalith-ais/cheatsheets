# Linux `ip` Command — Comprehensive Cheat Sheet

> The `ip` command (part of `iproute2`) replaces legacy tools like `ifconfig`, `route`, `arp`, and `netstat`. It's far more powerful, but most documentation only scratches the surface.

---

## Table of Contents

- [Syntax Overview](#syntax-overview)
- [Address Management](#address-management)
- [Link / Interface Management](#link--interface-management)
- [Routing](#routing)
- [Neighbour (ARP/NDP) Table](#neighbour-arpndp-table)
- [Network Namespaces](#network-namespaces)
- [Tunnels & Virtual Interfaces](#tunnels--virtual-interfaces)
- [Policy-Based Routing (Rules)](#policy-based-routing-rules)
- [Multipath Routing](#multipath-routing)
- [VRF (Virtual Routing and Forwarding)](#vrf-virtual-routing-and-forwarding)
- [Traffic Control Hints](#traffic-control-hints)
- [Monitoring & Events](#monitoring--events)
- [Output Formatting](#output-formatting)
- [Batch Mode](#batch-mode)
- [Hidden Gems & Power Tips](#hidden-gems--power-tips)
- [Quick Reference Card](#quick-reference-card)

---

## Syntax Overview

```
ip [ OPTIONS ] OBJECT { COMMAND | help }
```

| Object      | Alias  | What it manages                     |
|-------------|--------|--------------------------------------|
| `address`   | `a`    | IPv4/IPv6 addresses                  |
| `link`      | `l`    | Network interfaces                   |
| `route`     | `r`    | Routing table                        |
| `rule`      | `ru`   | Policy routing rules                 |
| `neighbour` | `n`    | ARP / NDP neighbour cache            |
| `tunnel`    | `t`    | IP tunnels                           |
| `maddress`  | `m`    | Multicast addresses                  |
| `netns`     | `netns`| Network namespaces                   |
| `vrf`       | `vrf`  | Virtual Routing & Forwarding         |
| `monitor`   | —      | Watch for real-time events           |
| `token`     | —      | Interface token (IPv6 SLAAC)         |
| `xfrm`      | —      | IPsec transforms                     |

**Global options:**

| Flag              | Meaning                                                  |
|-------------------|----------------------------------------------------------|
| `-4`              | IPv4 only                                                |
| `-6`              | IPv6 only                                                |
| `-br` / `-brief`  | Brief one-line output                                    |
| `-j` / `-json`    | JSON output (parseable)                                  |
| `-p` / `-pretty`  | Pretty-print JSON                                        |
| `-s` / `-stats`   | Show statistics (use twice for more detail)              |
| `-d` / `-details` | Show full details (especially for tunnels/vxlan)         |
| `-r`              | Resolve hostnames in output                              |
| `-n <netns>`      | Operate inside a named network namespace                 |
| `-batch <file>`   | Execute commands from a file                             |
| `-oneline`        | Print each record on one line (good for `grep`)          |
| `-ts`             | Add timestamps to `ip monitor` output                    |
| `-rc`             | Return NETLINK error codes                               |

---

## Address Management

### Show addresses

```bash
ip address show                    # All interfaces
ip addr show dev eth0              # Specific interface
ip -4 addr show                    # IPv4 only
ip -6 addr show                    # IPv6 only
ip -br addr show                   # Brief one-line per interface
ip -j -p addr show                 # JSON output (pretty)
ip addr show scope global          # Only global (public) addresses
ip addr show scope link            # Only link-local addresses
ip addr show dynamic               # Only DHCP-assigned addresses
ip addr show permanent             # Only statically configured addresses
ip addr show tentative             # IPv6 DAD — addresses being verified
ip addr show deprecated            # Deprecated IPv6 preferred lifetime expired
```

### Add / remove addresses

```bash
ip addr add 192.168.1.10/24 dev eth0
ip addr add 192.168.1.10/24 broadcast 192.168.1.255 dev eth0
ip addr add 2001:db8::1/64 dev eth0

# Add with label (visible in ifconfig compat output)
ip addr add 10.0.0.2/24 dev eth0 label eth0:1

# Add with specific scope
ip addr add 169.254.1.1/16 dev eth0 scope link

# Add with preferred/valid lifetime (seconds; 'forever' = permanent)
ip addr add 192.168.1.20/24 dev eth0 preferred_lft 3600 valid_lft 7200

# Remove specific address
ip addr del 192.168.1.10/24 dev eth0

# Flush all addresses on an interface
ip addr flush dev eth0

# Flush all addresses except loopback
ip addr flush scope global
```

### 💡 Less-known address tricks

```bash
# Assign multiple IPs quickly (loop)
for i in {10..20}; do ip addr add 10.0.0.$i/24 dev dummy0; done

# Check which interface holds a specific address
ip addr show to 10.0.0.1/32

# Show addresses that will expire soon (dynamic lifetime)
ip -6 addr show dynamic

# Peer address on a point-to-point link (e.g. PPP)
ip addr add 10.0.0.1 peer 10.0.0.2 dev ppp0
```

---

## Link / Interface Management

### Show interfaces

```bash
ip link show                       # All links
ip link show dev eth0              # One interface
ip -br link show                   # Brief (state + MAC only)
ip -d link show                    # Full details (incl. VLAN/VXLAN config)
ip -s link show eth0               # With TX/RX statistics
ip -s -s link show eth0            # With error counters
ip link show type vlan             # Filter by type (vlan, bridge, vxlan, …)
ip link show up                    # Only UP interfaces
ip link show master br0            # All members of a bridge
```

### Bring interfaces up/down

```bash
ip link set eth0 up
ip link set eth0 down
```

### Interface properties

```bash
# Change MTU
ip link set eth0 mtu 9000

# Change MAC address (interface must be DOWN)
ip link set eth0 down
ip link set eth0 address 02:42:ac:11:00:01
ip link set eth0 up

# Rename interface
ip link set eth0 name wan0

# Enable/disable promiscuous mode
ip link set eth0 promisc on
ip link set eth0 promisc off

# Enable/disable ARP
ip link set eth0 arp off           # Disable ARP (useful for p2p links)

# Set TX queue length
ip link set eth0 txqueuelen 10000

# Enable/disable multicast
ip link set eth0 multicast on

# Set interface alias (description)
ip link set eth0 alias "uplink to ISP"

# Enable/disable carrier sense
ip link set eth0 carrier off       # Simulate cable unplug (requires special driver support)

# Control neighbour solicitation count (IPv6 DAD)
ip link set eth0 addrgenmode none  # Disable IPv6 auto address generation
```

### Create virtual interfaces

```bash
# Dummy interface (loopback-like, useful for testing)
ip link add dummy0 type dummy

# VLAN interface
ip link add link eth0 name eth0.100 type vlan id 100
ip link add link eth0 name eth0.200 type vlan id 200 protocol 802.1ad  # QinQ outer

# Bridge
ip link add name br0 type bridge
ip link set eth0 master br0        # Add port to bridge
ip link set eth1 master br0
ip link set eth0 nomaster          # Remove from bridge

# Bridge options
ip link set br0 type bridge stp_state 1          # Enable STP
ip link set br0 type bridge forward_delay 4      # FDB forward delay (seconds)
ip link set br0 type bridge vlan_filtering 1     # Enable VLAN-aware bridge

# Bond (bonding/teaming)
ip link add bond0 type bond mode 802.3ad         # LACP
ip link set eth0 master bond0
ip link set eth1 master bond0

# VXLAN
ip link add vxlan10 type vxlan id 10 \
    remote 192.168.1.2 local 192.168.1.1 \
    dstport 4789 dev eth0

# GRE tunnel
ip link add gre1 type gre local 1.2.3.4 remote 5.6.7.8 ttl 64

# IPIP tunnel
ip link add ipip1 type ipip local 1.2.3.4 remote 5.6.7.8

# SIT (Simple Internet Transition — IPv6-in-IPv4)
ip link add sit1 type sit local 1.2.3.4 remote 5.6.7.8

# Macvlan (separate MAC, same underlying NIC)
ip link add macvlan0 link eth0 type macvlan mode bridge
# Modes: private | vepa | bridge | passthru | source

# Macvtap (macvlan exposed as /dev/tapX — useful for VMs)
ip link add macvtap0 link eth0 type macvtap mode bridge

# IPvlan (separate IP, same MAC)
ip link add ipvlan0 link eth0 type ipvlan mode l2
# Modes: l2 | l3 | l3s

# veth pair (virtual ethernet — used in containers/namespaces)
ip link add veth0 type veth peer name veth1

# tun/tap via ip (alternatively use `ip tuntap`)
ip tuntap add dev tap0 mode tap
ip tuntap add dev tun0 mode tun

# Geneve tunnel
ip link add geneve0 type geneve id 42 remote 10.0.0.2

# WireGuard (kernel module must be loaded)
ip link add wg0 type wireguard

# Delete any virtual interface
ip link delete dummy0
```

---

## Routing

### Show routes

```bash
ip route show                      # Main routing table
ip route show table all            # ALL tables (main, local, default, custom)
ip route show table local          # Kernel-managed local/broadcast routes
ip route show table 255            # Table 255 = local (by number)
ip -6 route show                   # IPv6 routes
ip route show dev eth0             # Routes via a specific interface
ip route show via 192.168.1.1      # Routes via a specific gateway
ip route show scope link           # Directly connected routes
ip route show proto static         # Only static routes
ip route show proto kernel         # Only kernel-generated routes
ip route show proto dhcp           # Only DHCP routes
ip -br route show                  # Brief
ip -j -p route show                # JSON

# Route lookup — which route would be used?
ip route get 8.8.8.8
ip route get 8.8.8.8 from 192.168.1.5     # Source-sensitive lookup
ip route get 8.8.8.8 tos 0x10             # With DSCP/TOS
ip route get 8.8.8.8 mark 100             # With firewall mark
ip -6 route get 2001:db8::1
```

### Add / modify / delete routes

```bash
# Default gateway
ip route add default via 192.168.1.1
ip route add default via 192.168.1.1 dev eth0    # Pin to interface too

# Host route
ip route add 10.0.0.5/32 via 192.168.1.1

# Network route
ip route add 10.10.0.0/16 via 192.168.1.1

# Route via interface (no gateway — for p2p)
ip route add 10.0.0.0/8 dev eth0

# Blackhole / reject / unreachable (traffic engineering)
ip route add blackhole 192.168.99.0/24            # Silently drop
ip route add unreachable 192.168.98.0/24          # ICMP unreachable
ip route add prohibit 192.168.97.0/24             # ICMP prohibited

# Route with metric (preference; lower wins)
ip route add 10.0.0.0/8 via 192.168.1.1 metric 100

# Route in a custom table
ip route add 10.0.0.0/8 via 10.10.1.1 table 100

# Modify (replace) an existing route atomically
ip route replace default via 192.168.1.254

# Delete routes
ip route del default
ip route del 10.0.0.0/8
ip route del default table 100

# Flush routes (dangerous — use with care!)
ip route flush dev eth0            # Flush all routes using eth0
ip route flush table 100           # Flush custom table
ip route flush cache               # Flush routing cache (kernel ≥3.6 is no-op)

# Append (add without replacing — creates ECMP entry)
ip route append 10.0.0.0/8 via 192.168.2.1
```

### Route attributes (less visible options)

```bash
# MTU lock — do not discover PMTU, always use this MTU
ip route add 10.0.0.0/8 via 192.168.1.1 mtu lock 1400

# Window, RTT, RTTVAR, SSTHRESH hints
ip route add 10.0.0.0/8 via 192.168.1.1 window 65535 rtt 50 rttvar 10

# Scope
ip route add 192.168.1.0/24 dev eth0 scope link     # Directly connected
ip route add 10.0.0.0/8 via 192.168.1.1 scope global

# Protocol tag (informational — who installed this route?)
ip route add 10.0.0.0/8 via 192.168.1.1 proto static
ip route add 10.0.0.0/8 via 192.168.1.1 proto 42    # Custom proto ID

# TOS matching
ip route add 10.0.0.0/8 via 192.168.1.1 tos 0x10

# Set source address hint
ip route add 10.0.0.0/8 via 192.168.1.1 src 192.168.1.50

# Realm (for use with tc filters)
ip route add 10.0.0.0/8 via 192.168.1.1 realm 5

# Expires (seconds — for dynamic routes)
ip route add 192.168.5.0/24 via 192.168.1.1 expires 300
```

---

## Neighbour (ARP/NDP) Table

```bash
ip neigh show                      # All neighbours
ip neigh show dev eth0             # Per-interface
ip -6 neigh show                   # IPv6 NDP only
ip neigh show nud reachable        # Only reachable entries
ip neigh show nud stale            # Stale (not verified recently)
ip neigh show nud permanent        # Static entries
# NUD states: permanent | noarp | reachable | stale | incomplete | delay | probe | failed

# Add a static ARP entry
ip neigh add 192.168.1.1 lladdr 00:11:22:33:44:55 dev eth0 nud permanent

# Replace (add or update)
ip neigh replace 192.168.1.1 lladdr aa:bb:cc:dd:ee:ff dev eth0 nud permanent

# Delete an entry
ip neigh del 192.168.1.1 dev eth0

# Flush all stale/failed entries
ip neigh flush dev eth0
ip neigh flush all nud stale
ip neigh flush all nud failed

# Force NUD reachability check (re-probe)
ip neigh change 192.168.1.1 dev eth0 nud reachable

# Show proxy ARP entries
ip neigh show proxy
ip neigh add proxy 10.0.0.5 dev eth0
```

---

## Network Namespaces

Network namespaces give you completely isolated network stacks — used by containers, VPNs, and testing environments.

```bash
# Create / delete
ip netns add red
ip netns del red

# List namespaces
ip netns list
ip netns show              # Same, shows nsid if available

# Execute command inside a namespace
ip netns exec red ip addr show
ip netns exec red bash     # Open a shell in the namespace

# Identify current namespace
ip netns identify          # Print name of current process's netns

# Monitor events inside a namespace
ip netns exec red ip monitor

# Attach interface to namespace
ip link set veth1 netns red

# Give it an address inside the namespace
ip netns exec red ip addr add 10.0.0.2/24 dev veth1
ip netns exec red ip link set veth1 up
ip netns exec red ip link set lo up

# Full veth pair + namespace setup (container pattern)
ip netns add myns
ip link add veth-host type veth peer name veth-ns
ip link set veth-ns netns myns
ip addr add 10.1.0.1/30 dev veth-host
ip link set veth-host up
ip netns exec myns ip addr add 10.1.0.2/30 dev veth-ns
ip netns exec myns ip link set veth-ns up
ip netns exec myns ip link set lo up
ip netns exec myns ip route add default via 10.1.0.1

# Assign numeric namespace ID (nsid) — useful for monitoring
ip netns set red 42
ip netns list-id

# Share a namespace with another process (by PID)
ip link set eth0 netns 1234        # Move eth0 into PID 1234's netns

# Run with namespace without exec (alternative)
nsenter --net=/run/netns/red ip addr
```

---

## Tunnels & Virtual Interfaces

### GRE

```bash
# Basic GRE (IP-in-IP with GRE header)
ip tunnel add gre1 mode gre local 1.2.3.4 remote 5.6.7.8 ttl 64
ip addr add 172.16.0.1/30 dev gre1
ip link set gre1 up

# GRE keepalive
ip tunnel change gre1 keepalive 10 hold 4   # 10s interval, 4 retries

# GRE key (multiple tunnels between same endpoints)
ip tunnel add gre2 mode gre local 1.2.3.4 remote 5.6.7.8 key 1234

# Show tunnels
ip tunnel show
```

### VXLAN

```bash
# Unicast VXLAN
ip link add vxlan100 type vxlan id 100 \
    local 10.0.0.1 remote 10.0.0.2 \
    dstport 4789 dev eth0

# Multicast VXLAN (learning mode)
ip link add vxlan100 type vxlan id 100 \
    group 239.1.1.1 \
    dstport 4789 dev eth0

# Inspect VXLAN FDB
bridge fdb show dev vxlan100

# Add static VXLAN FDB entry
bridge fdb add 00:00:00:00:00:00 dev vxlan100 dst 10.0.0.3 self permanent
```

### WireGuard

```bash
ip link add wg0 type wireguard
ip addr add 10.200.0.1/24 dev wg0
wg set wg0 private-key /etc/wireguard/private.key listen-port 51820
wg set wg0 peer <pubkey> allowed-ips 10.200.0.2/32 endpoint 5.6.7.8:51820
ip link set wg0 up
```

---

## Policy-Based Routing (Rules)

Policy routing lets you route packets differently based on source IP, destination IP, TOS, firewall mark, and more — beyond a single routing table.

```bash
# Show rules (priority order, lower = higher priority)
ip rule show
ip -6 rule show

# Add rules (traffic from 192.168.2.0/24 uses table 200)
ip rule add from 192.168.2.0/24 table 200
ip rule add to 10.0.0.0/8 table 200

# Route by firewall mark (from iptables/nftables -j MARK)
ip rule add fwmark 0x1 table 100

# Route by TOS/DSCP
ip rule add tos 0x10 table 300

# Priority (default is 32766 for main table)
ip rule add from 10.0.0.0/8 table 100 priority 100

# Blackhole in policy routing
ip rule add from 172.16.0.0/16 blackhole

# Delete a rule
ip rule del from 192.168.2.0/24 table 200
ip rule del priority 100

# Flush all non-default rules (careful!)
ip rule flush

# Typical dual-ISP setup pattern:
# Table 1 = ISP1, Table 2 = ISP2
ip route add default via 203.0.113.1 table 1
ip route add default via 198.51.100.1 table 2
ip rule add from 203.0.113.10 table 1
ip rule add from 198.51.100.10 table 2
```

---

## Multipath Routing

Send traffic across multiple gateways simultaneously (ECMP — Equal-Cost Multi-Path).

```bash
# Basic ECMP (equal weight)
ip route add default \
    nexthop via 192.168.1.1 dev eth0 \
    nexthop via 192.168.2.1 dev eth1

# Weighted ECMP (2:1 ratio eth0:eth1)
ip route add default \
    nexthop via 192.168.1.1 dev eth0 weight 2 \
    nexthop via 192.168.2.1 dev eth1 weight 1

# ECMP with encapsulation (MPLS)
ip route add 10.0.0.0/8 \
    nexthop via 192.168.1.1 encap mpls 100 \
    nexthop via 192.168.2.1 encap mpls 200

# View multipath routes
ip route show | grep -A5 "^default"

# Kernel hash policy for ECMP (system-wide)
# 0 = L3 (src+dst IP), 1 = L4 (src+dst IP+port), 2 = L3+L4
sysctl -w net.ipv4.fib_multipath_hash_policy=1
```

---

## VRF (Virtual Routing and Forwarding)

VRFs provide complete routing table isolation for different tenants/services on the same host.

```bash
# Create a VRF
ip link add vrf-blue type vrf table 10
ip link set vrf-blue up

# Assign interface to VRF
ip link set eth1 master vrf-blue

# Show VRF assignments
ip vrf show
ip link show master vrf-blue

# Execute command in VRF context
ip vrf exec vrf-blue ping 8.8.8.8
ip vrf exec vrf-blue ssh user@host

# Show routes in VRF
ip route show vrf vrf-blue
ip -6 route show vrf vrf-blue

# Add route to VRF
ip route add default via 10.0.0.1 vrf vrf-blue
```

---

## Traffic Control Hints

`ip` integrates with `tc` (traffic control) via route realms and marks. A few useful pointers:

```bash
# Assign realm to a route (used by tc for classification)
ip route add 10.0.0.0/8 via 192.168.1.1 realm 5

# Check interface qdisc (queue discipline)
tc qdisc show dev eth0

# Attach a simple rate limiter (HTB via tc, triggered from ip route)
tc qdisc add dev eth0 root handle 1: htb default 12
tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
tc class add dev eth0 parent 1:1 classid 1:12 htb rate 100mbit

# Use DSCP/TOS in routing + marking pipeline
ip route add 10.0.0.0/8 via 192.168.1.1 tos 0x10
```

---

## Monitoring & Events

```bash
# Watch for real-time network events (Ctrl+C to stop)
ip monitor                         # All events
ip monitor link                    # Interface up/down/changes
ip monitor address                 # Address add/del
ip monitor route                   # Route add/del/change
ip monitor neigh                   # ARP/NDP changes
ip monitor all                     # Everything

# With timestamps
ip -ts monitor all

# With namespace events
ip monitor nsid

# Watch only specific interfaces
ip monitor link dev eth0

# Log monitor output to file
ip -ts monitor all >> /var/log/ip-events.log &

# One-liner: alert when a link goes down
ip monitor link | awk '/DOWN/{print "Interface DOWN:", $0; fflush()}'
```

---

## Output Formatting

### JSON output (scriptable)

```bash
ip -j addr show | python3 -m json.tool
ip -j -p route show                # Pretty JSON
ip -j neigh show | jq '.[].dst'    # Extract with jq

# Get IP of specific interface as plain text
ip -j addr show dev eth0 | jq -r '.[0].addr_info[] | select(.family=="inet") | .local'

# List all interface names that are UP
ip -j link show | jq -r '.[] | select(.operstate=="UP") | .ifname'

# Get default gateway
ip -j route show default | jq -r '.[0].gateway'
```

### Oneline + grep patterns

```bash
ip -oneline addr show | grep "scope global"
ip -oneline route show | grep "^default"
ip -oneline neigh show | grep STALE
```

### Brief mode

```bash
ip -br addr show    # ifname  STATE  address/prefix
ip -br link show    # ifname  STATE  MAC
ip -br neigh show   # ip  dev  MAC  STATE
```

---

## Batch Mode

Run many `ip` commands from a file efficiently (single netlink socket — much faster than looping in shell).

```bash
# Create batch file
cat > /tmp/ip-batch.txt << 'EOF'
link add dummy0 type dummy
link add dummy1 type dummy
addr add 10.0.0.1/24 dev dummy0
addr add 10.0.0.2/24 dev dummy1
link set dummy0 up
link set dummy1 up
EOF

# Execute batch
ip -batch /tmp/ip-batch.txt

# Batch from stdin
ip -batch - << 'EOF'
route add 10.1.0.0/16 via 192.168.1.1
route add 10.2.0.0/16 via 192.168.1.1
route add 10.3.0.0/16 via 192.168.1.1
EOF

# Force continue on errors (default stops on first error)
ip -force -batch /tmp/ip-batch.txt
```

---

## Hidden Gems & Power Tips

### 1. Route `get` — simulate a real lookup

```bash
# Answers: "which route would the kernel actually use?"
ip route get 8.8.8.8
ip route get 8.8.8.8 from 10.0.0.1 iif eth0
ip route get 8.8.8.8 mark 100
ip route get 8.8.8.8 tos 0x10
ip -6 route get 2001:4860:4860::8888
```

### 2. Multiple routing tables

```bash
# Table IDs 1-252 are user-defined; named tables in /etc/iproute2/rt_tables
echo "200 isp2" >> /etc/iproute2/rt_tables
ip route add default via 203.0.113.1 table isp2
ip route show table isp2
```

### 3. Lifetime management for addresses

```bash
# Set IPv6 preferred/valid lifetime (SLAAC-like)
ip addr add 2001:db8::1/64 dev eth0 \
    preferred_lft 1800 valid_lft 3600

# Check remaining lifetime
ip -6 addr show dev eth0
```

### 4. Link local multicast — find neighbours fast

```bash
# Ping IPv6 all-nodes multicast (find all link devices)
ping6 -I eth0 ff02::1

# Check who responded (NDP cache)
ip -6 neigh show dev eth0
```

### 5. Get interface statistics as JSON

```bash
ip -j -s link show eth0 | jq '.[0].stats64'
```

### 6. Detect duplicate addresses (IPv4 ARP probe)

```bash
# arping — not ip, but pairs well with ip addr
arping -D -I eth0 192.168.1.100 && echo "Address is free"
```

### 7. Proxy ARP / NDP

```bash
# Proxy ARP (answer ARP requests on behalf of another host)
ip neigh add proxy 10.0.0.5 dev eth0

# Enable proxy ARP at kernel level
sysctl -w net.ipv4.conf.eth0.proxy_arp=1

# Proxy NDP (IPv6)
ip -6 neigh add proxy 2001:db8::5 dev eth0
sysctl -w net.ipv6.conf.eth0.proxy_ndp=1
```

### 8. MPLS encapsulation on routes

```bash
# Requires CONFIG_NET_MPLS_GSO and MPLS modules
modprobe mpls_router mpls_iptunnel
sysctl -w net.mpls.platform_labels=65536
sysctl -w net.mpls.conf.eth0.input=1

ip route add 10.0.0.5/32 encap mpls 100 via 192.168.1.1 dev eth0
ip -f mpls route show
ip -f mpls route add 100 via inet 192.168.1.1 dev eth0
```

### 9. SR-IOV VF (Virtual Functions) management

```bash
# Enable SR-IOV VFs (NIC must support it)
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# View VFs
ip link show eth0                  # Shows VF section

# Set VF MAC/VLAN/rate
ip link set eth0 vf 0 mac 00:11:22:33:44:55
ip link set eth0 vf 0 vlan 100
ip link set eth0 vf 0 max_tx_rate 100    # Mbps
ip link set eth0 vf 0 trust on
ip link set eth0 vf 0 spoofchk off

# Set VF into a namespace (for containers)
ip link set eth0 vf 0 netns mycontainer
```

### 10. XDP / offload status

```bash
# Check XDP program attached to an interface
ip -d link show eth0 | grep xdp

# Attach XDP program (requires iproute2 with BPF support)
ip link set eth0 xdp obj bpf_prog.o sec xdp
ip link set eth0 xdp off            # Detach
```

### 11. VLAN bridging details

```bash
# Show VLAN filtering on bridge ports
bridge vlan show

# VLAN-aware bridge setup
ip link add br0 type bridge vlan_filtering 1
ip link set eth0 master br0
ip link set eth1 master br0
bridge vlan add vid 100 dev eth0
bridge vlan add vid 100 dev eth1
bridge vlan add vid 100 dev br0 self    # Include bridge interface
```

### 12. IPv6 token (SLAAC with fixed IID)

```bash
# Force a specific interface identifier for SLAAC
# Address = prefix (from RA) + token suffix
ip token set ::dead:beef dev eth0
ip token show
```

### 13. Persist with NetworkManager / systemd-networkd

```bash
# Quick test without persistence — use ip commands
# To persist, write to /etc/systemd/network/*.network
# Or use nmcli / nmtui for NetworkManager

# Show what NetworkManager thinks
nmcli device show eth0

# Export current routes as systemd-networkd config
ip route show | awk '{print "# " $0}'
```

### 14. One-liner cheatcodes

```bash
# Find default gateway(s)
ip route | awk '/^default/{print $3}'

# Find public/global IP of an interface
ip -4 addr show eth0 | awk '/inet /{print $2}' | cut -d/ -f1

# Check if an IP is assigned anywhere on the host
ip addr show to 192.168.1.100/32

# Count routes in main table
ip route show | wc -l

# Show routing table 255 (local — kernel managed)
ip route show table local

# List all custom routing tables with routes
for t in $(ip rule show | awk '/lookup/{print $NF}' | sort -u); do
  echo "=== Table: $t ==="; ip route show table $t 2>/dev/null
done

# Quick connectivity test via specific interface
ip route get 8.8.8.8 | awk '{print "uses:", $5, "via", $3}'
```

---

## Quick Reference Card

| Goal | Command |
|------|---------|
| Show all IPs | `ip -br addr show` |
| Show routes | `ip route show` |
| Default gateway | `ip route \| awk '/^default/{print $3}'` |
| Add address | `ip addr add 10.0.0.1/24 dev eth0` |
| Remove address | `ip addr del 10.0.0.1/24 dev eth0` |
| Add default route | `ip route add default via 192.168.1.1` |
| Which route used? | `ip route get 8.8.8.8` |
| Bring link up | `ip link set eth0 up` |
| Change MTU | `ip link set eth0 mtu 9000` |
| Change MAC | `ip link set eth0 address aa:bb:cc:dd:ee:ff` |
| Flush ARP | `ip neigh flush dev eth0` |
| Static ARP | `ip neigh add 10.0.0.1 lladdr aa:bb... dev eth0 nud permanent` |
| Create VLAN | `ip link add link eth0 name eth0.10 type vlan id 10` |
| Create dummy | `ip link add dummy0 type dummy` |
| Create veth pair | `ip link add veth0 type veth peer name veth1` |
| New namespace | `ip netns add myns` |
| Run in namespace | `ip netns exec myns <command>` |
| Policy rule | `ip rule add from 10.0.0.0/8 table 100` |
| Watch events | `ip -ts monitor all` |
| JSON output | `ip -j -p addr show` |

---

## Further Reading

- `man ip` — the full manual
- `man ip-address`, `man ip-link`, `man ip-route`, `man ip-rule`, `man ip-neighbour` — per-object man pages
- `/etc/iproute2/rt_tables` — named routing table definitions
- `/etc/iproute2/rt_protos` — route protocol name definitions
- `/etc/iproute2/rt_dsfield` — DSCP/TOS field names
- [iproute2 source](https://github.com/iproute2/iproute2)
- [Linux Advanced Routing & Traffic Control (LARTC)](https://lartc.org/)

---

*Tested on Linux kernel 5.x–6.x with iproute2 ≥ 5.x. Some features (MPLS, XDP, SR-IOV) require specific kernel config or hardware support.*
