# DNS Configuration and Analysis

## Objective

Configure and validate DNS resource records in a Windows domain lab, observe how the Windows DNS client cache handles TTL values, and inspect a complete DNS query and response at packet level.

## Environment

| Component | Role | Address or name |
|---|---|---|
| Proxmox VE | Virtualization platform | Lab host |
| OPNsense (`FW01`) | Default gateway and firewall | `10.10.10.1` |
| Windows Server 2025 (`DC01`) | Active Directory domain controller and DNS server | `10.10.10.10` |
| Windows 11 (`PC01`) | Domain client and packet-capture host | `10.10.10.100` |
| Active Directory DNS zone | Internal namespace | `lab.home.arpa` |
| Lab network | Isolated client/server segment | `10.10.10.0/24` |

## Methodology

### 1. DNS resource records

A temporary A record was created for `web01.lab.home.arpa`, pointing to `10.10.10.10`. A CNAME record named `portal.lab.home.arpa` was then configured as an alias for `web01.lab.home.arpa` with a five-minute TTL.

The records were queried to confirm both alias resolution and the final IPv4 address. The temporary records were removed after validation so that the lab zone returned to its original state.

### 2. DNS resolution and client cache

The client resolved the domain controller's A record and inspected the matching Windows DNS cache entry:

```powershell
Resolve-DnsName dc01.lab.home.arpa -Type A
Get-DnsClientCache -Entry "dc01.lab.home.arpa" | Format-Table Entry,Type,TimeToLive,Data
```

Two cache observations showed the TTL decreasing from `1051` to `889` seconds. This demonstrated that Windows reused the cached answer while its remaining lifetime counted down.

### 3. Packet capture

Wireshark captured a fresh lookup from `PC01` to `DC01`. The following display filter isolated the exchange:

```text
dns.qry.name == "dc01.lab.home.arpa"
```

The DNS query and response used the same transaction ID, `0x81b4`, allowing the two packets to be correlated.

## Evidence

| File | Evidence | Result |
|---|---|---|
| [CNAME record resolution](screenshots/01-cname-record-resolution.png) | Temporary alias lookup | `portal.lab.home.arpa` resolved through `web01.lab.home.arpa` to `10.10.10.10` |
| [A record resolution](screenshots/02-a-record-resolution.png) | Direct host lookup | `dc01.lab.home.arpa` returned A record `10.10.10.10` |
| [DNS cache before](screenshots/03-dns-cache-before.png) | First cache observation | Cached A record with `1051` seconds remaining |
| [DNS cache after](screenshots/04-dns-cache-after.png) | Later cache observation | Same cached A record with `889` seconds remaining |
| [DNS query and response](screenshots/05-dns-query-response.png) | Filtered packet list | One client query and one successful server response |
| [DNS packet analysis](screenshots/06-dns-packet-analysis.png) | Decoded response packet | UDP ports, A answer, TTL, addresses, and response timing verified |

## Packet Analysis

| Field | Observed value | Interpretation |
|---|---|---|
| Transport protocol | UDP | The answer fit in a normal UDP DNS response |
| Client endpoint | `10.10.10.100:54685` | `PC01` used an ephemeral source port |
| Server endpoint | `10.10.10.10:53` | `DC01` provided the DNS service |
| Query type | A | The client requested an IPv4 host address |
| Answer | `dc01.lab.home.arpa → 10.10.10.10` | The authoritative lab record resolved correctly |
| Transaction ID | `0x81b4` | The response matched the request |
| Response code | No error | The DNS server processed the query successfully |
| Answer TTL | `1200` seconds | The answer could be cached for up to 20 minutes |
| Response time | Approximately `981.3 µs` | The isolated local exchange completed in under 1 ms |

The response five-tuple was:

```text
UDP 10.10.10.10:53 → 10.10.10.100:54685
```

## Findings

- The `lab.home.arpa` zone returned the expected A and CNAME data.
- A CNAME provides an alias; the client still needs the target host's A record to obtain an IPv4 address.
- Windows stored the A answer locally and reduced its remaining TTL over time.
- A cached result avoids an unnecessary DNS request until the entry expires or the cache is cleared.
- The query travelled from the client to UDP port 53, and the response returned to the client's ephemeral port.
- The transaction ID and reversed endpoint pair linked the response to the original query.
- A successful lookup proves that this record resolved correctly at that time. It does not by itself prove the health of every DNS zone, Active Directory service, or external name resolution path.

## Security and Privacy

- The work was performed in an isolated home-lab environment.
- No credentials, tokens, public IP addresses, personal domains, or production hostnames are published.
- Purpose-built lab names and private RFC 1918 addresses were retained because they are necessary to explain the DNS exchange.
- The selected screenshots required no masking and were published without altering their technical content.

## Conclusion

The lab connected three views of the same DNS process: server-side records, the Windows client cache, and packets on the network. The results confirmed correct A and CNAME resolution, demonstrated TTL-based caching, and showed how a DNS request and response can be verified through transaction IDs, endpoints, ports, response codes, and answer data.

[Back to portfolio](../../README.md)
