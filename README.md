# Securing the Local Area Network: A Study Walkthrough

> Personal study notes on Layer 2 and LAN security, written in my own words while studying Network Security at UniKL MIIT. Cisco IOS examples are illustrative; adapt them to your device and test in a lab first.

## 1. What needs protecting

Securing the edge is not enough. The internal LAN matters just as much, and most Layer 2 attacks come from users who already have internal access. A LAN includes:

- **Endpoints:** laptops, desktops, IP phones, servers, printers.
- **Non-endpoint devices:** switches, wireless devices, IP telephony and SAN devices.
- **The LAN infrastructure itself.**

Most router security techniques also apply to switches, but switches have their own attacks.

## 2. Layer 2 attacks at a glance

| Attack | What happens | Main defence |
|---|---|---|
| MAC address spoofing | Attacker pretends to be another device's MAC | Port security |
| MAC table overflow | Spoofed MACs fill the table, switch floods like a hub, attacker sniffs | Port security |
| STP manipulation | Attacker becomes root bridge with a lower priority BPDU | PortFast, BPDU guard, root guard |
| LAN storm | Excess broadcast, multicast or unicast traffic degrades the network | Storm control |
| VLAN hopping | Attacker reaches other VLANs via DTP or double tagging | Disable DTP, unused native VLAN |

## 3. Port security (MAC spoofing and table overflow)

Port security limits which MAC addresses, or how many, a port will accept. Secure addresses can be configured manually or learned dynamically. Learning a fixed number dynamically scales far better than typing static MACs.

### Configuration

```
interface <INTERFACE>
 switchport mode access
 switchport port-security
 switchport port-security maximum <N>
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 switchport port-security aging time <MINUTES>
```

- Default maximum is 1 (range 1 to 132).
- **Sticky** learning adds dynamically learned MACs to the running configuration.
- **Aging** can be *absolute* (addresses expire after the set time) or *inactivity* (expire only when idle).

### Violation modes

| Mode | Drops frames | Log, SNMP trap, counter | Port err-disabled |
|---|---|---|---|
| protect | Yes | No (silent) | No |
| restrict | Yes | Yes | No |
| shutdown (default, recommended) | Yes | Yes | Yes |

Restrict can struggle under attack load, so shutdown is recommended. To recover an err-disabled port, use `shutdown` then `no shutdown`, or `errdisable recovery cause psecure-violation`.

Useful checks: `show port-security`, `show port-security interface <INTERFACE>`, `show port-security address`. MAC Address Notification (`mac address-table notification`) sends SNMP traps when addresses are added or removed.

## 4. Mitigating STP manipulation

An attacker sends BPDUs with a lower bridge priority to be elected root, which lets them see frames they should never receive.

| Feature | Where | What it does |
|---|---|---|
| PortFast | Access ports only | Goes straight to forwarding. On switch-to-switch links it can cause a loop. |
| BPDU guard | PortFast / user-facing ports | If a BPDU arrives, the port is err-disabled. Recovery is manual or by timeout. |
| BPDU filter | PortFast ports | Stops BPDUs being sent or received. Enabling it per interface is like turning STP off there, so it can cause loops. |
| Root guard | Ports toward switches that must never be root | Superior BPDUs put the port in root-inconsistent state. No data forwarded. Recovers automatically when they stop. |

```
spanning-tree portfast
spanning-tree portfast default
spanning-tree portfast bpduguard default
spanning-tree guard root
show spanning-tree summary
show spanning-tree inconsistentports
```

**BPDU guard vs root guard:** BPDU guard disables the port on any BPDU (PortFast ports). Root guard lets the device take part in STP as long as it doesn't try to become root, and recovers by itself.

## 5. Storm control

Switches always forward broadcasts, and protocols like ARP and DHCP need them, so storm control watches traffic against thresholds instead of blocking broadcasts outright.

- Measured as a percentage of bandwidth, bits per second or packets per second.
- A rising threshold blocks the traffic. If a falling threshold is set, normal forwarding resumes below it.
- 100% means no limit, 0% blocks all traffic of that type.
- Action: `trap` (log) or `shutdown` (err-disable the port).

```
storm-control broadcast level 75.5
storm-control multicast level pps 2k 1k
storm-control action shutdown
show storm-control
```

## 6. Mitigating VLAN attacks

VLAN hopping happens two ways:

1. **Rogue switch with DTP:** the attacker's device negotiates a trunk and joins all VLANs.
2. **Double tagging:** the frame carries two 802.1Q tags. The first switch strips the outer tag, and the second switch delivers the frame to the inner VLAN.

Defences:

```
interface <TRUNK_INTERFACE>
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan <UNUSED_VLAN>
```

- Enable trunking only where needed and turn DTP off elsewhere.
- Use a dedicated, unused native VLAN (the default is VLAN 1).
- Shut down unused ports and place them in an unused VLAN.

## 7. Monitoring and isolation

- **SPAN** copies traffic from a port or VLAN to another port on the same switch (RSPAN to a port on another switch), typically for an IDS or analyzer. It enables monitoring but does not mitigate attacks.

```
monitor session 1 source interface <SRC>
monitor session 1 destination interface <DST> encapsulation replicate
show monitor session 1
```

- **PVLAN Edge (protected ports):** `switchport protected` blocks Layer 2 traffic between protected ports. Traffic between them must go through a Layer 3 device.

## 8. Layer 2 best practices

- Manage switches securely: SSH, out-of-band management and ACLs.
- Make user ports non-trunking, use port security on access ports, and keep CDP only where necessary.
- PortFast and BPDU guard on non-trunking ports, root guard on root ports.
- Disable auto-trunking on user ports, configure trunks explicitly, disable unused ports.
- Use separate VLANs for management, native, user data, voice, black hole and private. Avoid VLAN 1 except for Layer 2 control traffic.

## 9. Wireless

- **Autonomous** access points are configured one by one. **Lightweight** APs are managed by a wireless LAN controller, which handles security policy, intrusion prevention, RF management, QoS and mobility.
- Attack categories: reconnaissance (wardriving, tools such as NetStumbler and Kismet), access attacks and DoS.
- Security ladder, weakest to strongest: defaults, unique SSID with broadcast off, WEP, WEP with TKIP, WPA with TKIP, **WPA2 with AES**. WEP and WPA/TKIP are vulnerable.
- Use a long passphrase (21+ characters), a VPN on public wireless, disable unused radios, and use a central authentication server.

## 10. VoIP

VoIP rides on the data network, so data attacks apply too, plus voice-specific ones:

- **Unauthorised access** to voice resources, eavesdropping, and DoS.
- **SPIT:** bulk unwanted VoIP calls. Authenticated TLS stops most.
- **Vishing:** voice phishing to collect account details.
- **Toll fraud:** theft of long-distance service. Dial plan filters and authorisation codes help.
- **SIP weaknesses:** registration hijacking, message tampering and session tear-down.

Defences: a voice VLAN, firewalls that inspect voice protocols (SIP, SCCP, H.323, MGCP), IPsec VPNs, and hardened phones (no default accounts, signed images only).

## 11. Storage area networks (SANs)

A SAN is a specialised network giving servers fast, reliable access to shared storage.

| Technology | Role |
|---|---|
| Fibre Channel | Primary host-to-SAN transport, identifies devices by WWN |
| iSCSI | SCSI over TCP/IP, typically on the LAN, uses LUNs |
| FCIP | SAN-to-SAN connection across a WAN or MAN |

- **Zoning** partitions the Fibre Channel fabric so zone members see only each other.
- **VSANs** form virtual fabrics (like VLANs for SANs) with hardware-based isolation.
- Six areas to secure: management, fabric access, target access, SAN protocols, IP storage access, and data integrity and secrecy (encrypt in transit and at rest).

## 12. Key takeaways

- Protect the inside of the network, not just the edge.
- Most Layer 2 defences are configuration: port security, PortFast with BPDU guard, root guard, storm control, correct trunk settings.
- Know the recovery behaviour of each feature, since the differences are easy to mix up.
- Wireless, VoIP and SAN each add their own risks and need matching controls.

---

*Sources: Cisco CCNA Security course material (Chapter 6), summarised and rewritten in my own words.*
