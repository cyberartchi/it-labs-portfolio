# Evidence 02 - Bridge FDB and Neighbor Cache

> Sanitized command-output transcript. Role-based placeholders replace full hardware addresses.

## Dynamic FDB Entries

Command:

~~~bash
bridge fdb show br vmbr1 | grep -v permanent
~~~

Relevant result:

~~~text
<DC01_MAC>          dev fwpr101p0 vlan 1  master vmbr1
<PC01_MAC>          dev fwpr102p0 vlan 1  master vmbr1
<TEST01_MAC>        dev veth103i0 vlan 30 master vmbr1
<OPNSENSE_LAN_MAC>  dev tap100i0  vlan 1  master vmbr1
~~~

Interpretation:

| Learned address | Bridge port | VLAN | Role |
|---|---|---:|---|
| `<DC01_MAC>` | `fwpr101p0` | 1 | Windows Server VM |
| `<PC01_MAC>` | `fwpr102p0` | 1 | Windows client VM |
| `<TEST01_MAC>` | `veth103i0` | 30 | Isolated test container |
| `<OPNSENSE_LAN_MAC>` | `tap100i0` | 1 | OPNsense internal interface |

The test01 entry proves that the bridge received a frame from the container and learned its source MAC on VLAN 30.

## Neighbor Cache

Command on test01:

~~~bash
ip neigh show 10.10.30.1
~~~

Result:

~~~text
10.10.30.1 dev eth0 lladdr <OPNSENSE_LAN_MAC> STALE
~~~

`STALE` means the mapping exists but has not been confirmed recently. Linux may use it and perform reachability confirmation when new traffic is sent.

## Layer Comparison

| Data structure | Layer | Mapping | Diagnostic value |
|---|---:|---|---|
| Bridge FDB | 2 | MAC address to bridge port and VLAN | Shows where the virtual switch learned an Ethernet endpoint |
| Neighbor cache | 3/2 boundary | IPv4 address to next-hop MAC address | Shows how the host addresses the next hop on the local link |

FDB learning and ARP/neighbor resolution can succeed even when a routed ICMP packet is later blocked by firewall policy.
