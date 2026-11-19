# Evidence 03 - Packet Path Analysis

> Sanitized command-output transcript. MAC addresses and unrelated fields are omitted.

## Capture 1 - OPNsense LAN Side

Command on Proxmox:

~~~bash
tcpdump -eni tap100i0 -c 6 'vlan 30 and icmp'
~~~

Representative packets:

~~~text
<TEST01_MAC> > <OPNSENSE_LAN_MAC>, ethertype 802.1Q,
vlan 30, ethertype IPv4,
10.10.30.151 > 1.1.1.1: ICMP echo request, seq 1

10.10.30.151 > 1.1.1.1: ICMP echo request, seq 2
10.10.30.151 > 1.1.1.1: ICMP echo request, seq 3
~~~

Result: three tagged echo requests reached the internal OPNsense interface. No echo replies were captured.

## Capture 2 - OPNsense WAN Side

Command:

~~~bash
tcpdump -eni tap100i1 -c 6 'icmp and host 1.1.1.1'
~~~

Representative packets:

~~~text
<OPNSENSE_WAN_MAC> > <UPSTREAM_GATEWAY_MAC>, ethertype IPv4,
10.0.0.68 > 1.1.1.1: ICMP echo request, seq 1

10.0.0.68 > 1.1.1.1: ICMP echo request, seq 2
10.0.0.68 > 1.1.1.1: ICMP echo request, seq 3
~~~

Result: OPNsense routed the flow and translated the source from `10.10.30.151` to its WAN address, `10.0.0.68`. No echo replies were captured.

## Capture 3 - Physical Proxmox NIC

Command:

~~~bash
tcpdump -eni nic0 -c 6 'icmp and host 1.1.1.1'
~~~

Representative packets:

~~~text
<OPNSENSE_WAN_MAC> > <UPSTREAM_GATEWAY_MAC>, ethertype IPv4,
10.0.0.68 > 1.1.1.1: ICMP echo request, seq 1

10.0.0.68 > 1.1.1.1: ICMP echo request, seq 2
10.0.0.68 > 1.1.1.1: ICMP echo request, seq 3
~~~

Result: the NAT-translated requests left the physical host. No echo replies returned through this capture point.

## Control Tests

### Local WAN gateway

~~~text
OPNsense -> 10.0.0.1
3 ICMP replies received
~~~

### Public ICMP

~~~text
OPNsense -> 1.1.1.1: 100% packet loss
Proxmox -> 1.1.1.1: 100% packet loss
Proxmox -> 8.8.8.8: 100% packet loss
test01 -> 1.1.1.1: 100% packet loss
~~~

### Orange-router firewall A/B test

A controlled test changed only the upstream Orange-router firewall state.

~~~text
Firewall enabled:
Proxmox -> 1.1.1.1: 100% packet loss

Firewall disabled:
Proxmox -> 1.1.1.1: 3 packets transmitted, 3 received
~~~

The restored replies after disabling the firewall confirm that the upstream firewall filtered public ICMP.

### TCP from OPNsense

~~~bash
nc -vz -w 5 1.1.1.1 443
~~~

~~~text
Connection to 1.1.1.1 443 port [tcp/https] succeeded!
~~~

### DNS and TCP from test01

~~~bash
timeout 5 bash -c 'echo > /dev/tcp/example.com/443'   && echo "TCP 443 OK" || echo "TCP 443 FAIL"
~~~

~~~text
TCP 443 OK
~~~

## Diagnostic Conclusion

The observations support the following sequence:

1. test01 generated a VLAN 30 ICMP request.
2. `vmbr1` delivered the tagged frame to OPNsense.
3. OPNsense routed the packet and applied source NAT.
4. The translated packet left Proxmox through `nic0`.
5. No public ICMP reply returned.
6. TCP 443 succeeded from OPNsense and test01.

Routing, NAT, DNS, and permitted TCP connectivity were therefore operational. The controlled firewall on/off comparison confirmed that the enabled Orange-router firewall filtered the public ICMP replies.
