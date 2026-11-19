# Network Device, Bridge, and Packet Path Analysis

This module documents hands-on inspection of physical and virtual network interfaces, Linux bridge forwarding, VLAN-aware switching, ARP/neighbor resolution, NAT, and multi-point packet capture in a Proxmox VE and OPNsense laboratory.

## Objective

The exercises were designed to:

- inventory the interfaces that build the lab network;
- map virtual machines and containers to Proxmox bridge ports;
- distinguish physical link speed from virtual adapter speed;
- read the Linux bridge forwarding database (FDB);
- compare Layer 2 forwarding information with the Layer 3 neighbor cache;
- follow an ICMP packet through VLAN 30, OPNsense, NAT, and the physical uplink;
- diagnose why ICMP failed while TCP-based Internet access remained operational.

## Environment

| Component | Role | Relevant network information |
|---|---|---|
| Proxmox VE | Hypervisor and virtual switching | `nic0`, `vmbr0`, `vmbr1` |
| OPNsense VM100 | Firewall, router, and NAT gateway | LAN `10.10.10.1/24`, WAN `10.0.0.68/24` |
| Orange router | Upstream gateway | `10.0.0.1` |
| DC01 VM101 | AD DS, DNS, and DHCP server | `10.10.10.10/24` |
| PC01 VM102 | Windows domain client | `10.10.10.100/24` |
| test01 CT103 | Isolated VLAN test host | VLAN 30, `10.10.30.151` |

## Current Logical Topology

~~~mermaid
flowchart TD
    Internet["Internet"] --> Orange["Orange router - 10.0.0.1"]
    Orange --> NIC["Proxmox nic0 - 100 Mb/s full duplex"]
    NIC --> EXT["vmbr0 - external bridge"]
    EXT --> FW["OPNsense VM100 - WAN 10.0.0.68"]
    FW --> INT["vmbr1 - VLAN-aware internal bridge"]
    INT --> LAN["DC01 and PC01 - 10.10.10.0/24"]
    INT --> GUEST["test01 - VLAN 30 - 10.10.30.151"]
~~~

`vmbr0` carries the WAN-side connection between the physical NIC and OPNsense. `vmbr1` is the internal VLAN-aware bridge used by the protected lab networks.

## Lab 1 - Interface Inventory and Link Analysis

The Proxmox interface inventory was collected with:

~~~bash
ip -br link
bridge link
lspci -nnk | grep -A3 -i ethernet
ethtool nic0 | grep -E 'Speed|Duplex|Auto-negotiation|Link detected'
~~~

### Interface Mapping

| Interface or path | Function |
|---|---|
| `nic0` | Physical Realtek Ethernet interface connected to the upstream switch |
| `vmbr0` | External bridge carrying the OPNsense WAN connection |
| `tap100i1` | OPNsense WAN-side virtual interface |
| `tap100i0` | OPNsense LAN-side virtual interface attached to `vmbr1` |
| `vmbr1` | Internal VLAN-aware bridge |
| `fwbr101i0 / fwpr101p0 / fwln101i0` | Proxmox firewall bridge path for DC01 |
| `fwbr102i0 / fwpr102p0 / fwln102i0` | Proxmox firewall bridge path for PC01 |
| `veth103i0` | test01 container interface, assigned to VLAN 30 |

The physical controller was identified as a Realtek RTL8111/8168/8211/8411 PCIe Gigabit Ethernet device using the `r8169` kernel driver.

`ethtool` reported:

- speed: 100 Mb/s;
- duplex: full;
- auto-negotiation: enabled;
- link detected: yes.

The host NIC is Gigabit-capable, but the current physical link negotiated at only 100 Mb/s. The old 100 Mb/s switch is therefore the present wired bottleneck. The 10 Gb/s value shown by the Windows VirtIO adapters describes the virtual link presented to the guests; it does not override the physical 100 Mb/s uplink.

Windows validation confirmed:

| Host | Adapter | IPv4 | Gateway | DNS |
|---|---|---|---|---|
| DC01 | Red Hat VirtIO Ethernet Adapter, 10 Gb/s | `10.10.10.10` | `10.10.10.1` | `::1` and `10.10.10.10` |
| PC01 | Red Hat VirtIO Ethernet Adapter, 10 Gb/s | `10.10.10.100` | `10.10.10.1` | `10.10.10.10` |

PC01 also reported DHCP enabled and an IPv4 MTU of 1500.

## Lab 2 - Bridge FDB, Neighbor Cache, and Packet Path

### Layer 2: Bridge Forwarding Database

The dynamic forwarding entries were inspected with:

~~~bash
bridge fdb show br vmbr1 | grep -v permanent
~~~

The learned entries mapped the lab MAC addresses to the expected bridge ports:

- DC01 to `fwpr101p0` on VLAN 1;
- PC01 to `fwpr102p0` on VLAN 1;
- OPNsense LAN to `tap100i0` on VLAN 1;
- test01 to `veth103i0` on VLAN 30.

This proves that Ethernet frames from test01 reached `vmbr1` and that the bridge learned the source MAC on the correct VLAN-aware port.

### Layer 3: Neighbor Cache

On test01, the command below showed the gateway IP associated with the OPNsense LAN MAC:

~~~bash
ip neigh show 10.10.30.1
~~~

The state was `STALE`. This is normal: the mapping is valid but has not been confirmed recently. It does not mean the gateway is unreachable. The neighbor cache connects an IP address to a next-hop MAC, while the FDB tells the bridge through which port that MAC can be reached.

### Multi-Point Packet Capture

ICMP was captured at three points while test01 pinged `1.1.1.1`:

| Capture point | Observation | Meaning |
|---|---|---|
| `tap100i0` | `10.10.30.151 > 1.1.1.1`, 802.1Q VLAN 30 | The tagged frame reached the OPNsense LAN interface |
| `tap100i1` | `10.0.0.68 > 1.1.1.1` | OPNsense routed the packet and applied source NAT |
| `nic0` | `10.0.0.68 > 1.1.1.1` | The translated request left the physical Proxmox interface |

No ICMP echo replies were observed at any of these points.

## Troubleshooting Result

The following tests narrowed down the failure:

| Test | Result | Interpretation |
|---|---|---|
| OPNsense to Orange gateway `10.0.0.1` | ICMP reply received | Local WAN connectivity works |
| Proxmox to `1.1.1.1` and `8.8.8.8` | No ICMP reply | The behavior is not limited to VLAN 30 |
| OPNsense to `1.1.1.1:443` with `nc` | TCP connection succeeded | OPNsense has working upstream routing |
| test01 to `example.com:443` using Bash `/dev/tcp` | `TCP 443 OK` | VLAN 30 has DNS resolution, routing, firewall permission, NAT, and TCP Internet access |

The evidence shows that OPNsense forwarded and translated the traffic correctly, and that public TCP connectivity worked from both the firewall and VLAN 30. Therefore, failed public pings did not mean that Internet access was down.

A controlled A/B test was later completed on the Orange router. With its firewall enabled, public ICMP failed from Proxmox; after the firewall was disabled, three consecutive replies were received from `1.1.1.1`. This confirms that the upstream Orange-router firewall, rather than OPNsense routing or VLAN policy, filtered the public ICMP replies.

## Validation Summary

| Requirement | Status |
|---|---|
| Physical and virtual interfaces inventoried | PASS |
| Proxmox bridge membership mapped | PASS |
| Physical link parameters verified | PASS |
| Dynamic FDB entries identified | PASS |
| VLAN 30 MAC learned on the correct port | PASS |
| Neighbor entry for the VLAN gateway interpreted | PASS |
| Packet traced across LAN, WAN, NAT, and physical NIC | PASS |
| TCP Internet access from VLAN 30 verified | PASS |
| Cause of missing public ICMP confirmed through an upstream firewall A/B test | PASS |

## Evidence

All published transcripts are sanitized. Hardware addresses are replaced with role-based placeholders.

| Evidence | Description |
|---|---|
| [Interface inventory](evidence/01-interface-inventory.md) | Proxmox, physical NIC, bridge paths, and Windows adapter results |
| [Bridge FDB and neighbor cache](evidence/02-bridge-fdb-and-neighbor.md) | Layer 2 and Layer 3 mappings |
| [Packet path analysis](evidence/03-packet-path-analysis.md) | VLAN tag, NAT boundary, physical egress, and protocol comparison |

## What I Learned

- A Linux bridge learns source MAC addresses and stores MAC-to-port mappings in its FDB.
- The FDB is a Layer 2 data structure; the neighbor cache maps Layer 3 IP addresses to Layer 2 addresses.
- A `STALE` neighbor entry is normal and can still be used.
- VLAN isolation rules can block routed traffic without preventing local Ethernet learning or ARP resolution.
- Capturing the same flow on both sides of a firewall reveals where VLAN tags are removed and where NAT changes the source address.
- A 10 Gb/s virtual adapter does not guarantee 10 Gb/s end-to-end throughput when the physical uplink is negotiated at 100 Mb/s.
- Ping tests only ICMP. A failed ping must not be treated as proof that DNS, routing, NAT, or TCP Internet access is broken.
- A controlled A/B test can convert a well-supported hypothesis into a confirmed root cause; disabling the Orange-router firewall restored public ICMP replies.

## Security and Privacy

The original screenshots were reviewed before publication. Full MAC addresses and unnecessary hardware identifiers were replaced with descriptive placeholders. Credentials, public subscriber addresses, and personal information are not included. Private RFC 1918 lab addresses and purpose-built host names remain because they are required to explain the topology.

## Conclusion

The two exercises connected interface inventory, virtual switching, VLAN forwarding, neighbor discovery, routing, NAT, and packet capture into one diagnostic workflow. The lab infrastructure forwarded TCP traffic correctly, and the follow-up A/B test confirmed that the Orange-router firewall filtered the missing public ICMP replies.

[Back to the main portfolio](../../)
