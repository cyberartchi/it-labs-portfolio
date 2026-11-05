# DHCP Address Management and Analysis

## Objective

Validate a Windows DHCP server from the service, authorization, interface, and scope perspectives, then capture and analyze a client lease renewal at packet level.

## Environment

| Component | Role | Address or name |
|---|---|---|
| Proxmox VE | Virtualization platform | Lab host |
| OPNsense (`FW01`) | Default gateway and firewall | `10.10.10.1` |
| Windows Server 2025 (`DC01`) | Active Directory domain controller and DHCP server | `10.10.10.10` |
| Windows 11 (`PC01`) | Domain client and packet-capture host | `10.10.10.100` |
| Active Directory domain | Internal namespace | `lab.home.arpa` |
| Lab network | Isolated client/server segment | `10.10.10.0/24` |

## Methodology

### 1. DHCP server baseline

The server was checked from several independent perspectives:

```powershell
Get-Service DHCPServer
Get-DhcpServerInDC
Get-DhcpServerv4Binding -ComputerName DC01
```

The DHCP Server service was running, `DC01` was authorized in Active Directory, and the IPv4 service was bound to the expected Ethernet interface at `10.10.10.10`.

### 2. Scope configuration and utilization

The active scope and its current statistics were inspected:

```powershell
Get-DhcpServerv4Scope -ComputerName DC01 | Format-Table ScopeId,State,StartRange,EndRange,LeaseDuration
Get-DhcpServerv4ScopeStatistics -ComputerName DC01
```

The `10.10.10.0/24` scope was active. Its pool covered `10.10.10.100` through `10.10.10.200`, with an eight-day lease duration. At the time of observation, one address was in use and 100 were free.

### 3. Client renewal and packet capture

Wireshark captured the Ethernet interface on `PC01`. The following display filter isolated DHCP traffic:

```text
udp.port == 67 || udp.port == 68
```

While the capture was running, the client renewed its existing lease:

```powershell
ipconfig /renew "Ethernet"
```

The capture showed a DHCP Request from the client followed by a DHCP ACK from the server. Both packets used transaction ID `0xf52d40f9`, correlating them as one renewal exchange.

## Evidence

| File | Evidence | Result |
|---|---|---|
| [DHCP scope configuration](screenshots/01-dhcp-scope-configuration.png) | Scope state, pool, and lease duration | Active `10.10.10.0/24` scope; pool `10.10.10.100–10.10.10.200`; eight-day lease |
| [DHCP scope statistics](screenshots/02-dhcp-scope-statistics.png) | Current address utilization | One address in use, 100 free, no reservations or pending offers |
| [DHCP renewal analysis](screenshots/03-dhcp-renewal-analysis.png) | Filtered Request/ACK exchange and decoded ACK | Client renewal succeeded and the server returned DHCP configuration options |

## Packet Analysis

| Field | Observed value | Interpretation |
|---|---|---|
| Transport protocol | UDP | DHCP uses connectionless transport |
| Client endpoint | `10.10.10.100:68` | `PC01` sent the renewal request |
| Server endpoint | `10.10.10.10:67` | `DC01` provided the DHCP service |
| Client message | DHCP Request | The client requested renewal of its existing lease |
| Server message | DHCP ACK | The server approved the lease and returned configuration options |
| Transaction ID | `0xf52d40f9` | The ACK matched the Request |
| Assigned client address | `10.10.10.100` | The existing client lease remained valid |
| Server Identifier | `10.10.10.10` | The response came from the expected DHCP server |
| Subnet mask | `255.255.255.0` | The client remained in the `10.10.10.0/24` network |
| ACK options | Lease timing, router, DNS server, and domain name | The server supplied the parameters required for normal client networking |

## Findings

- The DHCP service was running, authorized in Active Directory, and bound to the correct server interface.
- The configured pool contained 101 addresses; one was leased and 100 remained available.
- DHCP used UDP port 68 on the client and UDP port 67 on the server.
- The matching transaction ID linked the DHCP ACK to the client's DHCP Request.
- Only Request and ACK appeared because this was a renewal of an existing lease. A new client without a valid lease would normally perform the full DORA sequence: Discover, Offer, Request, and Acknowledge.
- A successful ACK proves that this client renewed its lease from the expected server at that moment. It does not by itself prove DHCP availability on every VLAN or rule out the presence of another DHCP server elsewhere.

## Security and Privacy

- The work was performed in an isolated home-lab environment.
- No credentials, tokens, public IP addresses, personal domains, or production hostnames are published.
- Purpose-built lab names and private RFC 1918 addresses were retained because they are necessary to explain the exchange.
- The selected screenshots required no masking and were published without altering their technical content.

## Conclusion

The lab connected server-side DHCP configuration with a packet-level client renewal. It confirmed service readiness, Active Directory authorization, interface binding, scope capacity, and a successful Request/ACK exchange between `PC01` and `DC01`.

[Back to portfolio](../../README.md)
