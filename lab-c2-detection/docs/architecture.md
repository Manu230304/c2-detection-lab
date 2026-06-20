# Lab Architecture — Detailed Notes

## Virtualization

All VMs were created using a Type-2 hypervisor (e.g., VirtualBox / VMware Workstation) on the host machine.

### Network Configuration

**Network type: Bridge Adapter**

Bridge networking gives each VM its own IP on the host's physical network, allowing direct communication between VMs and the host without NAT. This was chosen to simplify C2 communication (no port forwarding needed) and to mirror a realistic flat network scenario.

| VM | OS | IP (example) | vCPU | RAM |
|---|---|---|---|---|
| Attacker | Kali Linux (rolling) | 192.168.1.10 | 2 | 4 GB |
| Victim | Windows 11 Pro | 192.168.1.20 | 2 | 4 GB |
| SIEM | Ubuntu Server 22.04 | 192.168.1.30 | 2 | 6 GB |

> ⚠️ Wazuh Manager + Indexer + Dashboard is resource-heavy. 6 GB RAM minimum is recommended for the SIEM VM.

---

## Attack Flow Diagram

```
[Kali — Attacker]
       │
       │  1. Generate implant (Sliver --http)
       │  2. Serve via python3 -m http.server 8080
       │
       ▼
[Windows 11 — Victim]
       │
       │  3. Download implant (browser/curl)
       │  4. Disable Windows Defender temporarily
       │  5. Execute implant
       │
       │◄──────────────────────────────────────┐
       │                                       │
       │  6. HTTP C2 beacon to Kali            │
       │                                       │
       ▼                                       │
[Kali — Sliver Listener]                       │
       │                                       │
       │  7. Interactive session open          │
       │  8. Post-exploitation commands        │
       │     → shell, ls, reg queries          │  All detected
       │                                       │  by Wazuh ↓
       ▼
[Ubuntu — Wazuh]
       │
       │  ALERTS FIRED:
       │  • Sysmon Event 1  — process create
       │  • Sysmon Event 13 — registry write
       │  • Sysmon Event 3  — network conn.
       │  • Sysmon Event 11 — file create
       │
       ▼  (Wazuh Manager stopped to create gap)
       
[Evasion Phase]
       │
       │  9.  Add Defender exclusion: C:\Windows\Tracing
       │  10. Move implant to C:\Windows\Tracing\DiagTrackHost.exe
       │  11. Establish persistence:
       │       a) HKCU Run Key
       │       b) Scheduled Task (onlogon, HIGHEST)
       │
       └──► Backdoor active on every user logon
```

---

## Design Decisions

### Why Sliver?

Sliver is an open-source, actively maintained C2 framework from BishopFox. It was chosen over Metasploit/Cobalt Strike because:
- Fully open-source (reproducible in a home lab)
- Supports multiple protocols (HTTP, HTTPS, DNS, mTLS)
- Good session management and built-in post-exploitation

### Why `C:\Windows\Tracing`?

- Real directory present on all Windows installations (used by ETW tracing)
- Not monitored by default in most AV/EDR exclusion lists
- Low-entropy path — doesn't stand out in process trees
- The filename `DiagTrackHost.exe` exists as a legitimate Windows binary in some builds

### Why both Run Key AND Scheduled Task?

Redundancy. If a blue teamer removes one persistence mechanism, the other survives. It also tests which of the two Wazuh is better at detecting (both were caught when the SIEM was active).
