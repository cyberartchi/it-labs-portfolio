# Wireless Environment Analysis

## Objective

Inspect the active Wi-Fi connection, identify the wireless adapter's supported standards, and verify local gateway connectivity, Internet reachability, and DNS resolution.

## Environment

- Client: Windows 11
- Connection: Wi-Fi
- Tools: Command Prompt, `netsh`, `ping`, and `nslookup`

## 1. Active Wi-Fi Interface

Command:

```powershell
netsh wlan show interfaces
```

Observed values:

- Radio band: 5 GHz
- Channel: 128
- Active radio type: 802.11n
- Authentication: WPA2-Personal
- Cipher: CCMP
- Receive rate: 120 Mbps
- Transmit rate: 52 Mbps
- Signal: 45%
- RSSI: -79 dBm

The connection was operational, although the signal level indicated a relatively weak Wi-Fi signal.

![Active Wi-Fi interface](screenshots/01-wifi-interface.png)

## 2. Wireless Adapter Capabilities

Command:

```powershell
netsh wlan show drivers
```

The RZ616 Wi-Fi 6E 160MHz adapter reported support for:

- 802.11b/a/g/n/ac/ax
- 2.4 GHz, 5 GHz, and 6 GHz operation
- WPA3 security
- 802.11w management frame protection

The adapter supports newer standards than the 802.11n mode used by the active connection.

![Wireless adapter capabilities](screenshots/02-wifi-driver-capabilities.png)

## 3. Local Gateway Latency

Command:

```powershell
ping -n 10 192.168.0.1
```

Results:

- Packet loss: 0%
- Minimum latency: 4 ms
- Maximum latency: 8 ms
- Average latency: 6 ms

This confirmed stable communication between the client and the local gateway during the test.

![Gateway latency](screenshots/03-gateway-latency.png)

## 4. Internet Latency

Command:

```powershell
ping -n 10 1.1.1.1
```

Results:

- Packet loss: 0%
- Minimum latency: 31 ms
- Maximum latency: 41 ms
- Average latency: 35 ms

The test confirmed IP connectivity beyond the local network.

![Internet latency](screenshots/04-internet-latency.png)

## 5. DNS Resolution

Command:

```powershell
nslookup example.com
```

The DNS resolver successfully returned IPv4 and IPv6 addresses for `example.com`. The message `Server: Unknown` indicated that the resolver address did not have a reverse DNS name; it did not indicate a failed lookup.

![DNS resolution](screenshots/05-dns-resolution.png)

## Conclusion

The client had working local and Internet connectivity with no packet loss in the captured tests, and DNS resolution completed successfully. The adapter supports modern Wi-Fi standards and security features, but the active connection used 802.11n and showed a relatively weak signal of 45% (`-79 dBm`). This suggests that signal quality or access-point placement could be investigated if performance becomes inconsistent.
