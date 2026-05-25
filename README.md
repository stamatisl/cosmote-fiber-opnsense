[README.md](https://github.com/user-attachments/files/28231036/README.md)
# Cosmote Fiber + OPNsense: Complete Setup Guide

> Replace your ISP-provided router with OPNsense on a mini PC, using a GPON SFP stick to connect directly to Cosmote's fiber network (Nokia OLT). This guide covers internet, WiFi (UniFi), VoIP, and IPv6.

---

## Hardware

| Device | Model | Role |
|---|---|---|
| Firewall/Router | Topton N300 (4x 2.5G + 1x SFP+) | OPNsense host |
| GPON Stick | FS.com GPON-SFP-ONT-MAC-I | Fiber ONT |
| Access Point | Ubiquiti UniFi U7-Pro | WiFi |
| UniFi Controller | Mini PC (N95 or similar) | Dedicated controller |
| Telephony | Fritz!Box 5530 Fiber | VoIP only |

**Why the Topton N300?** It has a 10G SFP+ port for the GPON stick, 4x 2.5G Ethernet for flexible LAN/WAN topology, and enough CPU for PPPoE at full 940Mbps line rate.

---

## Network Topology

```
Internet (Cosmote Fiber)
        │
  GPON Stick (SFP+ / ix0)     ← management: 192.168.1.10
        │
  ix0_vlan835 (VLAN 835)
        │
     PPPoE (pppoe0)  ──────────────────── WAN IP + IPv6 prefix
        │
   OPNsense (Topton N300)
        │
   bridge0 (10.0.68.1/24) ─── DHCP: 10.0.68.100–200 ─── IPv6 SLAAC
   ┌────┴────┐
  igc0     igc1
  (LAN)   (WIFI)
             │
         UniFi U7-Pro ──── WiFi clients

  igc2 (192.168.178.2/24)
        │
   Fritz!Box WAN port
   Fritz!Box LAN (192.168.178.1)
        │
   Analog phones / VoIP
```

### Fritz!Box Note

The Fritz!Box is configured in **router mode** (not modem/bridge mode), connected via its WAN port to `igc2` of the OPNsense. The OPNsense acts as the upstream gateway for the Fritz subnet (`192.168.178.x`). The Fritz!Box handles only VoIP — it does not manage internet access.

> ⚠️ **Warning:** If you factory reset the Fritz!Box, it will attempt to switch back to modem/fiber mode and may lock you out of its UI with an unknown password. The safest approach is to leave it as-is once VoIP is working.

---

## Part 1: GPON Stick Configuration

### 1.1 Finding the Correct GPON Serial Number

> ⚠️ **Critical:** The S/N printed on the Fritz!Box label is NOT the GPON S/N registered at Cosmote's Nokia OLT.

Find the real GPON S/N from the Fritz!Box web UI **before** disconnecting it:

```
http://fritz.box → System → FRITZ!Box Details → PON serial number
```

Example: `AVMG725730D2`

### 1.2 SSH Access to the GPON Stick

The FS GPON stick runs an SSH server at `192.168.1.10`:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
    -oHostKeyAlgorithms=+ssh-rsa \
    ONTUSER@192.168.1.10
# Default password: 7sp!lwUBz1 — change this after setup
```

### 1.3 Spoofing the GPON Serial Number

```bash
# Set the Fritz!Box GPON S/N
set_serial_number AVMG725730D2

# Vendor/device info (matching Fritz!Box values)
sfp_i2c -i0 -s "AVM"
sfp_i2c -i1 -s "fhm5820la"
sfp_i2c -i3 -s "AB22062209633"   # physical label S/N
sfp_i2c -i7 -s "F!box5530"
sfp_i2c -i8 -s "AB22062209633"

# Management IP
fw_setenv ipaddr 192.168.1.10
fw_setenv gatewayip 192.168.1.1

reboot
```

> **Note:** `sfp_i2c` on this firmware is write-only — reading back returns no output. This is expected.

### 1.4 Verify OLT Registration

After reboot, confirm the stick has registered:

```bash
gtop -b -g a | grep "PLOAM state"
# Must show: PLOAM state: 5
```

If not 5, the S/N does not match what is registered at Cosmote's OLT. Double-check the value from the Fritz!Box UI.

---

## Part 2: OPNsense — Interfaces

### 2.1 WAN Interface (ix0)

```
Interfaces → WAN:
  Interface:  ix0
  IPv4:       Static — 192.168.1.1/24  ← for GPON stick management
  IPv6:       None
```

### 2.2 VLAN 835

```
Interfaces → Other Types → VLAN:
  Parent:      ix0
  VLAN Tag:    835
  Description: COSMOTE_VLAN835
```

### 2.3 PPPoE — WAN_COSMOTE

```
Interfaces → WAN_COSMOTE (pppoe0):
  Interface:               ix0_vlan835
  Type:                    PPPoE
  Username:                <username>@otenet.gr
  Password:                <password>
  IPv6:                    DHCPv6
  Prefix Delegation Size:  56
  Send Prefix Hint:        ✅
```

> Cosmote does **not** use a PLOAM password — leave it empty.

### 2.4 LAN (igc0) and WIFI (igc1)

Both are bridge members — no IP assigned directly:

```
Interfaces → LAN (igc0):   No IP
Interfaces → WIFI (igc1):  No IP
```

### 2.5 Bridge — BRIDGE_LAN_WIFI

```
Interfaces → Other Types → Bridge:
  Members:     LAN (igc0), WIFI (igc1)
  Description: BRIDGE_LAN_WIFI

Interfaces → BRIDGE_LAN_WIFI (bridge0):
  IPv4:             Static — 10.0.68.1/24
  IPv6:             Track Interface
    Interface:      WAN_COSMOTE (opt1)
    Prefix ID:      0
```

### 2.6 Fritz!Box Interface (igc2)

```
Interfaces → FRITZBOX (igc2):
  IPv4: Static — 192.168.178.2/24
```

---

## Part 3: DHCP

### 3.1 IPv4 — Dnsmasq

```
Services → Dnsmasq → Settings:
  Enable:      ✅
  Interface:   BRIDGE_LAN_WIFI
  Enable RA:   ✅

Services → Dnsmasq → DHCP Ranges:
  Interface:   BRIDGE_LAN_WIFI
  Start:       10.0.68.100
  End:         10.0.68.200
  Constructor: BRIDGE_LAN_WIFI
```

### 3.2 IPv6 — SLAAC

Add an IPv6 range to Dnsmasq:

```
Services → Dnsmasq → DHCP Ranges (add new):
  Interface:      BRIDGE_LAN_WIFI
  Start:          ::1000
  End:            ::2000
  Constructor:    BRIDGE_LAN_WIFI
  Prefix Length:  64
  RA Mode:        slaac
```

### 3.3 Early Shell Command (bridge0 IPv6 fix)

Due to an OPNsense quirk with bridges, add this to:

```
System → Settings → Miscellaneous → Pre-config shell commands:
  ifconfig bridge0 inet6 auto_linklocal
```

This ensures bridge0 has a link-local IPv6 address before dhcp6c starts.

---

## Part 4: Firewall

### 4.1 LAN/Bridge Rules

```
Firewall → Rules → BRIDGE_LAN_WIFI:
  Action:      Pass
  Protocol:    TCP/UDP
  Source:      any
  Destination: any
  IPv4+IPv6:   ✅
```

### 4.2 Fritz!Box (VoIP) Rules

```
Firewall → Rules → FRITZBOX:
  Action:      Pass
  Source:      any
  Destination: any
```

### 4.3 Disable SIP ALG

> ⚠️ This is required for VoIP to work reliably. Without it, SIP packets get mangled by the firewall and calls show as "busy" even when the line is free.

```
System → Settings → Miscellaneous → Firewall:
  Disable SIP proxy: ✅
```

### 4.4 Firewall Optimization

```
System → Settings → Miscellaneous → Firewall:
  Firewall Optimization: conservative
```

`conservative` keeps UDP sessions alive longer — critical for SIP registration stability.

---

## Part 5: NAT for Fritz!Box VoIP

This is the key fix for VoIP reliability. Without static port NAT, the OPNsense may remap SIP ports, causing calls to fail or show as busy.

```
Firewall → NAT → Outbound → Mode: Hybrid

Add rule:
  Source:       192.168.178.0/24  (Fritz!Box subnet)
  Destination:  any
  Interface:    WAN_COSMOTE
  Static port:  ✅
  Protocol:     TCP/UDP
```

**Static port** tells OPNsense to preserve the original source port when NATting SIP traffic. Without this, Cosmote's SIP servers may reject re-registration attempts after the NAT mapping changes.

---

## Part 6: UniFi Access Point

### 6.1 Dedicated Controller

Run the UniFi controller on a separate mini PC (e.g. N95) rather than on the OPNsense itself:

```bash
# Ubuntu/Debian — Docker
curl -fsSL https://get.docker.com | sh
docker run -d \
  --name unifi \
  --network host \
  -v /opt/unifi:/config \
  lscr.io/linuxserver/unifi-network-application:latest
```

Controller UI: `https://<N95-IP>:8443`

### 6.2 Physical Connection

```
U7-Pro (PoE) ──── igc1 (WIFI interface) ──── bridge0 ──── LAN
```

### 6.3 Adoption

If the AP doesn't appear automatically in the controller:

```bash
# SSH to AP (default credentials: ubnt/ubnt or root/ubnt)
ssh admin@<AP-IP>
set-inform http://<controller-IP>:8080/inform
```

---

## Part 7: VoIP — Fritz!Box

### 7.1 Architecture

```
Cosmote SIP Servers
        │ (internet)
   OPNsense WAN
        │
   OPNsense (NAT with static port)
        │
   igc2 (192.168.178.2)
        │
   Fritz!Box WAN (192.168.178.1)
   Fritz!Box LAN
        │
   Analog phones
```

### 7.2 Fritz!Box Mode

The Fritz!Box runs in **router mode** (not bridge/modem mode). It connects its WAN port to `igc2` of the OPNsense and receives internet connectivity through NAT. The Fritz handles VoIP registration with Cosmote's SIP servers internally.

> **Important:** Do not factory reset the Fritz!Box once it is working. Resetting it may cause it to revert to fiber/modem mode, which will present an unknown admin password and require re-configuration. Leave it as-is.

### 7.3 VoIP Troubleshooting

If you lose dial tone or get a busy signal on all calls:

1. Confirm **SIP ALG is disabled** in OPNsense
2. Confirm **static port NAT** is in place for `192.168.178.0/24`
3. Confirm **firewall optimization** is set to `conservative`
4. Reboot the Fritz!Box from `http://192.168.178.1`

---

## Part 8: IPv6

### 8.1 Cosmote IPv6 Details

| Parameter | Value |
|---|---|
| WAN prefix | /56 (DHCPv6-PD) |
| LAN prefix | /64 (delegated to bridge0) |
| Mode | Stateless (SLAAC) |
| PD length config | 56 on WAN_COSMOTE |

### 8.2 Verify

```bash
# WAN IPv6 (link-local + global)
ifconfig pppoe0 | grep inet6

# LAN IPv6 (must be /64, not tentative)
ifconfig bridge0 | grep inet6

# DHCPv6 client config
cat /var/etc/dhcp6c.conf
```

The `dhcp6c.conf` must contain a `prefix-interface bridge0` stanza:

```
id-assoc pd 3 {
  prefix ::/64 infinity;
  prefix-interface bridge0 {
    sla-id 0;
    sla-len 0;
  };
};
```

If it's missing, go to Interfaces → BRIDGE_LAN_WIFI → set IPv6 to **Track Interface**, pointing to WAN_COSMOTE, then Save & Apply.

### 8.3 Test

```bash
# From a client
ping6 google.com

# Online
# https://test-ipv6.com
```

---

## Part 9: AdGuard Home

### 9.1 Installation

```
System → Firmware → Plugins → os-adguardhome-maxit → Install
```

### 9.2 Configuration

```
Services → AdGuard Home:
  Enable:    ✅
  Interface: 10.0.68.1
  Port:      3000 (UI), 53 (DNS)
```

Point Dnsmasq upstream to AdGuard:

```
Services → Dnsmasq → Upstream DNS: 127.0.0.1#53
```

### 9.3 Recommended Block Lists

From AdGuard Home UI at `http://10.0.68.1:3000`:

- AdGuard DNS filter
- EasyList
- Peter Lowe's Ad and tracking server list

---

## Troubleshooting Reference

### GPON — PLOAM state not 5

```bash
onu gtcsng               # verify programmed S/N
debug && cat /tmp/log/one_click   # full registration log
```

The S/N in the stick must exactly match what Cosmote has registered at the OLT (from Fritz!Box UI → System → Details → PON serial number).

### PPPoE — Not connecting

```bash
tcpdump -i ix0_vlan835 -n pppoes   # look for PADI sent, PADO received
clog /var/log/pppos.log | tail -50
```

If you see PADI but no PADO, the GPON stick is not registered (PLOAM ≠ 5).

### Lost OPNsense web access

Connect monitor + keyboard to the N300, then from the console menu:

```
Option 2 → Set interface IP → bridge0 → 10.0.68.1/24
```

### UniFi AP stuck on "Adopting"

```bash
ssh admin@<AP-IP>
set-inform http://<controller-IP>:8080/inform
```

### VoIP — Busy tone on all calls

1. SIP ALG disabled? → System → Settings → Firewall → Disable SIP proxy ✅
2. Static port NAT in place? → Firewall → NAT → Outbound → check rule
3. Firewall optimization → conservative
4. Reboot Fritz!Box

---

## Quick Command Reference

```bash
# GPON stick
gtop -b -g a | grep "PLOAM state"   # registration (need 5)
onu gtcsng                           # programmed S/N

# PPPoE
tcpdump -i ix0_vlan835 -n pppoes    # PPPoE handshake
clog /var/log/pppos.log | tail -50  # PPPoE daemon log

# Network state
ifconfig bridge0                     # bridge status + IPs
ifconfig pppoe0 | grep inet6        # WAN IPv6
netstat -6 -rn | grep default       # IPv6 default route
cat /var/etc/dhcp6c.conf            # DHCPv6 client config
cat /usr/local/etc/dnsmasq.conf     # Dnsmasq config

# Services
service dnsmasq onerestart
service dhcp6c onerestart
service radvd onerestart
/usr/local/sbin/pluginctl dns       # reconfigure DNS stack
```

---

## Services & Ports

| Service | Port | Interface |
|---|---|---|
| OPNsense Web UI | 443/tcp | bridge0 |
| OPNsense SSH | 22/tcp | bridge0 |
| AdGuard Home UI | 3000/tcp | bridge0 |
| DNS | 53 | bridge0 |
| DHCP | 67-68/udp | bridge0 |
| DHCPv6 | 547/udp | bridge0 |
| UniFi Controller | 8443/tcp | LAN |
| GPON Stick | 80/22 | 192.168.1.10 |
| Fritz!Box | 80/443 | 192.168.178.1 |

---

## Key Notes

- The GPON stick retains the programmed S/N across reboots (stored in flash via `fw_setenv`).
- Cosmote uses **Nokia OLT**, **VLAN 835** for PPPoE tagging, and PAP authentication.
- Cosmote does **not** use a PLOAM password — leave it blank.
- The Fritz!Box physical label S/N ≠ PON S/N. Always get the PON S/N from the Fritz web UI.
- For future XGS-PON upgrade: the N300's ix0 port supports 10G — only the SFP stick needs replacing.
- Privacy benefit: OPNsense eliminates ISP TR-069 remote management, DNS logging, and hidden admin accounts that are present on all ISP-provided routers.

---

*Based on a real installation — Cosmote Fiber 1Gbps, Nokia OLT R6.7.01, OPNsense 25.x, UniFi U7-Pro, Fritz!Box 5530 Fiber*
