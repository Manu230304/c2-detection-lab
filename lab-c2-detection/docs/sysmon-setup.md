# Sysmon Setup

## What is Sysmon?

System Monitor (Sysmon) is a Windows system service and device driver that logs detailed system activity to the Windows Event Log. When combined with Wazuh, it provides visibility into:

- Process creation and command-line arguments
- Network connections (source/destination IP, port)
- File creation events
- Registry modifications
- Driver and image loads

## Installation

Download from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon).

```powershell
# Install with default config (minimal)
.\Sysmon64.exe -accepteula -i

# Install with a community config (recommended)
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

The config file used in this lab is included at [`configs/sysmon-config.xml`](../configs/sysmon-config.xml) — a subset of the [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config) ruleset.

## Key Event IDs

| Event ID | Description | Why It Matters |
|---|---|---|
| 1 | Process Create | Captures every new process with full command line |
| 3 | Network Connection | Logs outbound C2 connections |
| 11 | File Create | Tracks new files written to disk |
| 12 | Registry Object Added/Deleted | Persistence via registry |
| 13 | Registry Value Set | Detects Run key writes |
| 22 | DNS Query | Tracks domain lookups (C2 domains) |

## Stopping/Protecting Sysmon

During the lab, attempts were made to stop or unload Sysmon from the compromised session:

```cmd
# These all failed without SYSTEM privileges
sc stop Sysmon64
fltMC unload SysmonDrv
```

Sysmon's driver (`SysmonDrv`) runs at kernel level and is protected against termination by non-SYSTEM processes. This demonstrates the value of Sysmon as a resilient sensor even against admin-level attackers.
