# Evidence 01 - Address and Route Inventory

> Sanitized transcript. Full MAC addresses, DHCP identifiers, and complete link-local IPv6 addresses are intentionally omitted.

## Windows PC01

Commands:

~~~powershell
ipconfig /all
Get-NetIPConfiguration -Detailed
Get-NetIPAddress | Sort-Object AddressFamily,InterfaceIndex |
    Format-Table InterfaceAlias,AddressFamily,IPAddress,PrefixLength,AddressState
~~~

Relevant result:

~~~text
Computer name:             PC01
Interface:                 Ethernet
Adapter:                   Red Hat VirtIO Ethernet Adapter
Network profile:           lab.home.arpa
IPv4 address:              10.10.10.100/24
IPv4 default gateway:      10.10.10.1
DNS server:                10.10.10.10
DHCP:                      Enabled
IPv4 MTU:                  1500
IPv6 link-local address:   <PC01_LINK_LOCAL_IPV6>/64
IPv6 default gateway:      None
IPv6 connectivity:         NoTraffic
MAC address:               <PC01_MAC>
DHCPv6 DUID:               <OMITTED>
~~~

Selected IPv4 routes:

~~~text
0.0.0.0/0       -> 10.10.10.1  interface 14
10.10.10.0/24   -> on-link     interface 14
~~~

Interpretation:

- `10.10.10.100/24` places PC01 in `10.10.10.0/24`.
- `10.10.10.1` is used for destinations outside that subnet.
- IPv6 is enabled, but the host has only a link-local address and no IPv6 default route.

## Debian test01

Commands:

~~~bash
ip -br address
ip -4 route
ip -6 route
~~~

Relevant result:

~~~text
lo           UNKNOWN  127.0.0.1/8  ::1/128
eth0@if16    UP       10.10.30.151/24  <TEST01_LINK_LOCAL_IPV6>/64

default via 10.10.30.1 dev eth0
10.10.30.0/24 dev eth0 proto kernel scope link src 10.10.30.151

fe80::/64 dev eth0 proto kernel metric 256 pref medium
~~~

`eth0@if16` identifies the container-side interface and its peer interface index; it does not mean VLAN 16.

Interpretation:

- the connected IPv4 route covers the local VLAN 30 subnet;
- the IPv4 default route points to OPNsense at `10.10.30.1`;
- the only IPv6 route is link-local `fe80::/64`;
- no IPv6 default route is present.

## Final Comparison

| Host | IPv4 subnet | IPv4 default gateway | IPv6 state |
|---|---|---|---|
| PC01 | `10.10.10.0/24` | `10.10.10.1` | Link-local only, no default route |
| test01 | `10.10.30.0/24` | `10.10.30.1` | Link-local only, no default route |
