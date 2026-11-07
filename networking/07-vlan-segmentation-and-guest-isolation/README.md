# VLAN Segmentation and Guest Network Isolation

## Objective

The objective of this lab was to create an isolated guest network using IEEE 802.1Q VLAN tagging, provide DHCP, DNS, and Internet access through OPNsense, and prevent the guest client from reaching the internal lab network.

The exercise also documents a Layer 2 troubleshooting case in which DHCP failed because the OPNsense virtual switch port had not inherited the updated VLAN-aware bridge configuration.

## Environment

| Component | Role | Configuration |
|---|---|---|
| Proxmox VE | Hypervisor and virtual switching | `vmbr0` for management/WAN and VLAN-aware `vmbr1` for the isolated lab trunk |
| OPNsense | Router, firewall, DHCP, and DNS gateway | LAN `10.10.10.1/24`; VLAN 30 interface `10.10.30.1/24` |
| Debian 13 LXC (`test01`) | Guest-network validation client | Unprivileged container, tagged VLAN 30, IPv4 by DHCP |
| Dnsmasq | DHCP and DNS service | Pool `10.10.30.100–10.10.30.200`; one-day lease |
| Proxmox console and OPNsense Live View | Validation and troubleshooting | Bridge membership, addressing, routing, name resolution, and firewall logs |

## Network Design

| Item | Value |
|---|---|
| VLAN ID | 30 |
| Interface name | `GUEST_V30` |
| Subnet | `10.10.30.0/24` |
| Gateway and DNS server | `10.10.30.1` |
| DHCP pool | `10.10.30.100–10.10.30.200` |
| Client address observed | `10.10.30.151/24` |
| Intended policy | Permit DNS and Internet access; block access to the internal `10.0.0.0/8` lab address space |

## Implementation

### 1. VLAN-aware Proxmox bridge

The isolated internal bridge was configured without a physical uplink and enabled for VLAN filtering:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

OPNsense used `vmbr1` as the internal trunk. The Debian container was connected to the same bridge with Proxmox VLAN tag 30, which presents untagged Ethernet to the container while preserving VLAN 30 on the virtual bridge.

### 2. OPNsense VLAN and services

VLAN 30 was created on the OPNsense LAN parent interface, assigned as `GUEST_V30`, and configured with the static address `10.10.30.1/24`.

Dnsmasq supplied:

- IPv4 addresses from `10.10.30.100` through `10.10.30.200`;
- default gateway `10.10.30.1`;
- DNS service through `10.10.30.1`;
- a one-day DHCP lease.

### 3. Firewall policy

Rules on `GUEST_V30` were evaluated in this order:

1. Permit TCP/UDP DNS from `GUEST_V30 net` to the `GUEST_V30` interface address on port 53.
2. Block and log IPv4 traffic from `GUEST_V30 net` to `10.0.0.0/8`.
3. Permit remaining IPv4 traffic to the Internet.

Rule order is essential: the DNS exception must appear above the private-network block because the DNS gateway itself is inside `10.0.0.0/8`.

The current block covers the internal networks used in this lab. A production-style policy should instead use an RFC1918 alias containing `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`.

## Troubleshooting

### Symptom

The Debian interface was up and had an IPv6 link-local address, but it did not receive IPv4 configuration. A verbose interface restart ended with:

```text
eth0: dhclient: timeout failed to detect new ip addresses
```

### Evidence and reasoning

- The Dnsmasq scope was active, but its logs contained no DHCP discovery from the container.
- `bridge vlan show` confirmed that the container-side `veth103i0` port used VLAN 30 as its PVID.
- The OPNsense-side `tap100i0` port still showed only VLAN 1.
- Because DHCP discovery is a Layer 2 broadcast, its absence from Dnsmasq logs indicated that the request never reached OPNsense.

### Root cause and remediation

The OPNsense tap interface had been created before `vmbr1` was changed to a VLAN-aware bridge. Its switch-port membership was therefore not rebuilt with the trunk VLAN range.

A complete OPNsense VM shutdown followed by a start recreated the tap interface and applied the updated bridge configuration. A guest operating-system reboot alone would not recreate the Proxmox tap port. After the cold restart, DHCP succeeded immediately.

## Validation

The following commands were run inside `test01`:

```bash
ip -br addr
ip route
ping -c 3 1.1.1.1
getent ahostsv4 example.com
ping -c 3 10.10.10.10
```

| Test | Expected result | Observed result |
|---|---|---|
| DHCP addressing | Address from the VLAN 30 pool | `10.10.30.151/24` assigned |
| Default route | Gateway through OPNsense | `default via 10.10.30.1 dev eth0` |
| Internet reachability | External IPv4 destination reachable | `1.1.1.1`: 3/3 replies, 0% packet loss |
| DNS resolution | Public hostname resolves | `example.com` returned IPv4 records |
| Internal isolation | Guest cannot reach LAN host | `10.10.10.10`: 100% packet loss |
| Firewall verification | Denied traffic appears in logs | ICMP from `10.10.30.151` to `10.10.10.10` logged as blocked |

## Evidence

| Evidence | Description |
|---|---|
| [VLAN-aware bridge configuration](screenshots/01-proxmox-vlan-aware-bridge.png) | `vmbr1` configured for VLAN IDs 2–4094 |
| [DHCP address on the guest client](screenshots/02-client-dhcp-address.png) | Debian received `10.10.30.151/24` on `eth0` |
| [Route and Internet test](screenshots/03-route-and-internet-test.png) | Default route through OPNsense and successful external ping |
| [DNS and isolation test](screenshots/04-dns-and-isolation-test.png) | DNS resolution succeeded while the LAN host remained unreachable |
| [OPNsense firewall log](screenshots/05-firewall-block-log.png) | Logged block from VLAN 30 to the internal LAN address |

## Findings

- VLAN 30 tagging was preserved between the Proxmox container port and the OPNsense trunk.
- DHCP, DNS, routing, and outbound Internet access worked after the trunk port was recreated.
- The guest client could not reach the internal LAN host, and the denial was visible in OPNsense Live View.
- An active DHCP service does not prove that broadcasts reach it; the switch-port VLAN state must also be verified.
- Comparing both ends of the virtual bridge narrowed the failure to Layer 2 before changing the DHCP or firewall configuration.

## Security and Privacy

The screenshots were reviewed before publication. They contain only purpose-built lab names, private RFC1918 addresses, and public test destinations. Credentials, public source addresses, personal identifiers, and unnecessary hardware identifiers were excluded.

## Conclusion

The lab successfully implemented a routed guest VLAN with automatic addressing, DNS, and Internet access while enforcing and logging isolation from the internal lab network. It also demonstrated a repeatable troubleshooting method: verify client state, service logs, VLAN membership on both switch ports, and only then modify higher-layer services.

[Back to the portfolio](../../README.md)
