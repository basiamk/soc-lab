# Sysmon Configuration

## What is Sysmon

System Monitor (Sysmon) is a Windows system service and device driver that
monitors and logs system activity to the Windows Event Log. It provides
detailed information about process creation, network connections, file
creation, registry changes, and more.

Without Sysmon, Windows event logs contain only basic security events.
With Sysmon, you get rich telemetry that maps directly to MITRE ATT&CK
techniques. For a SOC analyst, Sysmon is the difference between seeing
"something happened" and knowing exactly what process did what to which
file with which parent process.

---

## Key Sysmon Event IDs

| Event ID | Name | Detection Value |
|---|---|---|
| 1 | Process creation | Execution detection — see what ran and who spawned it |
| 2 | File creation time changed | Anti-forensics indicator |
| 3 | Network connection | C2 communication, lateral movement |
| 5 | Process terminated | Process lifecycle tracking |
| 7 | Image loaded | DLL injection detection |
| 8 | CreateRemoteThread | Process injection |
| 10 | ProcessAccess | LSASS dumping (T1003.001) |
| 11 | FileCreate | Dropper, payload staging |
| 12/13 | Registry events | Persistence mechanisms |
| 15 | FileCreateStreamHash | Alternate data streams |
| 22 | DNS query | Domain generation algorithm detection |
| 23 | FileDelete | Evidence wiping |

---

## Installation

Deployed on Windows 10 (192.168.1.30) and Windows Server 2022 (192.168.1.10).

PowerShell commands run as Administrator on each Windows VM:

```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "$env:TEMP\Sysmon.zip"

# Extract
Expand-Archive "$env:TEMP\Sysmon.zip" `
  -DestinationPath "$env:TEMP\Sysmon" -Force

# Download SwiftOnSecurity config
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
  -OutFile "$env:TEMP\sysmonconfig.xml"

# Install with config
& "$env:TEMP\Sysmon\Sysmon64.exe" -accepteula -i "$env:TEMP\sysmonconfig.xml"
```

Verified running:
```powershell
Get-Service Sysmon64
# Status: Running
```

---

## SwiftOnSecurity Config

The SwiftOnSecurity sysmon-config is a community-maintained, production-ready
Sysmon configuration used by organisations worldwide. It provides:
- Noise reduction: excludes known-good processes to reduce alert fatigue
- Coverage: captures the events most relevant to threat detection
- Maintenance: regularly updated as new techniques emerge

Source: github.com/SwiftOnSecurity/sysmon-config

---

## Wazuh Integration

ossec.conf updated on Ubuntu Server to collect Sysmon events from Windows
agents:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

This tells the Wazuh agent on each Windows machine to forward the Sysmon
operational event log channel to the manager for analysis.

---

## Impact on Detection

Before Sysmon: Windows agents sent basic Security event logs only.
After Sysmon: Process creation, network connections, file events, and
registry changes all visible in Wazuh Threat Hunting.

This dramatically improves detection coverage for Phase 3 attack simulations,
particularly for T1059.001 (PowerShell execution), T1003.001 (LSASS access
via Sysmon Event ID 10), and T1105 (file creation via Sysmon Event ID 11).
