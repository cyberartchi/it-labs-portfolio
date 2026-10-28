# Server Services and Infrastructure Validation

## Objective

Validate the core infrastructure services used by an isolated Windows domain lab and document what each test proves, what it does not prove, and how service dependencies affect troubleshooting.

## Environment

| Component | Role | Address or name |
|---|---|---|
| Proxmox VE | Virtualization platform | Lab host |
| OPNsense (`FW01`) | Default gateway and firewall | `10.10.10.1` |
| Windows Server 2025 (`DC01`) | AD DS, DNS, DHCP, Kerberos, LDAP, SMB, and time service | `10.10.10.10` |
| Windows 11 (`PC01`) | Domain client | DHCP lease `10.10.10.100` |
| Active Directory domain | Internal namespace | `lab.home.arpa` |
| Lab network | Isolated client/server segment | `10.10.10.0/24` |

The OPNsense DHCP service was disabled on the lab segment so that `DC01` was the only intended DHCP server.

## Service Map

The client depends on the services in the following order:

1. A working virtual link and correct network attachment.
2. DHCP for the client address, subnet mask, gateway, and DNS settings.
3. Internal DNS for host and Active Directory service discovery.
4. Time synchronization for reliable Kerberos authentication and log correlation.
5. Reachable service ports for Kerberos, LDAP, SMB, and DNS.
6. Authentication and authorization through Active Directory.
7. Application or resource access.
8. Event logs with comparable timestamps for troubleshooting.

The detailed dependency and failure analysis is available in [the dependency map](notes/dependency-map.md).

## Methodology

Validation was performed from both the client and server with built-in Windows tools. The main commands were:

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4
Resolve-DnsName dc01.lab.home.arpa
nslookup -type=SRV _ldap._tcp.dc._msdcs.lab.home.arpa
Get-DhcpServerv4Scope
Get-DhcpServerv4Lease -ScopeId 10.10.10.0
Get-Service DNS,DHCPServer,NTDS,W32Time,LanmanServer
53,88,389,445 | ForEach-Object { Test-NetConnection dc01.lab.home.arpa -Port $_ }
w32tm /query /source
w32tm /query /status
nltest /dsgetdc:lab.home.arpa
klist
```

The tests were interpreted together. A successful TCP connection proves port reachability, but it does not by itself prove application health, correct protocol negotiation, or user authorization. UDP-based services such as DHCP and NTP were validated through their application-specific results rather than `Test-NetConnection`.

## Evidence

| File | Evidence | Result |
|---|---|---|
| [DNS client settings](screenshots/01-dns-client-settings.png) | Configured IPv4 DNS server on `PC01` | `10.10.10.10` (`DC01`) |
| [DNS resolution](screenshots/02-dns-resolution.png) | Active Directory LDAP SRV lookup | `DC01.lab.home.arpa` returned on TCP 389 |
| [DHCP lease](screenshots/03-dhcp-lease.png) | Server-side lease record | Active lease for `PC01`; client identifier masked |
| [Service status](screenshots/04-service-status.png) | Core Windows services | DNS, DHCP, AD DS, SMB, and Windows Time running automatically |
| [Port connectivity](screenshots/05-port-connectivity.png) | Client-to-server TCP checks | TCP 53, 88, 389, and 445 reachable |
| [Time source](screenshots/06-time-source.png) | Windows Time configuration | `PC01` synchronized from `DC01` with no warning |

Structured results are also recorded in [service-validation.csv](data/service-validation.csv).

## Findings

- `PC01` used the internal DNS server at `10.10.10.10` rather than a public resolver.
- Forward lookups and Active Directory SRV discovery returned the expected domain controller.
- The DHCP scope was active and contained an active lease for `PC01` at `10.10.10.100`.
- DNS, DHCP Server, Active Directory Domain Services, Server, and Windows Time were running with the `Automatic` startup type.
- TCP 53, 88, 389, and 445 were reachable from the client. This supported the DNS, Kerberos, LDAP, and SMB service hypotheses but did not prove authorization to every resource.
- `PC01` used `DC01.lab.home.arpa` as its time source and reported stratum 4 with no warning. A separate check showed that `DC01` was synchronized to an external NTP source.
- Kerberos tickets were present in the standard `user01` logon session. An elevated PowerShell window initially showed no tickets because it used a different Windows logon session.
- Comparable client and server timestamps made event ordering meaningful. A large clock skew would make log correlation unreliable and could cause Kerberos failures.

### DHCP troubleshooting incident

After the virtual machines were started, `PC01` temporarily failed to obtain an address from `DC01`. The investigation confirmed that the DHCP scope was active and had free addresses, OPNsense DHCP was disabled on the segment, the Windows DHCP firewall rules were enabled, and the DHCP service was restarted. A client release/renew then restored the expected lease at `10.10.10.100`.

Because several corrective actions were performed before the successful renewal, the evidence does not prove one single root cause. The incident nevertheless demonstrated a repeatable troubleshooting order: verify topology, eliminate competing DHCP servers, check scope state and capacity, validate firewall rules and service status, and then renew the client lease.

## Security and Privacy

- The work was performed in an isolated home-lab environment.
- No passwords, keys, tokens, QR codes, or public IP addresses are published.
- The DHCP client identifier was masked because it represents a hardware identifier.
- Purpose-built lab names (`DC01`, `PC01`, and `lab.home.arpa`) and private RFC 1918 addresses were retained because they are required to explain the tests.
- Service access should follow network segmentation and least-privilege principles.

## Conclusion

The lab validated DNS, DHCP, Active Directory discovery, Windows service state, selected TCP ports, time synchronization, and the dependencies between them. The most important lesson was that a single successful test is not sufficient evidence of complete service health. Reliable diagnosis requires correlated results from client configuration, server state, port reachability, authentication, authorization, and timestamped logs.

[Back to portfolio](../../README.md)
