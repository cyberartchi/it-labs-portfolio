# Bluetooth and NFC Inventory

## Objective

Identify the Bluetooth adapter currently available to Windows and determine whether the system exposes an NFC or proximity device.

## Environment

- Client: Windows 11
- Tool: Windows PowerShell
- Data source: Windows Plug and Play device inventory

## 1. Bluetooth Adapter

Command:

```powershell
Get-PnpDevice -Class Bluetooth -PresentOnly |
    Format-Table Status, FriendlyName
```

Result:

- Device: RZ616 Bluetooth(R) Adapter
- Status: OK

The result confirms that Windows detected the Bluetooth adapter and reported it as operational at the Plug and Play level.

![Bluetooth adapter](screenshots/01-bluetooth-adapter.png)

## 2. NFC and Proximity Check

Command:

```powershell
$nfc = Get-PnpDevice | Where-Object {
    $_.Class -eq "Proximity" -or
    $_.FriendlyName -match "NFC|Near Field"
}
$nfc
```

The command returned no devices. This means that Windows did not detect an NFC or proximity device matching the query.

![NFC check](screenshots/02-nfc-check.png)

## Conclusion

The computer has a Bluetooth adapter that Windows reports as available and healthy. No NFC or proximity hardware was detected. This lab verifies hardware inventory and device availability; it does not test Bluetooth pairing, range, throughput, or data transfer.
