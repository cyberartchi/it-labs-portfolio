# \# Windows Network Configuration

# 

# \## Objective

# 

# The objective of this exercise was to identify the active Windows network configuration and understand the role of the local IP address, default gateway, DNS servers, and ephemeral ports.

# 

# \## Environment

# 

# \* Windows 11

# \* Windows PowerShell

# \* Command Prompt

# \* Home Wi-Fi network

# 

# \## IP Configuration

# 

# The detailed network configuration was displayed using:

# 

# ```powershell

# ipconfig /all

# ```

# 

# The output was used to identify:

# 

# \* the active Wi-Fi adapter,

# \* the local IPv4 and IPv6 configuration,

# \* the subnet mask or prefix length,

# \* the default gateway,

# \* the configured DNS servers,

# \* whether DHCP was enabled.

# 

# The local IP address identifies the workstation within its network. The default gateway provides access to destinations outside the local subnet. DNS servers translate domain names into IP addresses.

# 

# \## Ephemeral Port Ranges

# 

# The Windows dynamic TCP and UDP port ranges were checked using:

# 

# ```powershell

# netsh int ipv4 show dynamicport tcp

# netsh int ipv4 show dynamicport udp

# ```

# 

# An ephemeral port is a temporary source port selected by the operating system for client communication. It allows response traffic to be delivered to the correct local socket and application.

# 

# The same numerical port can be used independently by TCP and UDP because they are separate transport protocols.

# 

# \## Observations

# 

# \* The workstation used a private address inside the home network.

# \* The default gateway provided connectivity beyond the local subnet.

# \* Multiple DNS server addresses could be configured for availability and protocol support.

# \* Client applications used temporary source ports when communicating with remote services.

# 

# \## Conclusion

# 

# The exercise demonstrated how Windows stores the addressing and resolver configuration required for network communication. It also showed that client connections normally use ephemeral source ports, while servers commonly use stable destination ports associated with particular services.

# 

# \## Security and Privacy

# 

# Command outputs were reviewed before publication. Public IP addresses, device names, usernames, and other identifying information were excluded or masked where necessary.

# 

# \[Back to module](../README.md)

# 

