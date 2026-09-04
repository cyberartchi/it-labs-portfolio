# \# IP, TCP, UDP and DNS Analysis

# 

# \## Module Overview

# 

# This module documents practical analysis of IP configuration, TCP and UDP communication, DNS traffic, protocol encapsulation, network flows, and basic troubleshooting.

# 

# The exercises were completed on a Windows 11 workstation using PowerShell, Command Prompt, and Wireshark.

# 

# \## Learning Objectives

# 

# \* Identify local IP configuration, the default gateway, and DNS servers.

# \* Understand the role of source and destination ports.

# \* Identify Windows ephemeral port ranges.

# \* Compare UDP-based DNS traffic with TCP-based HTTPS traffic.

# \* Recognize the TCP three-way handshake.

# \* Identify Ethernet, IP, TCP, UDP, DNS, and TLS layers.

# \* Describe a network flow using a five-tuple.

# \* Separate diagnostic facts, hypotheses, and follow-up tests.

# 

# \## Practical Exercises

# 

# | Exercise                      | Status    | Documentation                                      |

# | ----------------------------- | --------- | -------------------------------------------------- |

# | Windows Network Configuration | Completed | \[View exercise](01-windows-network-configuration/) |

# | Wireshark Traffic Analysis    | Completed | \[View exercise](02-wireshark-traffic-analysis/)    |

# | Network Diagnostic Scenarios  | Completed | \[View exercise](03-network-diagnostic-scenarios/)  |

# 

# \## Environment

# 

# \* Windows 11

# \* Windows PowerShell

# \* Command Prompt

# \* Wireshark

# \* Home Wi-Fi network

# 

# \## Key Outcomes

# 

# The module demonstrated that an IP address identifies a host or interface, while transport protocols and port numbers identify communication endpoints. DNS queries and responses were matched using the DNS transaction ID, and a TCP connection was traced through the SYN, SYN-ACK, and ACK sequence.

# 

# Packet inspection also showed that encryption protects application data while network and transport metadata, including IP addresses, ports, and TCP flags, remains visible.

# 

# \## Security and Privacy

# 

# Captured traffic was limited to controlled activity generated on the test workstation. Sensitive network information was reviewed and masked where necessary before publication.

# 

# \[Back to portfolio](../../README.md)

# 

