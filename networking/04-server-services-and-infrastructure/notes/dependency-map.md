# Infrastructure Dependency Map

## Normal dependency chain

| Order | Layer or service | Evidence used | Failure impact |
|---:|---|---|---|
| 1 | Virtual link and network attachment | Proxmox NIC and lab-segment configuration | No DHCP discovery or IP connectivity |
| 2 | DHCP and IPv4 configuration | Scope state, lease table, and client address | Missing or APIPA address; no normal access to domain services |
| 3 | Internal DNS | Client DNS setting, A lookup, and LDAP SRV lookup | Domain controller and internal resources cannot be located reliably |
| 4 | Time synchronization | `w32tm /query /source` and `/status` | Kerberos failures and unreliable event correlation when skew is excessive |
| 5 | Port reachability | TCP checks for 53, 88, 389, and 445 | Individual services cannot be contacted even when addressing works |
| 6 | Authentication | `nltest` and `klist` | Identity cannot be validated and domain logon/service access fails |
| 7 | Authorization | Group membership, share permissions, and NTFS ACLs | Correct credentials but `Access denied` for a protected resource |
| 8 | Application and logs | Resource test and timestamped event records | Service failure or insufficient evidence to reconstruct the sequence |

## Diagnostic scenarios

### 1. PC01 uses a public DNS resolver

Internet names may still resolve, but the public resolver does not host the private `lab.home.arpa` zone or its Active Directory SRV records. Domain-controller discovery, Group Policy, Kerberos, and internal resource access may fail. The first checks are the configured DNS server, an internal A lookup, an LDAP SRV lookup, and `nltest /dsgetdc:lab.home.arpa`.

### 2. The DHCP lease expires and the scope has no free addresses

The client cannot obtain a new lease. After the previous lease becomes unusable, Windows may assign an APIPA address in `169.254.0.0/16`. The correct evidence is the client configuration, renewal output, scope statistics, exclusions, reservations, and the server lease table. This symptom resembles the real DHCP incident observed in the lab, but the completed checks showed that the actual scope was active and had capacity.

### 3. PC01 differs from DC01 by several minutes

Kerberos normally tolerates only limited clock skew. Excessive skew can cause domain authentication, SMB, LDAP, and Group Policy failures. It also makes event correlation misleading. The lab's measured offset was only about 17 milliseconds, so this failure condition was not present.

### 4. TCP 445 is blocked only between VLANs

SMB may work for systems on the same VLAN but fail across routed segments. DNS and ICMP could still succeed. The evidence should include a source-to-destination TCP 445 test plus the relevant OPNsense interface rule and firewall log. The current lab uses one `10.10.10.0/24` segment, so this scenario was analyzed without changing the firewall.

### 5. The account password is correct but the user is not in the required group

Authentication can succeed while authorization fails. A Kerberos ticket and reachable TCP 445 show that identity and connectivity are working, but a share may still return `Access denied`. The next checks are `whoami /groups`, Active Directory group membership, SMB share permissions, and NTFS ACLs. The fault is an authorization issue rather than a network or password issue.

## Troubleshooting principle

Test dependencies from the bottom upward and distinguish facts from hypotheses. For example, a timeout proves that no response was observed; it does not by itself prove that a firewall caused the failure. Each hypothesis should be followed by a test and a clear success or failure criterion.
