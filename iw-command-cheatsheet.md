# Linux `iw` Command — Comprehensive Cheat Sheet

> `iw` is the modern nl80211-based CLI tool for managing wireless devices and interfaces on Linux. It replaces the legacy `iwconfig`/`iwlist`/`iwspy` suite. Almost everything Wi-Fi lives here — scanning, connecting, monitoring, mesh, statistics, regulatory, and more.

---

## Table of Contents

- [Syntax Overview](#syntax-overview)
- [Device & Interface Information](#device--interface-information)
- [Scanning](#scanning)
- [Connecting to Networks](#connecting-to-networks)
- [Interface Management](#interface-management)
- [Station / Peer Information](#station--peer-information)
- [Regulatory Domain](#regulatory-domain)
- [Power Management](#power-management)
- [Transmit Power](#transmit-power)
- [Bitrate & Rate Control](#bitrate--rate-control)
- [Monitor Mode](#monitor-mode)
- [Mesh Networking (802.11s)](#mesh-networking-80211s)
- [IBSS / Ad-Hoc Mode](#ibss--ad-hoc-mode)
- [P2P / Wi-Fi Direct](#p2p--wi-fi-direct)
- [WoWLAN (Wake on Wireless LAN)](#wowlan-wake-on-wireless-lan)
- [Coalesce / Offload](#coalesce--offload)
- [Channel & Frequency Management](#channel--frequency-management)
- [Statistics & Debugging](#statistics--debugging)
- [Event Monitoring](#event-monitoring)
- [Hidden Gems & Power Tips](#hidden-gems--power-tips)
- [Quick Reference Card](#quick-reference-card)
- [iw vs iwconfig Mapping](#iw-vs-iwconfig-mapping)

---

## Syntax Overview

```
iw [ OPTIONS ] { dev <devname> | phy <phyname> | reg } COMMAND
```

| Target      | Meaning                                               |
|-------------|-------------------------------------------------------|
| `dev <if>`  | Operate on a wireless interface (e.g. `wlan0`)        |
| `phy <phy>` | Operate on a physical radio (e.g. `phy0`)             |
| `reg`       | Operate on the regulatory subsystem                   |

**Options:**

| Flag          | Meaning                                      |
|---------------|----------------------------------------------|
| `--debug`     | Show raw nl80211 netlink messages            |
| `--version`   | Print iw version                             |
| `-h` / `help` | Show help for an object or command           |

> **Tip:** Every object and sub-command accepts `help` — e.g. `iw dev wlan0 scan help` lists all scan options.

---

## Device & Interface Information

### Physical radio (phy)

```bash
# List all wireless physical devices
iw list
iw phy                             # Same

# Info for a specific phy
iw phy phy0 info

# What the phy supports — critical for capability checking
iw phy phy0 info | grep -A5 "Supported interface modes"
iw phy phy0 info | grep -A5 "Band "
iw phy phy0 info | grep "Capabilities"
iw phy phy0 info | grep "HT TX/RX MCS"
iw phy phy0 info | grep -A10 "VHT Capabilities"   # 802.11ac
iw phy phy0 info | grep -A10 "HE "                # 802.11ax (Wi-Fi 6)
iw phy phy0 info | grep "valid antenna"           # Antenna bitmasks
iw phy phy0 info | grep "max scan SSIDs"
iw phy phy0 info | grep "max # scan plans"
```

### Wireless interfaces

```bash
# List all wireless interfaces with their phy, type, MAC, channel
iw dev

# Detailed info for a specific interface
iw dev wlan0 info

# Show interface name, type, MAC, channel, width, TX power, etc.
iw dev wlan0 info
```

**Example output fields:**
```
Interface wlan0
  ifindex 3
  wdev 0x1
  addr aa:bb:cc:dd:ee:ff
  type managed
  wiphy 0
  channel 6 (2437 MHz), width: 20 MHz, center1: 2437 MHz
  txpower 20.00 dBm
```

---

## Scanning

### Basic scan

```bash
# Passive scan — trigger and dump results
iw dev wlan0 scan

# Scan and dump results (with -u for unknown elements)
iw dev wlan0 scan dump

# Trigger scan only (non-blocking)
iw dev wlan0 scan trigger

# Scan for specific SSIDs only (active/directed probe)
iw dev wlan0 scan ssid "MyNetwork"
iw dev wlan0 scan ssid "Net1" ssid "Net2"

# Scan specific frequencies (MHz)
iw dev wlan0 scan freq 2412 2437 2462

# Scan specific channels
iw dev wlan0 scan freq 5180 5200 5220 5240       # 5 GHz channels 36,40,44,48

# Flush cached results and get fresh scan
iw dev wlan0 scan trigger flush 1

# Passive-only scan (do not send probe requests)
iw dev wlan0 scan passive

# Scan with custom IEs (hex — e.g. vendor element)
iw dev wlan0 scan ies <hexstring>

# Dump last scan results without re-scanning
iw dev wlan0 scan dump

# Duration in TUs (time units, 1 TU = 1024 µs) per channel
iw dev wlan0 scan duration 200
```

### Parsing scan output

```bash
# Show only SSIDs and signal strength
iw dev wlan0 scan | awk '/SSID:|signal:/{print $0}' | paste - -

# Show SSID, BSSID, channel, signal — clean table
iw dev wlan0 scan | grep -E "^BSS|SSID:|signal:|DS Parameter" \
  | paste - - - - \
  | awk '{print $2, $4, $6, $8}'

# Find all hidden networks (empty SSID)
iw dev wlan0 scan | grep 'SSID: $'

# Check capabilities of a specific AP
iw dev wlan0 scan | grep -A 40 "BSS aa:bb:cc:dd:ee:ff"

# Show only 5 GHz APs
iw dev wlan0 scan | awk '/^BSS/{bss=$0} /5[0-9]{3} MHz/{print bss}'

# Show WPA3/SAE capable networks
iw dev wlan0 scan | grep -B5 "SAE"

# Show only networks with WPA2
iw dev wlan0 scan | grep -B2 "WPA2"

# Sort by signal strength
iw dev wlan0 scan | grep -E "BSS |signal:" | paste - - | sort -k4 -n -r
```

### Scheduled scan (if supported)

```bash
# Scan every 30 seconds, with 10 channels per scan pass
iw dev wlan0 scan sched_start interval 30 \
    plans 10:2412,2437,2462,5180,5200

# Stop scheduled scan
iw dev wlan0 scan sched_stop
```

---

## Connecting to Networks

> **Note:** `iw` handles the 802.11 layer only. For full WPA2/WPA3 authentication use `wpa_supplicant`. `iw connect` works only for **open** (unencrypted) networks.

### Open network

```bash
# Connect to open network
iw dev wlan0 connect "MyOpenNetwork"

# Connect to specific BSSID (access point)
iw dev wlan0 connect "MyOpenNetwork" aa:bb:cc:dd:ee:ff

# Connect on a specific frequency
iw dev wlan0 connect "MyOpenNetwork" 2437

# Connect with keys (WEP — legacy, avoid)
iw dev wlan0 connect "OldNetwork" key 0:d1:23:45:67:89

# Disconnect
iw dev wlan0 disconnect
```

### Link status

```bash
# Show current association details
iw dev wlan0 link

# Example output includes: SSID, BSSID, channel, RSSI, TX/RX bitrate, MCS index, flags
```

---

## Interface Management

### Create / delete interfaces

```bash
# Add a new virtual interface on phy0
iw phy phy0 interface add wlan1 type managed
iw phy phy0 interface add mon0 type monitor
iw phy phy0 interface add mesh0 type mesh_point
iw phy phy0 interface add ibss0 type ibss
iw phy phy0 interface add p2p0 type p2p-client

# Delete an interface
iw dev mon0 del

# Rename (use ip link — iw doesn't rename)
ip link set wlan1 name wlan-guest
```

### Interface types

| Type            | Use case                                        |
|-----------------|-------------------------------------------------|
| `managed`       | Normal client (STA) mode                        |
| `monitor`       | Promiscuous capture — sees all frames           |
| `ap`            | Access point (usually managed by hostapd)       |
| `mesh_point`    | 802.11s mesh node                               |
| `ibss`          | Ad-hoc / IBSS network                           |
| `p2p-client`    | Wi-Fi Direct client                             |
| `p2p-go`        | Wi-Fi Direct Group Owner (AP role)              |
| `ocb`           | Outside Context of BSS (802.11p — DSRC/V2X)    |
| `nan`           | Neighbour Awareness Networking                  |

### Set interface type

```bash
# Interface must be DOWN first
ip link set wlan0 down
iw dev wlan0 set type monitor
ip link set wlan0 up

# Back to managed
ip link set wlan0 down
iw dev wlan0 set type managed
ip link set wlan0 up
```

### Set 4-address (WDS) mode

```bash
# 4-address mode — for bridging through AP (client + bridge)
iw dev wlan0 set 4addr on
iw dev wlan0 set 4addr off
```

### Set mesh ID / peer offset

```bash
iw dev mesh0 set mesh_id "my-mesh"
```

---

## Station / Peer Information

```bash
# Show all associated stations (AP mode or mesh)
iw dev wlan0 station dump

# Show a specific station
iw dev wlan0 station get aa:bb:cc:dd:ee:ff

# Key fields in station dump output:
#   signal:           Last beacon/frame RSSI (dBm)
#   signal avg:       Average RSSI
#   tx bitrate:       Current TX MCS/rate
#   rx bitrate:       Current RX MCS/rate
#   tx packets/failed/retries
#   rx packets/bytes
#   connected time
#   inactive time

# Delete / kick a station (AP mode)
iw dev wlan0 station del aa:bb:cc:dd:ee:ff

# Delete with reason code (802.11 reason codes 1-66)
iw dev wlan0 station del aa:bb:cc:dd:ee:ff reason 3   # 3 = leaving BSS

# Inject station statistics update (for mesh)
iw dev mesh0 station set aa:bb:cc:dd:ee:ff mesh_power_mode active
```

### Parse station dump

```bash
# Signal strength for all stations
iw dev wlan0 station dump | grep -E "Station|signal:"

# Top stations by TX bitrate
iw dev wlan0 station dump | grep -E "Station|tx bitrate" | paste - -

# Count connected stations
iw dev wlan0 station dump | grep -c "^Station"
```

---

## Regulatory Domain

The regulatory domain controls which channels and power levels are legal in your country.

```bash
# Show current regulatory domain
iw reg get

# Set regulatory domain (requires root; persists until reboot or change)
iw reg set GB                      # United Kingdom
iw reg set US                      # United States
iw reg set DE                      # Germany
iw reg set JP                      # Japan

# Show per-phy regulatory info
iw phy phy0 reg get

# Show what channels are allowed in current regdomain
iw list | grep -A 30 "Frequencies:"

# Check if 5 GHz DFS channels are available
iw list | grep "MHz \[" | grep -v disabled | grep 5[0-9][0-9][0-9]

# DFS (Dynamic Frequency Selection) state
iw dev wlan0 cac done              # Signal CAC complete (testing)
```

> **Note:** The `CRDA` daemon and `/lib/crda/regulatory.bin` enforce regulatory rules in older setups. Modern kernels use built-in `cfg80211` regulatory database.

---

## Power Management

```bash
# Show current power save state
iw dev wlan0 get power_save

# Enable / disable power saving
iw dev wlan0 set power_save on
iw dev wlan0 set power_save off

# U-APSD (WMM power save) per-AC queue — if supported
# Controlled via wpa_supplicant uapsd config

# Check if phy supports power save modes
iw phy phy0 info | grep -i "power save"
```

---

## Transmit Power

```bash
# Show current TX power
iw dev wlan0 info | grep txpower

# Set TX power (auto — let driver/regdomain decide)
iw dev wlan0 set txpower auto

# Set fixed TX power in mBm (millidBm; 1 dBm = 100 mBm)
iw dev wlan0 set txpower fixed 2000    # 20 dBm
iw dev wlan0 set txpower fixed 1500    # 15 dBm
iw dev wlan0 set txpower fixed 0       # 0 dBm (minimum)

# Set TX power limit (cap; driver may use less)
iw dev wlan0 set txpower limit 2000

# Per-phy TX power
iw phy phy0 set txpower fixed 2000
iw phy phy0 set txpower auto
```

---

## Bitrate & Rate Control

```bash
# Show supported rates/MCS
iw phy phy0 info | grep -A 20 "Bitrates"

# Force legacy bitrate (Mbps) — 2.4 GHz
iw dev wlan0 set bitrates legacy-2.4 54        # Force 54 Mbps only
iw dev wlan0 set bitrates legacy-2.4 6 9 12 18 24 36 48 54

# Force HT (802.11n) MCS index
iw dev wlan0 set bitrates ht-mcs-2.4 7         # MCS 7 only on 2.4 GHz
iw dev wlan0 set bitrates ht-mcs-5 15          # MCS 15 (2-stream) on 5 GHz

# Force VHT (802.11ac) MCS
iw dev wlan0 set bitrates vht-mcs-5 0:9        # NSS 1, MCS 9 on 5 GHz

# Force HE (802.11ax / Wi-Fi 6) MCS
iw dev wlan0 set bitrates he-mcs-5 0:11        # NSS 1, MCS 11 on 5 GHz

# Reset to automatic rate control
iw dev wlan0 set bitrates                      # No args = auto

# GI (Guard Interval) — 802.11n/ac
iw dev wlan0 set bitrates sgi-2.4              # Short GI on 2.4 GHz
iw dev wlan0 set bitrates sgi-5                # Short GI on 5 GHz

# Set STBC (Space-Time Block Coding)
iw dev wlan0 set bitrates stbc-2.4 1
```

---

## Monitor Mode

Monitor mode allows capturing all 802.11 frames, including management and control frames.

```bash
# Method 1: Change existing interface to monitor
ip link set wlan0 down
iw dev wlan0 set type monitor
ip link set wlan0 up

# Method 2: Create dedicated monitor interface (keep wlan0 managed)
iw phy phy0 interface add mon0 type monitor
ip link set mon0 up

# Set monitor flags (control which frames are seen)
iw dev mon0 set monitor none          # All frames (default)
iw dev mon0 set monitor fcsfail       # Include frames with bad FCS
iw dev mon0 set monitor plcpfail      # Include PLCP errors
iw dev mon0 set monitor control       # Include control frames
iw dev mon0 set monitor otherbss      # Include other BSS frames
iw dev mon0 set monitor cooked        # Cooked monitor (no radiotap)
iw dev mon0 set monitor active        # Active monitor (can TX probe requests)
# Flags can be combined: iw dev mon0 set monitor fcsfail control otherbss

# Set channel for capture
iw dev mon0 set channel 6             # 20 MHz
iw dev mon0 set channel 6 HT20
iw dev mon0 set channel 6 HT40+       # 40 MHz, secondary above
iw dev mon0 set channel 6 HT40-       # 40 MHz, secondary below
iw dev mon0 set freq 5180 80MHz       # 80 MHz VHT on 5 GHz

# Now capture with tcpdump or Wireshark
tcpdump -i mon0 -w capture.pcap
tcpdump -i mon0 type mgt               # Management frames only
tcpdump -i mon0 type ctl               # Control frames
tcpdump -i mon0 'wlan addr2 aa:bb:cc:dd:ee:ff'    # Filter by source MAC

# Channel hopping script (basic)
for ch in 1 6 11 36 40 44 48; do
  iw dev mon0 set channel $ch
  sleep 0.5
done
```

---

## Mesh Networking (802.11s)

```bash
# Create mesh interface
iw phy phy0 interface add mesh0 type mesh_point
ip link set mesh0 up

# Join a mesh network
iw dev mesh0 mesh join "my-mesh-id"

# Join with specific frequency
iw dev mesh0 mesh join "my-mesh-id" freq 2412

# Join with HT40
iw dev mesh0 mesh join "my-mesh-id" freq 5180 HT40+

# Leave mesh
iw dev mesh0 mesh leave

# Show mesh configuration
iw dev mesh0 get mesh_param

# Set mesh parameters
iw dev mesh0 set mesh_param mesh_retry_timeout 100
iw dev mesh0 set mesh_param mesh_confirm_timeout 100
iw dev mesh0 set mesh_param mesh_holding_timeout 100
iw dev mesh0 set mesh_param mesh_max_peer_links 10
iw dev mesh0 set mesh_param mesh_hwmp_rootmode 1       # RANN (root announcement)
iw dev mesh0 set mesh_param mesh_gate_announcements 1  # Announce as mesh gate
iw dev mesh0 set mesh_param mesh_fwding 1              # Enable frame forwarding
iw dev mesh0 set mesh_param mesh_rssi_threshold -70    # Minimum RSSI for peering
iw dev mesh0 set mesh_param mesh_power_mode active
iw dev mesh0 set mesh_param mesh_awake_window 10       # ms awake in power-save

# Show mesh peers
iw dev mesh0 station dump

# Show mesh path table (HWMP routing)
iw dev mesh0 mpath dump

# Get a specific mesh path
iw dev mesh0 mpath get aa:bb:cc:dd:ee:ff

# Add static mesh path
iw dev mesh0 mpath set aa:bb:cc:dd:ee:ff next_hop 11:22:33:44:55:66

# Delete mesh path
iw dev mesh0 mpath del aa:bb:cc:dd:ee:ff

# Show mesh portal table (for bridging)
iw dev mesh0 mpp dump
```

---

## IBSS / Ad-Hoc Mode

```bash
# Create IBSS interface
iw phy phy0 interface add ibss0 type ibss
ip link set ibss0 up

# Join / create an IBSS cell
iw dev ibss0 ibss join "my-adhoc" 2437

# Join with HT40
iw dev ibss0 ibss join "my-adhoc" 5200 HT40+

# Join with fixed BSSID
iw dev ibss0 ibss join "my-adhoc" 2437 02:ca:fe:ba:be:00

# Leave IBSS
iw dev ibss0 ibss leave

# Show current IBSS info
iw dev ibss0 link
```

---

## P2P / Wi-Fi Direct

```bash
# Create P2P interfaces
iw phy phy0 interface add p2p-dev type p2p-device
iw phy phy0 interface add p2p-cl type p2p-client

# P2P is mostly managed by wpa_supplicant with p2p_* commands
# iw provides the low-level interface setup

# Start P2P device
ip link set p2p-dev up

# Find peers (trigger P2P scan)
# Use wpa_cli: p2p_find
# iw: iw dev p2p-dev scan triggers general scan visible to P2P stack
```

---

## WoWLAN (Wake on Wireless LAN)

```bash
# Show WoWLAN configuration and what the phy supports
iw phy phy0 wowlan show

# Enable wake on any packet
iw phy phy0 wowlan enable any

# Enable wake on magic packet (standard WoL)
iw phy phy0 wowlan enable magic_pkt

# Enable wake on disconnect
iw phy phy0 wowlan enable disconnect

# Enable wake on GTK rekey failure (security event)
iw phy phy0 wowlan enable gtk_rekey_failure

# Enable wake on EAP identity request
iw phy phy0 wowlan enable eap_identity_req

# Enable wake on 4-way handshake
iw phy phy0 wowlan enable 4way_handshake

# Enable wake on RFKILL release
iw phy phy0 wowlan enable rfkill_release

# Enable wake on TCP connection (if supported)
iw phy phy0 wowlan enable tcp \
    src 192.168.1.10 dst 192.168.1.1 \
    src_port 12345 dst_port 80

# Disable WoWLAN
iw phy phy0 wowlan disable
```

---

## Coalesce / Offload

```bash
# Show coalesce configuration (interrupt coalescing for Wi-Fi frames)
iw phy phy0 coalesce show

# Enable coalescing (group low-priority frames to reduce wake-ups)
iw phy phy0 coalesce enable \
    delay 100 \
    condition 1 \
    pkt-offset 0 pattern ffffffffffff    # Match broadcast

# Disable coalescing
iw phy phy0 coalesce disable
```

---

## Channel & Frequency Management

```bash
# Set channel (interface must not be associated)
iw dev wlan0 set channel 1
iw dev wlan0 set channel 6 HT20
iw dev wlan0 set channel 36 HT40+
iw dev wlan0 set channel 100 HT40+        # DFS channel — needs CAC

# Set frequency directly (MHz)
iw dev wlan0 set freq 2412               # Channel 1
iw dev wlan0 set freq 5180               # Channel 36
iw dev wlan0 set freq 5180 80MHz         # 80 MHz VHT
iw dev wlan0 set freq 5180 160MHz        # 160 MHz VHT (if supported)

# Center frequency for 80 MHz
iw dev wlan0 set freq 5180 80MHz 5210    # Primary 5180, center 5210

# Show all available channels (respects regdomain)
iw phy phy0 channels

# Show channels per band in detail
iw list | grep -A 40 "Band 1"            # 2.4 GHz
iw list | grep -A 40 "Band 2"            # 5 GHz

# Channel width modes
# 20MHz    — HT20 (safe, compatible)
# 40MHz    — HT40+ (secondary above) or HT40- (secondary below)
# 80MHz    — VHT / 802.11ac
# 80+80MHz — VHT with non-contiguous 80+80
# 160MHz   — VHT / 802.11ax

# DFS — check if a channel requires DFS (radar detection)
iw list | grep "MHz \[" | grep "radar detection"

# Trigger CAC manually (Channel Availability Check for DFS)
iw dev wlan0 cac trigger 5260 HT20
```

---

## Statistics & Debugging

```bash
# Interface counters (TX/RX packets, bytes, errors)
iw dev wlan0 get survey             # Survey data (noise, channel busy time)

# Survey dump — channel utilisation
iw dev wlan0 survey dump

# Key survey fields:
#   frequency:       Channel in MHz
#   noise:           Noise floor in dBm
#   channel active time:   Time (ms) radio was on channel
#   channel busy time:     Time medium was sensed busy (CCA)
#   channel receive time:  Time spent receiving
#   channel transmit time: Time spent transmitting

# Calculate channel utilisation (busy_time / active_time × 100)
iw dev wlan0 survey dump | awk '
  /active time:/ {active=$NF}
  /busy time:/   {busy=$NF}
  /frequency:/   {freq=$NF}
  /noise:/       {print freq, "MHz util:", int(busy/active*100)"%", "noise:", $NF, "dBm"}
'

# Link quality snapshot
iw dev wlan0 link

# Station-level counters (TX retries, failures, etc.)
iw dev wlan0 station dump

# Interface-level feature flags
iw dev wlan0 info

# Check number of active timers / connections
iw dev wlan0 station dump | grep -c "^Station"

# Antenna configuration
iw phy phy0 info | grep -E "antenna|Antenna"

# Set antenna bitmask (if driver supports it)
iw phy phy0 set antenna 3 3        # Enable antennas 1+2 for TX and RX (bitmask)
iw phy phy0 set antenna all        # Use all antennas
```

---

## Event Monitoring

```bash
# Watch for all wireless events in real time
iw event

# With timestamps
iw event -t

# With frame contents (very verbose)
iw event -f

# Filter specific events (use grep)
iw event -t | grep -E "connect|disconnect|scan"

# Watch for scan results becoming available
iw event -t | grep "scan results"

# Watch for association events
iw event -t | grep -E "connected|disconnected"

# Useful for scripts — wait for association
iw event | grep -m1 "connected to"

# Combine with logger
iw event -t >> /var/log/wifi-events.log &
```

---

## Hidden Gems & Power Tips

### 1. Verify 802.11ac/ax support before assuming it works

```bash
# Check HE (Wi-Fi 6 / 802.11ax) support
iw phy phy0 info | grep -A 20 "HE MAC Capabilities"
iw phy phy0 info | grep -A 20 "HE PHY Capabilities"

# Check VHT (Wi-Fi 5 / 802.11ac) MCS map
iw phy phy0 info | grep -A 10 "VHT RX MCS"

# Number of spatial streams from MCS map
iw phy phy0 info | grep "VHT Capabilities" -A 20 | grep "MCS"
```

### 2. Check actual negotiated connection details

```bash
# Full link info — tells you exactly what 802.11 features are active
iw dev wlan0 link

# Look for:
#   tx bitrate: 300.0 MBit/s MCS 15 40MHz short GI
#               ^^^^^^^^^^^  ^^^^^^  ^^^^^  ^^^^^^^^
#               PHY rate     MCS     Width  Guard Interval
```

### 3. Noise floor and channel quality script

```bash
#!/bin/bash
# Snapshot channel quality for all scanned APs
iw dev wlan0 scan | awk '
/^BSS /    { bss=$2 }
/SSID:/    { ssid=substr($0, index($0,$2)) }
/signal:/  { sig=$2 }
/freq:/    { freq=$2; print freq"MHz", sig"dBm", bss, ssid }
' | sort -k2 -n -r
```

### 4. Find hidden SSIDs

```bash
# Scan and show APs with empty SSID
iw dev wlan0 scan | grep -E "^BSS |SSID:" | awk '
  /^BSS/ { bss=$2 }
  /SSID:$/ { print "Hidden:", bss }
'
```

### 5. Identify Wi-Fi generation from scan

```bash
# Look at capabilities in scan output
iw dev wlan0 scan | awk '
  /^BSS/     { bss=$2; gen="802.11b/g" }
  /HT capa/  { gen="802.11n (Wi-Fi 4)" }
  /VHT capa/ { gen="802.11ac (Wi-Fi 5)" }
  /HE capa/  { gen="802.11ax (Wi-Fi 6)" }
  /EHT capa/ { gen="802.11be (Wi-Fi 7)" }
  /SSID:/    { print $2, bss, gen }
' | sort -u
```

### 6. Channel utilisation monitoring loop

```bash
# Monitor channel busy % every 5 seconds
while true; do
  echo -n "$(date +%T) "
  iw dev wlan0 survey dump | awk '
    /in use/ { inuse=1 }
    inuse && /active time:/ { active=$NF }
    inuse && /busy time:/   { busy=$NF }
    inuse && /noise:/       { printf "noise: %s dBm  util: %d%%\n", $NF, busy/active*100; inuse=0 }
  '
  sleep 5
done
```

### 7. Detect neighbouring APs on same channel (co-channel interference)

```bash
MY_CHAN=6
iw dev wlan0 scan | awk -v chan="$MY_CHAN" '
  /^BSS/  { bss=$2 }
  /SSID:/ { ssid=$2 }
  /DS Parameter set: channel/ && $NF==chan { print "Co-channel:", bss, ssid }
'
```

### 8. Export scan results as CSV

```bash
iw dev wlan0 scan | awk '
BEGIN { print "BSSID,SSID,Signal,Channel,Security" }
/^BSS /    { bss=$2; ssid=""; sig=""; ch=""; sec="Open" }
/SSID:/    { ssid=$2 }
/signal:/  { sig=$2 }
/DS Param/ { ch=$NF }
/WPA\|RSN/ { sec="WPA" }
/capability:.*Privacy/ { if(sec=="Open") sec="WEP" }
/^BSS /    { if(bss!="") print bss","ssid","sig","ch","sec }
END        { print bss","ssid","sig","ch","sec }
' > wifi_scan.csv
```

### 9. Quick check — is the driver nl80211 capable?

```bash
iw dev 2>&1 | grep -c "wlan"       # 0 = no nl80211 interfaces
iw list 2>&1 | grep "No such"       # Driver doesn't support nl80211

# Check loaded kernel modules
lsmod | grep -E "mac80211|cfg80211|nl80211"
```

### 10. Long-range / outdoor tweaks

```bash
# Increase ACK timeout for long-distance links (coverage class)
# 1 unit = ~450m; max is 31 (~13.9 km for 802.11a/n)
iw phy phy0 set coverage 10        # ~4.5 km

# Disable rate control and force lowest rate for max range
iw dev wlan0 set bitrates legacy-5 6   # 6 Mbps on 5 GHz

# Increase TX power to maximum allowed
iw dev wlan0 set txpower fixed 3000   # 30 dBm (check legal limit!)
```

### 11. Frame injection test

```bash
# Verify injection capability on monitor interface
# mon0 must be in monitor mode with correct channel set
iw dev mon0 set monitor active

# Use aireplay-ng or packetforge-ng to inject
# or use Scapy/Python for custom frames
```

### 12. Quickly compare two radios

```bash
diff <(iw phy phy0 info) <(iw phy phy1 info)
```

### 13. Grab all info in one shot for bug reports

```bash
{
  echo "=== iw dev ==="
  iw dev
  echo "=== iw list ==="
  iw list
  echo "=== iw reg get ==="
  iw reg get
  echo "=== link ==="
  iw dev wlan0 link
  echo "=== survey ==="
  iw dev wlan0 survey dump
  echo "=== stations ==="
  iw dev wlan0 station dump
} > wifi_debug_$(date +%Y%m%d_%H%M%S).txt
```

### 14. Check TDLS (Tunneled Direct Link Setup) support

```bash
# TDLS allows direct STA-to-STA frames without going through AP
iw phy phy0 info | grep -i tdls

# Operate on TDLS link
iw dev wlan0 tdls discover aa:bb:cc:dd:ee:ff
iw dev wlan0 tdls setup aa:bb:cc:dd:ee:ff
iw dev wlan0 tdls teardown aa:bb:cc:dd:ee:ff
```

---

## Quick Reference Card

| Goal | Command |
|------|---------|
| List all Wi-Fi interfaces | `iw dev` |
| List all physical radios | `iw phy` |
| Full radio capabilities | `iw phy phy0 info` |
| Scan for networks | `iw dev wlan0 scan` |
| Scan specific SSIDs | `iw dev wlan0 scan ssid "Name"` |
| Current connection info | `iw dev wlan0 link` |
| Connect (open only) | `iw dev wlan0 connect "SSID"` |
| Disconnect | `iw dev wlan0 disconnect` |
| Set TX power (auto) | `iw dev wlan0 set txpower auto` |
| Set TX power (fixed) | `iw dev wlan0 set txpower fixed 2000` |
| Enable power save | `iw dev wlan0 set power_save on` |
| Set channel | `iw dev wlan0 set channel 6 HT20` |
| Set regulatory domain | `iw reg set GB` |
| Show regulatory domain | `iw reg get` |
| Create monitor interface | `iw phy phy0 interface add mon0 type monitor` |
| Delete interface | `iw dev mon0 del` |
| Switch to monitor mode | `iw dev wlan0 set type monitor` |
| Show stations (AP mode) | `iw dev wlan0 station dump` |
| Show channel utilisation | `iw dev wlan0 survey dump` |
| Force bitrate (legacy) | `iw dev wlan0 set bitrates legacy-2.4 54` |
| Reset bitrate to auto | `iw dev wlan0 set bitrates` |
| Watch Wi-Fi events | `iw event -t` |
| Add mesh interface | `iw phy phy0 interface add mesh0 type mesh_point` |
| Join mesh | `iw dev mesh0 mesh join "my-mesh"` |
| Show mesh paths | `iw dev mesh0 mpath dump` |
| Coverage class (range) | `iw phy phy0 set coverage 10` |
| 4-address mode (WDS) | `iw dev wlan0 set 4addr on` |
| Show WoWLAN config | `iw phy phy0 wowlan show` |

---

## iw vs iwconfig Mapping

| Legacy `iwconfig`/`iwlist` | Modern `iw` equivalent |
|---------------------------|------------------------|
| `iwconfig wlan0` | `iw dev wlan0 info` + `iw dev wlan0 link` |
| `iwconfig wlan0 essid "SSID"` | `iw dev wlan0 connect "SSID"` |
| `iwconfig wlan0 freq 2.437G` | `iw dev wlan0 set freq 2437` |
| `iwconfig wlan0 channel 6` | `iw dev wlan0 set channel 6` |
| `iwconfig wlan0 txpower 20` | `iw dev wlan0 set txpower fixed 2000` |
| `iwconfig wlan0 power on` | `iw dev wlan0 set power_save on` |
| `iwconfig wlan0 rate 54M` | `iw dev wlan0 set bitrates legacy-2.4 54` |
| `iwconfig wlan0 mode monitor` | `iw dev wlan0 set type monitor` |
| `iwlist wlan0 scan` | `iw dev wlan0 scan` |
| `iwlist wlan0 frequency` | `iw phy phy0 channels` |
| `iwlist wlan0 bitrate` | `iw phy phy0 info \| grep Bitrates` |
| `iwlist wlan0 ap` | `iw dev wlan0 scan` (with BSSID in results) |
| `iwlist wlan0 power` | `iw dev wlan0 get power_save` |
| `iwevent` | `iw event -t` |
| `iwspy` | No direct equivalent (nl80211 doesn't support spy) |

---

## Companion Tools

| Tool | Purpose |
|------|---------|
| `wpa_supplicant` / `wpa_cli` | WPA2/WPA3 authentication (required for encrypted networks) |
| `hostapd` | Software AP / hotspot daemon |
| `rfkill` | Block/unblock radio kill switch (`rfkill list`, `rfkill unblock wifi`) |
| `iwmon` | Dump raw nl80211 netlink messages |
| `wavemon` | ncurses Wi-Fi monitor with signal graphs |
| `airmon-ng` | Easy monitor mode from Aircrack-ng suite |
| `tcpdump` / `tshark` | Capture frames in monitor mode |
| `crda` | Central Regulatory Domain Agent (older kernels) |
| `iw`  | This tool — kernel ≥ 2.6.33, iproute2-style |

---

## Further Reading

- `man iw` — full manual
- `iw help` — built-in command list
- `iw dev wlan0 <command> help` — per-command help
- [Linux Wireless wiki](https://wireless.wiki.kernel.org/en/users/documentation/iw)
- [cfg80211 kernel docs](https://www.kernel.org/doc/html/latest/driver-api/80211/cfg80211.html)
- [nl80211 header](https://github.com/torvalds/linux/blob/master/include/uapi/linux/nl80211.h) — authoritative source for all attributes
- [hostapd / wpa_supplicant](https://w1.fi/wpa_supplicant/)

---

*Tested with iw 5.x–6.x on Linux kernel 5.x–6.x. Some commands (HE/EHT, 6 GHz) require kernel ≥ 5.4 and hardware support. Always respect local regulatory requirements when adjusting TX power or using DFS channels.*
