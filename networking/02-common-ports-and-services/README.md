# Common Ports and Network Services

## Module Overview

This module focuses on commonly used TCP and UDP ports, the services associated with them, secure protocol alternatives, Windows port inventory, process mapping, and service-oriented troubleshooting.

Port numbers are treated as diagnostic indicators rather than complete proof of a service. Validation may require process information, service configuration, protocol negotiation, firewall analysis, or packet inspection.

## Learning Objectives

- distinguish an IP address, transport protocol, port, socket, and process;
- associate common ports with their typical network services;
- recognize secure and modern alternatives to legacy protocols;
- identify TCP listeners and UDP endpoints in Windows;
- map selected ports to owning processes;
- analyze established connections and their 5-tuples;
- create evidence-based service hypotheses;
- understand how firewall rules use protocols, ports, interfaces, and source addresses.

## Services Covered

| Service | Common Port or Ports | Primary Role | Secure or Modern Alternative |
|---|---|---|---|
| FTP | TCP 20/21 | File transfer | SFTP over SSH or FTPS |
| SSH | TCP 22 | Encrypted remote administration | Keys, MFA, VPN or bastion access |
| Telnet | TCP 23 | Legacy remote terminal | SSH |
| SMTP | TCP 25 | Server-to-server mail transfer | Submission on 587 with STARTTLS or implicit TLS on 465 |
| DNS | UDP/TCP 53 | Name resolution | DoT, DoH, and DNSSEC validation |
| DHCP | UDP 67/68 | Automatic IPv4 configuration | DHCP relay, reservations, and network access controls |
| HTTP | TCP 80 | Unencrypted web traffic | HTTPS |
| HTTPS | TCP 443 | HTTP protected by TLS | HTTP/3 over QUIC on UDP 443 |
| POP3 | TCP 110 | Mail retrieval | POP3S on TCP 995 |
| IMAP | TCP 143 | Mailbox synchronization | IMAPS on TCP 993 |
| NetBIOS | UDP 137-138, TCP 139 | Legacy Windows naming and sharing | Direct-hosted SMB and modern DNS |
| SMB | TCP 445 | File, printer, and Windows service sharing | SMB 3 with signing and encryption where appropriate |
| LDAP | TCP/UDP 389 | Directory services | LDAPS on TCP 636 or LDAP with appropriate TLS protection |
| RDP | TCP/UDP 3389 | Windows remote desktop | VPN, RD Gateway, MFA, and restricted source access |

## Practical Exercises

| Exercise | Status | Documentation |
|---|---|---|
| Windows Port Inventory | Completed | [View exercise](01-windows-port-inventory/) |
| Wireshark Port Analysis | Covered in the previous module | [View related exercise](../01-ip-tcp-udp-analysis/02-wireshark-traffic-analysis/) |
| Proxmox and Windows Server Service Map | Planned | Directory will be added after implementation |
| Service Diagnostic Scenarios | Planned | Directory will be added after completion |

Only completed exercises receive their own directories. Planned work is listed here without empty placeholder folders.

## Environment

- Windows 11
- Windows PowerShell
- Wireshark
- Home-lab network
- Proxmox and Windows Server planned for a later exercise

## Key Principle

An open or listening port suggests a possible service, but it does not prove the application protocol, security level, or authorization scope. Reliable validation should combine several sources of evidence.

Examples include:

- the transport protocol and port number;
- the local interface and connection state;
- the owning process;
- the related Windows service;
- firewall and ACL configuration;
- protocol negotiation or packet analysis.

## Security and Privacy

Exercises are performed in a controlled environment. Credentials, public IP addresses, company information, usernames, and identifying host or domain names are excluded from the published documentation.

[Back to portfolio](../../README.md)
