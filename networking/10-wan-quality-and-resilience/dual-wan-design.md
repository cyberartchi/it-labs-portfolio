# Dual-WAN Design and Rollback Plan

> Design-only exercise. No second WAN, gateway group, NAT rule, policy-routing rule, or firewall setting was deployed.

## Current Sanitized Inventory

A read-only Proxmox query identified the OPNsense interface attachment points:

~~~text
name: FW01-OPNsense
net0: virtio=<REDACTED>,bridge=vmbr1
net1: virtio=<REDACTED>,bridge=vmbr0
~~~

| Path | Current role |
|---|---|
| `net0/vtnet0 -> vmbr1` | Protected LAN and lab networks |
| `net1/vtnet1 -> vmbr0` | Current fixed-line WAN1 |
| `net2/vtnet2 -> vmbr2` | Planned LTE/5G WAN2 |

## Target Architecture

WAN2 would require:

1. an LTE/5G CPE from a different carrier;
2. a dedicated physical NIC or supported USB Ethernet adapter on the Proxmox host;
3. a new external bridge, such as `vmbr2`;
4. a third virtual adapter, `net2`, attached only to FW01;
5. an OPNsense WAN2 interface and gateway;
6. a gateway group used by the intended LAN firewall rules.

The secondary link should use separate power or UPS coverage where practical. A second logical interface connected to the same CPE or provider would not provide true access redundancy.

## Proposed Gateway Policy

| Setting | WAN1 | WAN2 |
|---|---|---|
| Role | Primary fixed-line service | Backup LTE/5G service |
| Gateway-group tier | Tier 1 | Tier 2 |
| Monitor | Independent external endpoint A | Independent external endpoint B |
| Trigger | Packet Loss or High Latency | Packet Loss or High Latency |
| Normal state | Carries outbound traffic | Available but not preferred |
| WAN1 failure | Marked degraded/down | Carries new outbound traffic |
| WAN1 recovery | Preferred after stable recovery | Returns to backup role |

Gateway reachability must not be the only health signal. The monitoring plan should combine per-gateway reachability with DNS and HTTPS checks so that a responding CPE is not mistaken for complete Internet access.

## Configuration Impact Review

| Area | Required consideration |
|---|---|
| Outbound NAT | Translation must exist for both WAN addresses |
| Policy routing | Selected LAN rules must use the gateway group |
| DNS | Resolver access must remain valid over either path |
| Firewall rules | WAN2 must not inherit unintended inbound exposure |
| State table | Existing sessions may reset when the source path changes |
| VPN | Tunnels may reconnect or require a multi-WAN-specific configuration |
| CGNAT | LTE/5G may support outbound access but prevent unsolicited inbound connections |
| Monitoring | Thresholds must avoid both slow detection and unstable flapping |
| Power | CPE, host, switch, and firewall must remain powered for failover to help |

Dual-WAN failover is not bandwidth bonding. A single existing flow does not automatically gain the sum of both link speeds.

## Planned Maintenance-Window Test

| Step | Action | Expected result | Evidence to collect |
|---|---|---|---|
| 1 | Export OPNsense configuration and create an appropriate FW01 snapshot | Recoverable starting point | Backup identifier and snapshot state |
| 2 | Verify both gateways while WAN1 is preferred | WAN1 online, WAN2 online/standby | Gateway status and route |
| 3 | Start continuous ICMP, DNS, and HTTPS tests | Baseline established | Timestamped test output |
| 4 | Disconnect only the WAN1 uplink | Monitor declares WAN1 degraded/down | Detection timestamp |
| 5 | Observe new connections | DNS and HTTPS recover through WAN2 | Recovery timestamp and WAN2 path |
| 6 | Record interrupted sessions | Existing long-lived sessions may reset | Application notes |
| 7 | Reconnect WAN1 | WAN1 becomes healthy after a stable interval | Gateway status |
| 8 | Verify failback | New traffic returns to WAN1 | Route and external-path evidence |
| 9 | Recheck LAN/VLAN policy | Segmentation remains unchanged | Firewall and connectivity tests |

The report should record detection time, recovery time, lost probes, affected sessions, and whether failback was stable.

## Rollback

If the live implementation causes unexpected behavior:

1. restore the original LAN firewall rules and remove gateway-group policy routing;
2. disable or unassign WAN2;
3. return outbound NAT and DNS behavior to the known WAN1-only configuration;
4. reconnect the original WAN1 physical path;
5. restore the exported OPNsense configuration if needed;
6. restore the FW01 snapshot only when the snapshot and storage state make that safe;
7. verify WAN1 routing, DNS, HTTPS, NAT, and existing segmentation.

## Deployment Status

The architecture and rollback plan are complete. Physical WAN2 installation and failover validation are intentionally deferred to a separate practical lab.
