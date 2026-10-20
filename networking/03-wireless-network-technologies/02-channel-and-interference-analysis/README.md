# Channel and Interference Analysis

## Objective

Survey nearby Wi-Fi networks and compare their frequency bands, channels, signal levels, and radio standards to identify possible channel overlap or interference.

## Environment

- Client: Windows 11
- Tool: `netsh`
- Scan type: Passive snapshot of networks visible from the client location

## Command

```powershell
netsh wlan show networks mode=bssid
```

## Results

Network names and access-point identifiers were masked before publication.

| Anonymized network | Band | Channel | Signal | Radio type |
|---|---:|---:|---:|---|
| Network 1, access point 1 | 5 GHz | 36 | 9% | 802.11ac |
| Network 1, access point 2 | 2.4 GHz | 11 | 21% | 802.11n |
| Network 2 | 2.4 GHz | 6 | 42% | 802.11n |
| Network 3 | 5 GHz | 128 | 51% | 802.11n |

The visible 2.4 GHz networks used channels 6 and 11, which are non-overlapping channels. The visible 5 GHz networks used channels 36 and 128, which were widely separated. No same-channel conflict was visible in this scan.

![Wireless channel scan](screenshots/01-channel-scan.png)

## Conclusion

The captured snapshot showed low apparent channel contention among the detected networks. The result is specific to the test location and time because nearby access points, client activity, and signal levels can change. A repeated scan at different times would provide stronger evidence for long-term channel planning.
