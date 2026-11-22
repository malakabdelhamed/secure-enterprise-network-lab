# Secure Enterprise Network Lab (Cisco Packet Tracer)

A small enterprise network built in Cisco Packet Tracer 9.0.1 and hardened with Layer 2 and Layer 3 security controls. I built it to apply what I learned in the **CCNA for CyberSec** course and to practice both configuring a network and testing it against a simulated attack.

**Author:** [Your Name]

## What this project demonstrates

- Network segmentation with VLANs and 802.1Q trunking
- Inter-VLAN routing with router-on-a-stick
- Access control with an extended ACL that isolates the Guest network
- Port Security with sticky MAC addresses, tested against a rogue device
- Shutdown of unused switch ports
- Secure remote management (SSH v2) on a dedicated Management VLAN

## Topology

> Add `screenshots/topology.png` here.

| Device | Model | Role |
|---|---|---|
| R1 | Cisco 2911 | Router-on-a-stick, ACL |
| SW1 | Cisco 2960-24TT | Access switch, Port Security |
| PC-IT, PC-HR, PC-Guest | PC | One host per department |

## Addressing plan

| VLAN | Name | Network | Gateway (R1) | Host |
|---|---|---|---|---|
| 10 | IT | 192.168.10.0/24 | 192.168.10.1 | PC-IT 192.168.10.10 |
| 20 | HR | 192.168.20.0/24 | 192.168.20.1 | PC-HR 192.168.20.10 |
| 30 | Guests | 192.168.30.0/24 | 192.168.30.1 | PC-Guest 192.168.30.10 |
| 99 | Management | 192.168.99.0/24 | 192.168.99.1 | SW1 192.168.99.2 |

| Link | Ports |
|---|---|
| R1 ↔ SW1 (trunk, native VLAN 99) | R1 Gig0/0 ↔ SW1 Fa0/1 |
| PC-IT | SW1 Fa0/10 (VLAN 10) |
| PC-HR | SW1 Fa0/20 (VLAN 20) |
| PC-Guest | SW1 Fa0/24 (VLAN 30) |

## Security controls

### 1. Segmentation and routing
Each department sits in its own VLAN. R1 routes between them using dot1Q subinterfaces (`g0/0.10`, `.20`, `.30`, `.99`).

### 2. Guest isolation (extended ACL)
The ACL `GUEST-IN` is applied inbound on `g0/0.30`. It denies the Guest network access to IT, HR and Management, and permits everything else.

```
ip access-list extended GUEST-IN
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip any any
```

### 3. Port Security
Access ports allow one MAC address (sticky), and shut down on violation.

```
interface range fa0/10, fa0/20, fa0/24
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

Unused ports are administratively shut down.

### 4. Secure management
SSH v2 only on the VTY lines, with local authentication, and an access class limiting which network can manage the switch. Passwords are stored encrypted.

## Verification and test results

| # | Test | Expected | Result |
|---|---|---|---|
| 1 | PC-IT pings PC-HR | Success | Pass |
| 2 | PC-Guest pings its gateway 192.168.30.1 | Success | Pass |
| 3 | PC-Guest pings PC-IT and PC-HR | Blocked by ACL (`Destination host unreachable`) | Pass |
| 4 | `show access-lists` | Match counters increase on deny lines | Pass |
| 5 | Rogue PC plugged into Fa0/10 | Port goes `Secure-shutdown`, violation count increases | Pass |
| 6 | SSH from PC-IT to 192.168.99.2 | Login prompt, session opens | Pass |
| 7 | SSH from PC-Guest to 192.168.99.2 | Connection timed out | Pass |
| 8 | SSH from PC-HR to 192.168.99.2 | Rejected | Pass |

Screenshots are in the [`screenshots/`](screenshots/) folder.

## Lessons learned

- Many `Invalid input` errors came from typing configuration commands while still in user mode. Checking the prompt (`>` vs `#` vs `(config)#`) first saves a lot of time.
- `show vlan brief` lists a port under a VLAN even if nothing is connected to it. `show mac address-table` showed that PC-Guest was on the wrong port, which explained the failed pings.
- Sticky MAC learns the first device that sends traffic. My first attack test failed because the rogue PC was the first to talk on the port, so it became the trusted device. The port has to learn the legitimate host first.

## Files

```
.
├── README.md
├── Secure-Enterprise-Network.pkt   # Packet Tracer file
├── configs/
│   ├── R1.txt
│   └── SW1.txt
└── screenshots/
```

## How to run

1. Install Cisco Packet Tracer (free from Cisco Networking Academy / Skills for All).
2. Open `Secure-Enterprise-Network.pkt`.
3. Use the tests in the table above to verify behaviour.

## Possible next steps

- DHCP server with DHCP Snooping
- Dynamic ARP Inspection
- OSPF and NAT toward a simulated ISP
- Syslog and NTP for monitoring
