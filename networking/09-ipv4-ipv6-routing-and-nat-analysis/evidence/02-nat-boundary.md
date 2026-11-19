# Evidence 02 - NAT Boundary Analysis

> Sanitized packet-capture transcript. Layer 2 hardware addresses and the subscriber's public address are omitted.

## Test Traffic

Command on PC01:

~~~powershell
curl.exe -I https://example.com
~~~

The command generated a short HTTPS connection to TCP port 443.

## Capture Filter Correction

The first LAN capture used host `10.10.10.100` and TCP port 443. Because the OPNsense web interface was also being accessed from PC01 over HTTPS, the capture initially contained local management traffic:

~~~text
10.10.10.100:<EPHEMERAL> -> 10.10.10.1:443
10.10.10.1:443 -> 10.10.10.100:<EPHEMERAL>
~~~

That flow terminated on the firewall itself and was not subject to outbound NAT. The LAN filter was refined to:

~~~text
10.10.10.100 and not 10.10.10.1
~~~

with IPv4, TCP, and port 443 selected.

## LAN - Before NAT

Interface: `LAN / vtnet0`

Representative result:

~~~text
10.10.10.100:<EPHEMERAL> -> 104.20.23.154:443
104.20.23.154:443 -> 10.10.10.100:<EPHEMERAL>
~~~

Finding: PC01 remained the Layer 3 source on the protected LAN side.

At Layer 2, the Ethernet frame was addressed to the OPNsense LAN MAC because the public destination was outside `10.10.10.0/24`. At Layer 3, the destination remained the remote server address.

## WAN - After NAT

Interface: `WAN / vtnet1`

Representative result:

~~~text
10.0.0.68:<EPHEMERAL> -> 172.66.147.243:443
172.66.147.243:443 -> 10.0.0.68:<EPHEMERAL>
~~~

Finding: OPNsense replaced the internal source with its WAN address, `10.0.0.68`.

The LAN and WAN samples came from separate `curl` executions. `example.com` returned two IPv4 addresses, so the destination IP and ephemeral port differed. These samples demonstrate the NAT boundary, but they are not presented as a packet-for-packet mapping of one TCP session.

## Interpretation

| Function | Observed behavior |
|---|---|
| Routing | OPNsense forwarded traffic from LAN to WAN |
| Source NAT | Source changed from `10.10.10.100` to `10.0.0.68` |
| Firewall policy | The outbound HTTPS flow was permitted |
| State tracking | Matching replies returned through OPNsense to PC01 |
| Upstream NAT | Orange later translated `10.0.0.68` to an omitted public address |

No firewall or NAT configuration was changed for the purpose of producing the capture.
