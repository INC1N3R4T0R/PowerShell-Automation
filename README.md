# PowerShell-Automation

PowerShell scripts for IT professionals to deploy through an RMM platform for Windows device monitoring, inventory, and remediation. The scripts are written with **Datto RMM (CentraStage)** conventions in mind, but most of the logic works in any RMM that can run PowerShell as SYSTEM.

## Contents

| Script | Type | Purpose | Writes UDF | Alerts (exit 1) when |
|---|---|---|---|---|
| [Active NICs](#active-nics) | Check | Lists network adapters that are up | - | Never |
| [Current AV](#current-av) | Check | Detects installed AV/EDR products | `Custom24` | Never |
| [Check DHCP on All NICs](#check-dhcp-on-all-nics) | Monitor | Flags static IP configuration | `Custom19` | A static IP is found |
| [.OST Size](#ost-size) | Monitor | Flags oversized Outlook data files | - | A file is over 45 GB |
| [Block Specific Apps](#block-specific-apps) | Remediation | Removes AnyDesk | - | AnyDesk was found and removed |
| [View C: Drive Size](#view-c-drive-size) | Monitor | Records total C: capacity | `Custom20` | Value retrieved (see notes) |
| [Full System Report](#full-system-report) | Script | Belarc-style printable inventory | - | - |
| [Re-map Network Drives](#re-map-network-drives) | Script | Refreshes mapped drives without reboot or logoff | - | - |

## How these work with an RMM

- **Result markers:** Output between `<-Start Result->` and `<-End Result->` is what Datto RMM captures as the monitor result.
- **Exit codes:** Monitors exit `0` when healthy and `1` when an alert should be raised. Configure the alert trigger on the exit code or the captured result.
- **UDFs:** Several scripts write to `HKLM:\SOFTWARE\CentraStage` as `Custom19`, `Custom20`, and `Custom24`. These map to Datto RMM user-defined fields. Change the value names to match your own UDF layout.
- **Run context:** Run as SYSTEM with the 64-bit PowerShell host. Several scripts need admin rights (registry writes, BitLocker, TPM, all user profiles).

## Scripts

### Active NICs
Returns all network adapters currently in the `Up` state.

```powershell
Get-NetAdapter | Where-Object { $_.Status -eq "Up" }
```

### Current AV
Checks services and processes for Trend Micro, Avast, Datto EDR, Bitdefender, Sophos, Malwarebytes, and McAfee. It also looks for Datto EDR install folders (`C:\ProgramData\CentraStage\AEMAgent\RMM.AdvancedThreatDetection` and `C:\Program Files\infocyte\agent`). Detected products are written as a comma-separated list to `Custom24`, or `No AV found` if nothing matches.

To add a vendor, add a detection flag, a `Check-AV` call with a match pattern, and an entry in the final list.

### Check DHCP on All NICs
Inspects all IP-enabled adapters, skipping any whose description matches `VPN|Tailscale|SonicWALL`. If a static adapter is found, the first one's name, IP, and DNS servers are written to `Custom19` and the script exits `1`. Otherwise it writes `DHCP` and exits `0`.

### .OST Size
Scans each user profile under `C:\Users` (excluding Public and Default) in `AppData\Local\Microsoft\Outlook` for `.ost` and `.pst` files over the threshold. The default threshold is 45 GB, set by `$SizeThresholdGB`. It reports the first offender as `STATUS=<user> has OST file <name> (<size> GB)` and exits `1`.

### Block Specific Apps
Remediation script that kills AnyDesk processes, stops and deletes its services, removes its folders (Program Files, AppData, LocalAppData, ProgramData), and deletes its uninstall registry entries. It exits `1` with a summary of what was removed, or `0` with `No instances found`.

> **Warning:** This is destructive and forced. Test on a single device first. The pattern can be reused for other applications by changing the match strings and folder paths.

### View C: Drive Size
Calculates total C: capacity (used plus free) in GB and writes it to `Custom20`.

### Full System Report
Generates a Belarc-style text report saved to `C:\Forvan\SystemReport_<timestamp>.txt` using `Start-Transcript`. Sections:

- System security profile and summary (make, model, serial, OS, BitLocker status)
- Local users and local administrators
- Windows 11 readiness (Secure Boot, TPM, partition style)
- RAM slots and modules
- Physical disks and logical volumes
- Network configuration
- Saved Wi-Fi profiles
- Installed software and printers
- Antivirus (Defender) and firewall status
- Hardware devices (storage controllers, GPU, monitors, audio, other)

> **Security note:** The Wi-Fi section runs `netsh wlan show profile ... key=clear` and writes saved Wi-Fi passwords into the report in plain text. Remove that block, or restrict access to `C:\Forvan`, before deploying broadly.

### Re-map Network Drives
Deletes the `S:` and `I:` mappings, runs `gpupdate /force`, and re-maps both to file shares. Replace `\\FILESERVER1\Fullpath\` and `\\FILESERVER2\Fullpath\` with your own paths. Run in the user's context, not as SYSTEM, since drive mappings are per user.

## Usage

1. Create a new component or script in your RMM and paste the script in.
2. Set the script type to PowerShell and run as SYSTEM (except the drive re-map script).
3. Map `Custom19`, `Custom20`, and `Custom24` to UDFs if you use them.
4. For monitors, alert on exit code `1` or on the captured result text.
5. Test on one device before pushing to a policy.

## Disclaimer

These scripts change system state and read sensitive inventory data. Review and test them in your own environment before deploying. Use at your own risk.

