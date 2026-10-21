# Wireless Network Technologies

This module documents practical wireless-network analysis performed on a Windows 11 client. It covers Wi-Fi interface and driver inspection, IP and DNS connectivity, channel selection, 2.4 GHz versus 5 GHz performance, Bluetooth hardware and driver inventory, and a basic Bluetooth range test.

## Lab Environment

- Windows 11 client
- RZ616 Wi-Fi 6E 160MHz / Bluetooth adapter
- Dual-band home wireless router
- Windows PowerShell and Command Prompt
- `netsh`, `Get-NetAdapter`, `Get-NetIPConfiguration`, `ipconfig`, `ping`, `nslookup`, and Plug and Play cmdlets

## Labs

| Lab | Scope | Main result |
|---|---|---|
| [Wireless Environment Analysis](01-wireless-environment-analysis/) | Active Wi-Fi connection, adapter capabilities, DHCP, DNS, gateway, and Internet connectivity | The client had working local, Internet, and DNS connectivity; the adapter supported 2.4, 5, and 6 GHz operation. |
| [Channel and Interference Analysis](02-channel-and-interference-analysis/) | Nearby networks, channel-change experiment, DFS observation, and 2.4/5 GHz comparison | Channel 36 did not improve the connection. After restoring 5 GHz channel 128, the measured download rate was 130 Mbps versus 16 Mbps on 2.4 GHz. |
| [Bluetooth and NFC Inventory](03-bluetooth-and-nfc-inventory/) | Bluetooth device and driver inventory, NFC detection, and a basic range test | Bluetooth was detected and operational; stability decreased behind one wall. No NFC/proximity device was detected. |

Detailed numerical results are available in [the measurement dataset](02-channel-and-interference-analysis/data/measurements.csv).

## Key Findings

- A supported Wi-Fi standard is not necessarily the standard currently used by a connection.
- Link rates shown by Windows are negotiated PHY rates, not measured Internet throughput.
- A theoretically suitable channel change does not guarantee better real-world performance.
- 5 GHz was substantially faster and more stable than 2.4 GHz at the test location.
- A temporary disappearance of the 5 GHz network after selecting channel 128 was consistent with DFS channel-availability checking. This is an inference from the observed behavior, not proof of the router's internal decision.
- Physical obstacles reduced Bluetooth stability during the basic range test.

## Concept Review

### Why can Bluetooth and Wi-Fi interfere in the 2.4 GHz band?

Both technologies may transmit in the unlicensed 2.4 GHz ISM band. When transmissions overlap in time and frequency, they compete for airtime and may require retries. Bluetooth frequency hopping reduces this problem but does not remove it completely.

### How can a passive NFC tag work without a battery?

A passive NFC tag receives energy from the reader's electromagnetic field through inductive coupling. It rectifies that energy to power its chip and sends data back using load modulation.

### Which report values can reveal equipment or location?

SSIDs, BSSIDs and other MAC addresses, adapter GUIDs, public IP addresses, computer and user names, profile names, serial numbers, file paths, and visible geographic clues can identify a device, owner, or location.

### Does short wireless range guarantee security?

No. Short range reduces exposure but does not replace authentication, encryption, or user verification. Relay attacks, malicious NFC tags, unauthorized close-range reads, device loss, and social engineering remain possible.

## Privacy

Sensitive identifiers were removed from the published evidence. Only the affected values were covered; the remaining command output was left unchanged. Private RFC 1918 addresses were retained where they were necessary to explain local-network tests.

