# ModemManager Cheatsheet (Linux)

> **ModemManager** is a DBus-activated daemon that controls mobile broadband (2G/3G/4G/5G) devices and connections on Linux. It works alongside NetworkManager and is the standard interface for managing modems.

---

## Table of Contents

1. [Installation](#installation)
2. [Service Management](#service-management)
3. [mmcli Basics](#mmcli-basics)
4. [Listing & Inspecting Modems](#listing--inspecting-modems)
5. [SIM Card Operations](#sim-card-operations)
6. [Enabling & Connecting](#enabling--connecting)
7. [Bearer Management](#bearer-management)
8. [SMS Messaging](#sms-messaging)
9. [USSD (Unstructured Supplementary Service Data)](#ussd)
10. [Location & GPS](#location--gps)
11. [Modem Firmware & Info](#modem-firmware--info)
12. [Signal Monitoring](#signal-monitoring)
13. [Debugging & Logs](#debugging--logs)
14. [DBus Direct Access](#dbus-direct-access)
15. [NetworkManager Integration](#networkmanager-integration)
16. [Common Use Cases](#common-use-cases)
17. [Troubleshooting](#troubleshooting)

---

## Installation

```bash
# Debian / Ubuntu
sudo apt install modemmanager

# Fedora / RHEL / CentOS
sudo dnf install ModemManager

# Arch Linux
sudo pacman -S modemmanager

# openSUSE
sudo zypper install ModemManager

# Check version
mmcli --version
```

---

## Service Management

```bash
# Start ModemManager
sudo systemctl start ModemManager

# Stop ModemManager
sudo systemctl stop ModemManager

# Restart ModemManager
sudo systemctl restart ModemManager

# Enable at boot
sudo systemctl enable ModemManager

# Disable at boot
sudo systemctl disable ModemManager

# Check service status
sudo systemctl status ModemManager

# Check if running
systemctl is-active ModemManager
```

---

## mmcli Basics

`mmcli` is the command-line interface for ModemManager.

```bash
# General help
mmcli --help
mmcli --help-all

# List all available modems
mmcli -L
mmcli --list-modems

# Monitor ModemManager events (live)
mmcli -m <index> --monitor

# Verbose output (show all properties)
mmcli -m <index> -v
```

> **Note:** `<index>` refers to the modem index number shown in `mmcli -L` (e.g., `0`, `1`). You can also use the full DBus path (e.g., `/org/freedesktop/ModemManager1/Modem/0`).

---

## Listing & Inspecting Modems

```bash
# List all modems (short)
mmcli -L

# Full modem info (modem index 0)
mmcli -m 0

# Specific modem by DBus path
mmcli -m /org/freedesktop/ModemManager1/Modem/0

# Show all modem properties verbosely
mmcli -m 0 -v

# Show modem capabilities
mmcli -m 0 --capabilities

# Show current access technology
mmcli -m 0 | grep "access tech"

# Show modem state
mmcli -m 0 | grep "state"
```

**Common modem states:**
| State | Description |
|---|---|
| `disabled` | Modem is present but not enabled |
| `enabling` | Modem is being enabled |
| `enabled` | Modem is enabled, searching for network |
| `searching` | Searching for a network |
| `registered` | Registered on a network |
| `connecting` | Establishing data connection |
| `connected` | Data connection active |
| `disconnecting` | Tearing down connection |
| `failed` | Modem encountered an error |

---

## SIM Card Operations

```bash
# List SIMs for modem 0
mmcli -m 0 --sim-slot-status

# Get SIM info (SIM index 0)
mmcli -i 0
mmcli --sim=0

# Show SIM ICCID, IMSI, operator
mmcli -i 0 | grep -E "iccid|imsi|operator"

# Unlock SIM with PIN
mmcli -i 0 --pin=1234

# Unlock SIM with PUK (when PIN locked)
mmcli -i 0 --puk=12345678 --pin=1234

# Change PIN
mmcli -i 0 --change-pin=1234 --new-pin=5678

# Enable PIN lock
mmcli -i 0 --enable-pin --pin=1234

# Disable PIN lock
mmcli -i 0 --disable-pin --pin=1234
```

---

## Enabling & Connecting

```bash
# Enable modem (power on radio)
mmcli -m 0 --enable

# Disable modem (power off radio)
mmcli -m 0 --disable

# Simple connect (using default APN settings)
mmcli -m 0 --simple-connect="apn=internet"

# Connect with username/password
mmcli -m 0 --simple-connect="apn=internet,user=myuser,password=mypass"

# Connect with specific IP type
mmcli -m 0 --simple-connect="apn=internet,ip-type=ipv4"
mmcli -m 0 --simple-connect="apn=internet,ip-type=ipv6"
mmcli -m 0 --simple-connect="apn=internet,ip-type=ipv4v6"

# Connect with specific allowed modes (e.g. force 4G)
mmcli -m 0 --simple-connect="apn=internet,allowed-auth=pap|chap"

# Disconnect
mmcli -m 0 --simple-disconnect

# Reset modem
mmcli -m 0 --reset

# Factory reset
mmcli -m 0 --factory-reset=<unlock-code>
```

**Allowed auth values:** `none`, `pap`, `chap`, `mschap`, `mschapv2`, `eap`

---

## Bearer Management

Bearers represent a configured data connection (PDP context).

```bash
# List bearers for modem 0
mmcli -m 0 --list-bearers

# Show bearer details (bearer index 0)
mmcli -b 0
mmcli --bearer=0

# Create a new bearer
mmcli -m 0 --create-bearer="apn=internet"

# Create bearer with auth
mmcli -m 0 --create-bearer="apn=internet,user=foo,password=bar,allowed-auth=chap"

# Connect a specific bearer
mmcli -b 0 --connect

# Disconnect a specific bearer
mmcli -b 0 --disconnect

# Delete a bearer
mmcli -m 0 --delete-bearer=<bearer-dbus-path>
```

**Bearer properties:**
| Property | Description |
|---|---|
| `apn` | Access Point Name |
| `ip-type` | `ipv4`, `ipv6`, `ipv4v6` |
| `user` | Authentication username |
| `password` | Authentication password |
| `allowed-auth` | Auth protocol(s) |
| `number` | Dial-up number (legacy) |

---

## SMS Messaging

```bash
# List SMS messages (modem 0)
mmcli -m 0 --messaging-list-sms

# Read a specific SMS (index 0)
mmcli -s 0
mmcli --sms=0

# Create a new SMS message
mmcli -m 0 --messaging-create-sms="number=+441234567890,text=Hello"

# Send a created SMS (sms index 0)
mmcli -s 0 --send

# Create and send immediately
mmcli -m 0 --messaging-create-sms="number=+441234567890,text=Hello" 
# Note the returned SMS path, then:
mmcli -s <index> --send

# Delete an SMS
mmcli -m 0 --messaging-delete-sms=<sms-dbus-path>

# Get SMS store status
mmcli -m 0 --messaging-status
```

---

## USSD

USSD allows interaction with carrier services (e.g. balance checks).

```bash
# Initiate a USSD session
mmcli -m 0 --3gpp-ussd-initiate="*100#"

# Send response during active USSD session
mmcli -m 0 --3gpp-ussd-respond="1"

# Cancel active USSD session
mmcli -m 0 --3gpp-ussd-cancel

# Get USSD status
mmcli -m 0 --3gpp-ussd-status
```

---

## Location & GPS

```bash
# Show location capabilities
mmcli -m 0 --location-status

# Enable GPS (NMEA)
mmcli -m 0 --location-enable-gps-nmea

# Enable GPS (raw)
mmcli -m 0 --location-enable-gps-raw

# Enable 3GPP location (cell-based)
mmcli -m 0 --location-enable-3gpp

# Get current location
mmcli -m 0 --location-get

# Disable all location services
mmcli -m 0 --location-disable-gps-nmea
mmcli -m 0 --location-disable-gps-raw
mmcli -m 0 --location-disable-3gpp

# Enable GPS signal refresh interval (seconds)
mmcli -m 0 --location-set-gps-refresh-rate=10
```

---

## Modem Firmware & Info

```bash
# Show firmware info
mmcli -m 0 --firmware-list

# Show current firmware
mmcli -m 0 | grep firmware

# Select firmware
mmcli -m 0 --firmware-select=<unique-id>

# Show hardware info
mmcli -m 0 | grep -E "manufacturer|model|revision|device identifier"

# Show IMEI
mmcli -m 0 | grep "equipment id"

# Show hardware revision
mmcli -m 0 | grep "hw revision"

# Show plugin being used
mmcli -m 0 | grep plugin
```

---

## Signal Monitoring

```bash
# Get current signal quality (percentage)
mmcli -m 0 --signal-quality

# Get detailed signal info (RSSI, RSRP, RSRQ, SNR, etc.)
mmcli -m 0 --signal-get

# Enable periodic signal updates (every N seconds)
mmcli -m 0 --signal-setup=5

# Disable signal updates
mmcli -m 0 --signal-setup=0
```

**Signal metrics by technology:**
| Technology | Metrics |
|---|---|
| GSM/2G | RSSI |
| UMTS/3G | RSSI, RSCP, Ec/Io |
| LTE/4G | RSSI, RSRQ, RSRP, SNR |
| NR/5G | RSRP, RSRQ, SNR |

---

## Debugging & Logs

```bash
# Run ModemManager in foreground with debug output
sudo /usr/sbin/ModemManager --debug

# View logs via journald
journalctl -u ModemManager
journalctl -u ModemManager -f          # Follow live
journalctl -u ModemManager -n 100      # Last 100 lines
journalctl -u ModemManager --since "1 hour ago"

# View logs with timestamps
journalctl -u ModemManager -o short-iso

# Enable debug logging at runtime
sudo mmcli --set-logging=DEBUG

# Set logging back to info
sudo mmcli --set-logging=INFO

# Log levels: ERR, WARN, INFO, DEBUG

# Check loaded plugins
mmcli -G DEBUG 2>&1 | grep plugin
```

---

## DBus Direct Access

ModemManager exposes its API over DBus at `org.freedesktop.ModemManager1`.

```bash
# Introspect the ModemManager DBus service
gdbus introspect --system \
  --dest org.freedesktop.ModemManager1 \
  --object-path /org/freedesktop/ModemManager1

# List managed modem objects
gdbus call --system \
  --dest org.freedesktop.ModemManager1 \
  --object-path /org/freedesktop/ModemManager1 \
  --method org.freedesktop.DBus.ObjectManager.GetManagedObjects

# Get modem properties via DBus
gdbus call --system \
  --dest org.freedesktop.ModemManager1 \
  --object-path /org/freedesktop/ModemManager1/Modem/0 \
  --method org.freedesktop.DBus.Properties.GetAll \
  org.freedesktop.ModemManager1.Modem

# Scan for devices
gdbus call --system \
  --dest org.freedesktop.ModemManager1 \
  --object-path /org/freedesktop/ModemManager1 \
  --method org.freedesktop.ModemManager1.ScanDevices

# Use dbus-monitor to watch ModemManager signals
dbus-monitor --system "sender=org.freedesktop.ModemManager1"
```

---

## NetworkManager Integration

ModemManager works with NetworkManager for managing mobile connections.

```bash
# List mobile broadband connections
nmcli connection show

# Create a mobile broadband connection
nmcli connection add \
  type gsm \
  ifname "*" \
  con-name "Mobile" \
  apn "internet"

# Create connection with auth
nmcli connection add \
  type gsm \
  ifname "*" \
  con-name "Mobile" \
  apn "internet" \
  gsm.username "myuser" \
  gsm.password "mypass"

# Connect
nmcli connection up "Mobile"

# Disconnect
nmcli connection down "Mobile"

# Edit APN of existing connection
nmcli connection modify "Mobile" gsm.apn "newapn"

# Show mobile connection details
nmcli connection show "Mobile"

# Delete a connection
nmcli connection delete "Mobile"
```

---

## Common Use Cases

### Check if modem is detected
```bash
mmcli -L
lsusb | grep -i modem
lsusb | grep -i huawei    # For Huawei modems
ls /dev/ttyUSB* /dev/ttyACM*
```

### Quick connect to mobile data
```bash
sudo mmcli -m 0 --enable
sudo mmcli -m 0 --simple-connect="apn=YOUR_APN"
# Bring up the network interface manually if needed:
sudo dhclient <interface>
```

### Check mobile data IP
```bash
mmcli -b 0 | grep address
ip addr show
```

### Force 4G/LTE only
```bash
mmcli -m 0 --set-allowed-modes="4g"
# Modes: 2g, 3g, 4g, any, 2g|3g, 3g|4g, etc.
```

### Force 5G preferred
```bash
mmcli -m 0 --set-allowed-modes="5gnr|4g"
mmcli -m 0 --set-preferred-mode="5gnr"
```

### Send a quick SMS
```bash
SMS=$(mmcli -m 0 --messaging-create-sms="number=+441234567890,text=Hello from Linux")
SMS_INDEX=$(echo $SMS | grep -oP '\d+(?= successfully)')
mmcli -s $SMS_INDEX --send
```

### Watch signal live
```bash
watch -n 2 'mmcli -m 0 --signal-get'
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `mmcli -L` shows no modems | Check `lsusb`; restart ModemManager; check udev rules |
| Modem stuck in `disabled` | Run `mmcli -m 0 --enable`; check SIM is inserted |
| PIN error | Verify PIN with `mmcli -i 0 --pin=<pin>`; check PUK if needed |
| Connection fails | Verify APN; check signal quality; try `--simple-disconnect` then reconnect |
| No IP after connect | Run `dhclient <iface>` or check NetworkManager integration |
| Device not recognised | Install modem-specific firmware or `usb-modeswitch` for USB modems |
| Bearer shows connected but no internet | Check DNS (`/etc/resolv.conf`); verify routing table (`ip route`) |
| `org.freedesktop.ModemManager1` not found | Ensure service is running: `systemctl start ModemManager` |
| Modem disappears randomly | Check USB power management: `echo on > /sys/bus/usb/devices/<id>/power/control` |

```bash
# Useful diagnostic commands bundle
echo "=== ModemManager Status ===" && systemctl status ModemManager
echo "=== Modems ===" && mmcli -L
echo "=== USB Devices ===" && lsusb
echo "=== Serial Ports ===" && ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
echo "=== Kernel Modules ===" && lsmod | grep -E "cdc|option|qmi|mbim"
```

---

## Quick Reference Card

| Task | Command |
|---|---|
| List modems | `mmcli -L` |
| Modem info | `mmcli -m 0` |
| Enable modem | `mmcli -m 0 --enable` |
| Disable modem | `mmcli -m 0 --disable` |
| Connect | `mmcli -m 0 --simple-connect="apn=internet"` |
| Disconnect | `mmcli -m 0 --simple-disconnect` |
| SIM info | `mmcli -i 0` |
| Unlock PIN | `mmcli -i 0 --pin=1234` |
| Signal quality | `mmcli -m 0 --signal-get` |
| Send SMS | `mmcli -m 0 --messaging-create-sms="number=+44...,text=Hi"` |
| List SMS | `mmcli -m 0 --messaging-list-sms` |
| GPS on | `mmcli -m 0 --location-enable-gps-nmea` |
| Debug logs | `sudo mmcli --set-logging=DEBUG` |
| Live logs | `journalctl -u ModemManager -f` |
| Reset modem | `mmcli -m 0 --reset` |

---

*ModemManager project: https://www.freedesktop.org/wiki/Software/ModemManager/*  
*API docs: https://www.freedesktop.org/software/ModemManager/doc/latest/*
