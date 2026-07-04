# Project Sentinel — Enterprise Network Security Lab

> A dual-environment enterprise network security lab built entirely with open-source and free tools, demonstrating real-world network security architecture, centralized AAA/RADIUS authentication, GRE tunneling, perimeter firewalling, and DMZ segmentation across two simulated sites.

---

## Table of Contents

- [Overview](#overview)
- [Tools and Technologies](#tools-and-technologies)
- [Network Topology](#network-topology)
- [IP Addressing Scheme](#ip-addressing-scheme)
- [VLAN Design](#vlan-design)
- [Security Features Implemented](#security-features-implemented)
- [Configuration Highlights](#configuration-highlights)
- [AAA / RADIUS Architecture](#aaa--radius-architecture)
- [GRE Tunnel (Site-to-Site)](#gre-tunnel-site-to-site)
- [Packet Tracer Limitations and Production Notes](#packet-tracer-limitations-and-production-notes)
- [Verification Evidence](#verification-evidence)
- [Skills Demonstrated](#skills-demonstrated)
- [How to Reproduce](#how-to-reproduce)
- [Author](#author)

---

## Overview

Project Sentinel is a hands-on enterprise network security lab that simulates a two-site corporate network complete with a perimeter firewall, DMZ, centralized RADIUS authentication, site-to-site GRE tunneling, and perimeter ACLs. The project was built entirely using free, open-source tools — Cisco Packet Tracer and Oracle VirtualBox — with no paid subscriptions required.

The goal was to design and configure a network that reflects real enterprise security principles:

- **Defense in depth** — multiple security layers so no single failure compromises the entire network
- **Least privilege** — VLAN segmentation ensures devices only communicate with what they need
- **Centralized access control** — RADIUS authentication means device credentials are managed from one place and are immediately revocable
- **Management plane security** — network devices themselves are treated as attack surfaces, not just the traffic passing through them

---

## Tools and Technologies

| Tool | Purpose | Cost |
|------|---------|------|
| Cisco Packet Tracer 8.x | Network simulation — routers, switches, ASA firewall | Free (Cisco NetAcad) |
| Oracle VirtualBox | VM hypervisor for live security stack | Free |
| Security Onion | IDS/NSM platform — Suricata, Zeek, Kibana | Free/Open-source |
| draw.io | Network topology diagrams | Free |

**Cisco IOS devices used:**
- Cisco 2901 Router (ISP-R, R1-HQ, R2-BR)
- Cisco ASA 5505 Firewall (ASA-HQ)
- Cisco Catalyst 3650 Layer 3 Switch (MSW1-HQ)
- Cisco Catalyst 2960 Layer 2 Switch (SW-BR)

---

## Network Topology

```
                          [ISP-R]
                        203.0.113.2 / 198.51.100.2
                         /              \
                    (WAN)                (WAN)
                 203.0.113.1          198.51.100.1
                  [R1-HQ]  <====GRE====>  [R2-BR]
               Edge router/VPN         Edge router/VPN
                    |                       |
               10.0.1.1                  G0/1 trunk
                    |                       |
               10.0.1.2                  [SW-BR]
                [ASA-HQ]              Access switch
          Outside | DMZ | Inside      /         \
                  |      |       [PC-BR1]    [PC-BR2]
             [WEB-SRV]  10.0.2.1  VLAN 20    VLAN 20
              172.16.30.10   |
              (DMZ zone) 10.0.2.2
                        [MSW1-HQ]
                       Core L3 switch
                      /             \
                [AAA-SRV]         [PC-HQ1/HQ2]
                VLAN 10            VLAN 20
               10.10.10.10       10.10.20.10/11
```

**Site A — HQ** contains the primary infrastructure: edge router, ASA perimeter firewall, Layer 3 core switch, DMZ web server, centralized RADIUS server, and user workstations.

**Site B — Branch** is a lightweight remote site connected to HQ via a GRE tunnel through the simulated ISP.

---

## IP Addressing Scheme

### WAN Links

| Device | Interface | IP Address | Mask | Role |
|--------|-----------|------------|------|------|
| ISP-R | G0/0 | 203.0.113.2 | /30 | HQ WAN |
| ISP-R | G0/1 | 198.51.100.2 | /30 | Branch WAN |
| R1-HQ | G0/0 | 203.0.113.1 | /30 | to ISP-R |
| R1-HQ | G0/1 | 10.0.1.1 | /30 | to ASA outside |
| R2-BR | G0/0 | 198.51.100.1 | /30 | to ISP-R |

### HQ Internal

| Device | Interface | IP Address | Mask | Role |
|--------|-----------|------------|------|------|
| ASA-HQ | E0/0 / Vlan2 (outside) | 10.0.1.2 | /30 | from R1-HQ |
| ASA-HQ | E0/2 / Vlan1 (inside) | 10.0.2.1 | /30 | to MSW1-HQ |
| ASA-HQ | E0/1 / Vlan3 (dmz) | 172.16.30.1 | /24 | DMZ gateway |
| MSW1-HQ | G1/0/1 | 10.0.2.2 | /30 | uplink to ASA |
| MSW1-HQ | VLAN 10 SVI | 10.10.10.1 | /24 | MGMT gateway |
| MSW1-HQ | VLAN 20 SVI | 10.10.20.1 | /24 | USERS gateway |
| WEB-SRV | NIC | 172.16.30.10 | /24 | DMZ |
| AAA-SRV | NIC | 10.10.10.10 | /24 | VLAN 10 |
| PC-HQ1 | NIC | 10.10.20.10 | /24 | VLAN 20 |
| PC-HQ2 | NIC | 10.10.20.11 | /24 | VLAN 20 |

### Branch

| Device | Interface | IP Address | Mask | Role |
|--------|-----------|------------|------|------|
| R2-BR | G0/1.10 | 10.20.10.1 | /24 | Branch MGMT |
| R2-BR | G0/1.20 | 10.20.20.1 | /24 | Branch USERS |
| SW-BR | VLAN 10 SVI | 10.20.10.2 | /24 | Management IP |
| PC-BR1 | NIC | 10.20.20.10 | /24 | VLAN 20 |
| PC-BR2 | NIC | 10.20.20.11 | /24 | VLAN 20 |

### GRE Tunnel

| Device | Tunnel0 IP | Tunnel Source | Tunnel Destination |
|--------|-----------|--------------|-------------------|
| R1-HQ | 172.16.100.1/30 | G0/0 (203.0.113.1) | 198.51.100.1 |
| R2-BR | 172.16.100.2/30 | G0/0 (198.51.100.1) | 203.0.113.1 |

---

## VLAN Design

### HQ Site — MSW1-HQ

| VLAN ID | Name | Subnet | Gateway | Devices |
|---------|------|--------|---------|---------|
| 10 | MGMT | 10.10.10.0/24 | 10.10.10.1 | AAA-SRV |
| 20 | USERS | 10.10.20.0/24 | 10.10.20.1 | PC-HQ1, PC-HQ2 |
| 99 | NATIVE | — | — | Trunk native VLAN |

### Branch Site — R2-BR subinterfaces

| VLAN ID | Name | Subnet | Gateway | Devices |
|---------|------|--------|---------|---------|
| 10 | BR-MGMT | 10.20.10.0/24 | 10.20.10.1 | SW-BR management |
| 20 | BR-USERS | 10.20.20.0/24 | 10.20.20.1 | PC-BR1, PC-BR2 |
| 99 | NATIVE | — | — | Trunk native VLAN |

**Design note:** The branch uses router-on-a-stick (subinterfaces on R2-BR) rather than a Layer 3 switch. This is a realistic choice for a small remote site where a multilayer switch would be unnecessary overhead. The contrast between HQ (L3 switch SVIs) and Branch (router-on-a-stick) demonstrates knowing when to apply each approach.

---

## Security Features Implemented

### 1. Three-Zone ASA Firewall (DMZ Architecture)

The ASA 5505 divides the network into three security zones with explicit trust levels:

| Zone | Interface | Security Level | Purpose |
|------|-----------|---------------|---------|
| outside | Vlan2 / E0/0 | 0 (untrusted) | Internet / WAN |
| dmz | Vlan3 / E0/1 | 50 (semi-trusted) | Public-facing services |
| inside | Vlan1 / E0/2 | 100 (trusted) | Internal corporate network |

Traffic flows from higher to lower security levels by default. Traffic from lower to higher security levels requires explicit ACL permits. This means:
- Inside hosts can initiate connections to DMZ and outside
- DMZ hosts cannot initiate connections to inside (enforced by license restriction + policy)
- Outside hosts cannot reach inside without an explicit permit rule

### 2. DMZ Segmentation

WEB-SRV is isolated in the DMZ on a dedicated ASA interface (172.16.30.0/24). If WEB-SRV is compromised, the attacker is trapped in the DMZ — they cannot pivot to the inside corporate network without breaking through the ASA again at a higher security boundary.

### 3. VLAN Segmentation

Management infrastructure (RADIUS server) is on VLAN 10, isolated from user workstations on VLAN 20. A compromised user workstation cannot directly reach the AAA server at Layer 2. All inter-VLAN traffic is routed through MSW1-HQ's SVIs, giving a central point of control.

### 4. Centralized AAA / RADIUS Authentication

All network devices authenticate administrative logins against a central RADIUS server (AAA-SRV). Benefits:
- Single point of credential management — revoke one account and access is removed from all devices simultaneously
- Full accounting trail — every login attempt is logged centrally
- Local fallback ensures management access survives a RADIUS server failure

### 5. Perimeter ACL on R1-HQ WAN Interface

```
ip access-list extended WAN-IN
 permit gre any any
 permit icmp any any
 permit tcp any 10.10.0.0 0.0.255.255 established
 deny ip any any
```

Applied inbound on G0/0 (WAN interface). Only GRE tunnel traffic, ICMP, and established TCP sessions are permitted inbound. All other unsolicited inbound traffic is dropped.

### 6. Trunk Security

All trunk ports use VLAN 99 as the native VLAN — a VLAN that doesn't exist anywhere in the network. This prevents VLAN hopping attacks where an attacker on the default native VLAN (VLAN 1) sends double-tagged frames to jump into another VLAN.

### 7. Service Password Encryption

`service password-encryption` is configured on all IOS devices, ensuring line passwords in the running configuration are stored as encrypted strings rather than plaintext.

### 8. Enable Secret

All devices use `enable secret` (MD5 hashed) rather than `enable password` (reversible encryption). This ensures privileged EXEC passwords cannot be recovered from the configuration file.

### 9. Login Banners

All devices display a legal warning banner on login:
```
==========================================
  Project Sentinel --- [DEVICE]
  Authorized Access Only
==========================================
```

In real environments, login banners are a legal requirement. Without them, prosecuting unauthorized access becomes significantly harder because an attacker can argue they had no indication access was restricted.

---

## Configuration Highlights

### R1-HQ — Edge Router

```ios
! WAN interface
interface GigabitEthernet0/0
 description LINK-TO-ISP-R
 ip address 203.0.113.1 255.255.255.252
 ip access-group WAN-IN in
 no shutdown

! Inside interface toward ASA
interface GigabitEthernet0/1
 description LINK-TO-ASA-OUTSIDE
 ip address 10.0.1.1 255.255.255.252
 no shutdown

! GRE tunnel to branch
interface Tunnel0
 ip address 172.16.100.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 no shutdown

! Default and static routes
ip route 0.0.0.0 0.0.0.0 203.0.113.2
ip route 10.10.10.0 255.255.255.0 10.0.1.2
ip route 10.10.20.0 255.255.255.0 10.0.1.2
ip route 10.20.10.0 255.255.255.0 172.16.100.2
ip route 10.20.20.0 255.255.255.0 172.16.100.2
ip route 172.16.30.0 255.255.255.0 10.0.1.2

! Perimeter ACL
ip access-list extended WAN-IN
 permit gre any any
 permit icmp any any
 permit tcp any 10.10.0.0 0.0.255.255 established
 deny ip any any

! AAA
aaa new-model
radius-server host 10.10.10.10 auth-port 1645 key Sentinel@2025
aaa authentication login default group radius local
```

### MSW1-HQ — Core Layer 3 Switch

```ios
! Enable Layer 3 routing
ip routing

! VLANs
vlan 10
 name MGMT
vlan 20
 name USERS

! SVIs (default gateways for each VLAN)
interface Vlan10
 description MGMT-VLAN-GATEWAY
 ip address 10.10.10.1 255.255.255.0
 no shutdown

interface Vlan20
 description USERS-VLAN-GATEWAY
 ip address 10.10.20.1 255.255.255.0
 no shutdown

! Routed uplink to ASA
interface GigabitEthernet1/0/1
 description LINK-TO-ASA-INSIDE
 no switchport
 ip address 10.0.2.2 255.255.255.252
 no shutdown

! Access ports
interface GigabitEthernet1/0/2
 description LINK-TO-AAA-SRV
 switchport mode access
 switchport access vlan 10

interface GigabitEthernet1/0/3
 description LINK-TO-PC-HQ1
 switchport mode access
 switchport access vlan 20

interface GigabitEthernet1/0/4
 description LINK-TO-PC-HQ2
 switchport mode access
 switchport access vlan 20

! Default route toward ASA
ip route 0.0.0.0 0.0.0.0 10.0.2.1
```

### ASA-HQ — Perimeter Firewall

```asa
! Interface zones
interface Vlan2
 nameif outside
 security-level 0
 ip address 10.0.1.2 255.255.255.252

interface Vlan1
 nameif inside
 security-level 100
 ip address 10.0.2.1 255.255.255.252

interface Vlan3
 no forward interface Vlan1
 nameif dmz
 security-level 50
 ip address 172.16.30.1 255.255.255.0

! Physical port assignments
interface Ethernet0/0
 switchport access vlan 2
interface Ethernet0/2
 switchport access vlan 1
interface Ethernet0/1
 switchport access vlan 3

! Static routes
route outside 0.0.0.0 0.0.0.0 10.0.1.1 1
route inside 10.10.10.0 255.255.255.0 10.0.2.2 1
route inside 10.10.20.0 255.255.255.0 10.0.2.2 1
route outside 10.20.10.0 255.255.255.0 10.0.1.1 1
route outside 10.20.20.0 255.255.255.0 10.0.1.1 1

! Stateful inspection policy
class-map inspection_default
 match default-inspection-traffic
policy-map global_policy
 class inspection_default
  inspect icmp
  inspect http
  inspect ftp
  inspect dns
service-policy global_policy global

! Inbound ACL permitting branch traffic
access-list OUTSIDE-IN extended permit ip 10.20.0.0 255.255.0.0 10.10.0.0 255.255.0.0
access-list OUTSIDE-IN extended permit ip 10.20.0.0 255.255.0.0 172.16.30.0 255.255.255.0
access-group OUTSIDE-IN in interface outside

! Management access
telnet 10.10.20.0 255.255.255.0 inside
telnet 10.10.10.0 255.255.255.0 inside
aaa authentication telnet console LOCAL
username netadmin password Sentinel@2025
```

### R2-BR — Branch Edge Router

```ios
! WAN interface
interface GigabitEthernet0/0
 description LINK-TO-ISP-R
 ip address 198.51.100.1 255.255.255.252
 no shutdown

! Router-on-a-stick subinterfaces
interface GigabitEthernet0/1
 description LINK-TO-SW-BR
 no ip address
 no shutdown

interface GigabitEthernet0/1.10
 description BR-MGMT-VLAN10
 encapsulation dot1Q 10
 ip address 10.20.10.1 255.255.255.0

interface GigabitEthernet0/1.20
 description BR-USERS-VLAN20
 encapsulation dot1Q 20
 ip address 10.20.20.1 255.255.255.0

! GRE tunnel to HQ
interface Tunnel0
 ip address 172.16.100.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.1
 no shutdown

! Routes
ip route 0.0.0.0 0.0.0.0 198.51.100.2
ip route 10.10.10.0 255.255.255.0 172.16.100.1
ip route 10.10.20.0 255.255.255.0 172.16.100.1
ip route 172.16.30.0 255.255.255.0 172.16.100.1
ip route 10.0.1.0 255.255.255.252 172.16.100.1
```

---

## AAA / RADIUS Architecture

```
Network Device (R1-HQ, R2-BR, MSW1-HQ, SW-BR)
        |
        | RADIUS request (UDP port 1645)
        | Source: device management IP
        | Shared secret: Sentinel@2025
        |
        v
   AAA-SRV (10.10.10.10)
   Packet Tracer RADIUS server
   VLAN 10 — Management network
        |
        | Authentication response
        | (Accept / Reject)
        |
        v
Network Device grants or denies access
```

**Registered RADIUS clients:**

| Device | Source IP | Notes |
|--------|-----------|-------|
| R1-HQ | 10.0.1.1 | Closest interface to AAA-SRV |
| MSW1-HQ | 10.10.10.1 | VLAN 10 SVI |
| R2-BR | 172.16.100.2 | Tunnel0 interface |
| SW-BR | 10.20.10.2 | VLAN 10 management SVI |
| ASA-HQ | Local only | PT limitation — see production notes |

**User accounts:**

| Username | Access Level | Purpose |
|----------|-------------|---------|
| netadmin | Privilege 15 | Full administrative access |
| netview | Read-only | Monitoring and verification |

**Authentication flow:** Device → RADIUS (AAA-SRV) → if server unreachable → local account fallback. This ensures management access is never completely locked out even during a server failure.

---

## GRE Tunnel (Site-to-Site)

Generic Routing Encapsulation (GRE) creates a logical point-to-point tunnel between R1-HQ and R2-BR, allowing private subnet traffic from both sites to travel over the simulated public internet (ISP-R) as if they were directly connected.

**How it works:**

```
PC-BR1 sends packet to PC-HQ1 (10.10.20.10)
    |
R2-BR encapsulates packet in GRE header
    Outer IP: src=198.51.100.1, dst=203.0.113.1
    Inner IP: src=10.20.20.10, dst=10.10.20.10
    |
ISP-R routes outer packet normally (sees only 203.0.113.1)
    |
R1-HQ receives packet, strips GRE header
    Sees inner packet: dst=10.10.20.10
    Routes to ASA → MSW1 → PC-HQ1
    |
PC-HQ1 receives packet from real source IP 10.20.20.10
```

ISP-R only ever sees the outer GRE header — it has no visibility into the inner packet or the private addressing. This is the foundational concept behind all tunneling protocols including IPsec VPN.

**Routes installed for tunnel operation:**

| Router | Route | Via | Purpose |
|--------|-------|-----|---------|
| R1-HQ | 10.20.10.0/24 | 172.16.100.2 | Branch MGMT through tunnel |
| R1-HQ | 10.20.20.0/24 | 172.16.100.2 | Branch USERS through tunnel |
| R2-BR | 10.10.10.0/24 | 172.16.100.1 | HQ MGMT through tunnel |
| R2-BR | 10.10.20.0/24 | 172.16.100.1 | HQ USERS through tunnel |
| R2-BR | 172.16.30.0/24 | 172.16.100.1 | DMZ through tunnel |
| R2-BR | 10.0.1.0/30 | 172.16.100.1 | Return path for NAT'd traffic |

---

## Packet Tracer Limitations and Production Notes

Packet Tracer is a simulation environment with known limitations. The following features were either unavailable or partially supported and would be implemented differently on real hardware:

### 1. IKEv2 IPsec VPN

**PT limitation:** The Cisco 2901 in Packet Tracer requires the `securityk9` license package which could not be activated in simulation.

**Production implementation — R1-HQ:**
```ios
crypto ikev2 proposal SENTINEL-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy SENTINEL-POL
 proposal SENTINEL-PROP
!
crypto ikev2 keyring SENTINEL-KEYS
 peer R2-BR
  address 198.51.100.1
  pre-shared-key Sentinel@2025
!
crypto ikev2 profile SENTINEL-PROF
 match identity remote address 198.51.100.1
 authentication remote pre-share
 authentication local pre-share
 keyring local SENTINEL-KEYS
!
crypto ipsec transform-set SENTINEL-TS esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto map SENTINEL-MAP 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set SENTINEL-TS
 set ikev2-profile SENTINEL-PROF
 match address VPN-ACL
!
interface GigabitEthernet0/0
 crypto map SENTINEL-MAP
```

GRE was used in this lab to demonstrate the same tunneling concept — traffic encapsulation over a public network — while working within simulator constraints.

### 2. ASA RADIUS Authentication

**PT limitation:** The ASA 5505 in Packet Tracer does not support the `aaa-server` command group.

**Production implementation:**
```asa
aaa-server RADIUS-GROUP protocol radius
aaa-server RADIUS-GROUP (inside) host 10.10.10.10
 key Sentinel@2025
aaa authentication telnet console RADIUS-GROUP LOCAL
aaa authentication ssh console RADIUS-GROUP LOCAL
aaa authentication enable console RADIUS-GROUP LOCAL
```

### 3. SSH Instead of Telnet

**Current lab:** Telnet is used for management access because PT's SSH support has limitations with AAA integration.

**Production implementation:** All VTY lines should use SSH only:
```ios
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 transport input ssh
 login local
```

Telnet transmits credentials in plaintext and should never be used on production devices.

### 4. RADIUS Password Encryption

**Current lab:** RADIUS shared keys are stored as type 0 (plaintext) due to PT limitations.

**Production implementation:** Use type 6 encrypted keys:
```ios
key config-key password-encrypt [master-key]
password encryption aes
radius-server host 10.10.10.10 auth-port 1645 key 6 [encrypted-key]
```

### 5. ASA 5505 Base License — DMZ Limitation

The ASA 5505 Base License restricts the number of fully routable named interfaces to two. A third interface (DMZ) requires the `no forward interface vlan1` command, which prevents DMZ hosts from initiating connections directly to the inside interface. This is actually a reasonable security constraint for a DMZ — web servers in the DMZ should never initiate connections inward to the corporate network.

**Production recommendation:** Use an ASA 5506-X or higher which does not have this license restriction and supports full three-zone routing without the `no forward` workaround.

---

## Verification Evidence

The following connectivity tests were all verified successful:

### Site-to-Site (through GRE tunnel)

| Source | Destination | Result |
|--------|-------------|--------|
| PC-HQ1 (10.10.20.10) | PC-BR1 (10.20.20.10) | ✅ Success |
| PC-HQ1 (10.10.20.10) | PC-BR2 (10.20.20.11) | ✅ Success |
| PC-BR1 (10.20.20.10) | PC-HQ1 (10.10.20.10) | ✅ Success |
| PC-BR1 (10.20.20.10) | PC-HQ2 (10.10.20.11) | ✅ Success |
| PC-BR1 (10.20.20.10) | AAA-SRV (10.10.10.10) | ✅ Success |
| PC-BR1 (10.20.20.10) | WEB-SRV (172.16.30.10) | ✅ Success |

### HQ Internal Connectivity

| Source | Destination | Result |
|--------|-------------|--------|
| PC-HQ1 | ASA inside (10.0.2.1) | ✅ Success |
| PC-HQ1 | R1-HQ (10.0.1.1) | ✅ Success |
| PC-HQ1 | AAA-SRV (10.10.10.10) | ✅ Success |
| PC-HQ1 | WEB-SRV/DMZ (172.16.30.10) | ✅ Success |

### Management Access (RADIUS authenticated)

| Source | Device | Method | Result |
|--------|--------|--------|--------|
| PC-HQ1 | MSW1-HQ (10.10.20.1) | RADIUS | ✅ Success |
| PC-HQ1 | R1-HQ (10.0.1.1) | RADIUS | ✅ Success |
| PC-HQ1 | ASA-HQ (10.0.2.1) | Local | ✅ Success |
| PC-BR1 | R2-BR (10.20.20.1) | RADIUS | ✅ Success |
| PC-BR1 | SW-BR (10.20.10.2) | RADIUS | ✅ Success |

---

## Skills Demonstrated

| Skill | Evidence |
|-------|---------|
| Network topology design | Two-site enterprise network from scratch |
| Layer 3 switching | MSW1-HQ SVIs, `ip routing`, inter-VLAN routing |
| Router-on-a-stick | R2-BR subinterfaces for branch VLANs |
| VLAN segmentation | MGMT vs USERS vs Native VLAN design |
| Firewall configuration | ASA 5505 three-zone architecture |
| Stateful inspection | ASA inspection policy for ICMP, HTTP, DNS, FTP |
| GRE tunneling | Site-to-site tunnel with full routing |
| Static routing | Multi-hop routing across tunnel and ASA |
| Centralized AAA | RADIUS server with multiple device clients |
| Access control lists | Extended named ACL on WAN perimeter |
| Trunk security | Native VLAN hardening against VLAN hopping |
| Management plane security | Devices treated as attack surfaces |
| Troubleshooting methodology | Systematic isolation of 10+ configuration issues |
| Documentation | Production-grade hardening notes |

---

## How to Reproduce

### Requirements
- Cisco Packet Tracer 8.x (free with Cisco NetAcad account at netacad.com)
- Approximately 2-3 hours

### Steps

1. Clone this repository
2. Open `packet-tracer/topology.pkt` in Packet Tracer
3. All device configurations are pre-loaded
4. To verify: open any PC → Desktop → Command Prompt and run the ping tests listed in the Verification Evidence section
5. To test AAA: Telnet from any PC to a network device using `netadmin` / `Sentinel@2025`

### Device Credentials

| Username | Password | Access |
|----------|----------|--------|
| netadmin | Sentinel@2025 | Privilege 15 |
| netview | View@2025 | Read-only |
| Enable password | Sentinel@2025 | Privileged EXEC |

---

## Author

**Ryzen**
AAS Student — Cybersecurity and Network Administration
South Puget Sound Community College

Pursuing a career in Network Security Engineering and Cybersecurity Engineering.

- Built entirely with free and open-source tools
- No paid subscriptions required
- Designed to demonstrate real enterprise security architecture principles

---

*Project Sentinel is part of an ongoing series of network security labs designed to build hands-on proficiency across a range of networking and security technologies.*
