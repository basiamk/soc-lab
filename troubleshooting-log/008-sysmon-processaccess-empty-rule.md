# Incident 008: Sysmon ProcessAccess Empty Include Block — Zero LSASS Visibility by Default

**Date:** 2026-11-09 to 2026-11-10  
**Severity:** High (complete blind spot on the most consequential technique in the lab)  
**Time to resolve:** ~90 minutes across two sessions  
**Phase:** Phase 3 — Threat Simulation (T1003.001 LSASS Credential Dumping)

---

## Symptom

A full Meterpreter-based attack chain was executed against Windows 10:
payload delivery, privilege escalation via `getsystem` (Named Pipe
Impersonation), and credential dumping via Metasploit's Kiwi (embedded
Mimikatz) `creds_all` command. The attack succeeded completely —
domain Administrator NTLM hashes and a plaintext Kerberos password
were recovered from live LSASS memory.

Despite this, neither Wazuh nor the local Windows 10 Sysmon event log
showed any Event ID 10 (ProcessAccess) entries for the attack period.
Searching Wazuh for `data.win.eventdata.targetImage: *lsass*` and
`data.win.system.eventID: 10` both returned zero results.

---

## Impact

LSASS credential dumping is the single most consequential technique in
this lab's threat model — it is the mechanism by which a single
compromised endpoint becomes full domain compromise via Pass-the-Hash
or direct credential reuse. A complete absence of detection for this
technique, while every other attack (T1046, T1110, T1059.001) had at
least partial coverage, represented the most severe gap identified in
Phase 3.

---

## Diagnosis Process

### Step 1 — Verify locally on Windows 10, bypassing Wazuh entirely

Following the same verification discipline established in Incident 007
(never trust SIEM silence without checking the source), Sysmon's own
local operational log was queried directly:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" |
  Where-Object {$_.Id -eq 10 -and $_.TimeCreated -gt (Get-Date).AddMinutes(-15)}
```

Result: zero events. This confirmed the gap was at the Sysmon
collection layer itself, not a Wazuh forwarding issue — Sysmon had
not generated the event at all, so there was nothing for Wazuh to
receive.

### Step 2 — Inspect the active Sysmon configuration directly

```powershell
& "C:\Windows\Sysmon64.exe" -c
```

This command prints Sysmon's currently loaded ruleset directly from
the running service — the authoritative source, more reliable than
inspecting a config file on disk which may not reflect what was
actually loaded. The output revealed:

```
- ProcessAccess                      onmatch: include   combine rules using 'Or'
```

With no filter lines beneath it — an empty include block.

### Root cause identified

The SwiftOnSecurity Sysmon configuration deployed in Phase 2 ships with
ProcessAccess (Event ID 10) monitoring present in the file structure
but deliberately disabled by default, via an empty `onmatch="include"`
rule group. The original source file contains an explicit comment
confirming this is intentional, not an oversight:

```xml
<ProcessAccess onmatch="include">
    <!--NOTE: Using "include" with no rules means nothing in this
    section will be logged-->
</ProcessAccess>
```

ProcessAccess logging is extremely high-volume if enabled broadly —
legitimate processes (antivirus engines, debuggers, Task Manager)
constantly access other processes' memory for benign reasons. The
SwiftOnSecurity config author left the capability present in the file
but switched off, deferring the decision of what to monitor to each
deploying organisation rather than imposing a default that could
overwhelm log volume.

---

## Fix Attempt 1 — Supplemental config file (failed, caused regression)

A minimal XML file containing only a targeted ProcessAccess rule for
`lsass.exe` was created and applied:

```powershell
& "C:\Windows\Sysmon64.exe" -c "$env:TEMP\lsass-access-rule.xml"
```

Verification (`Sysmon64.exe -c`) showed the LSASS rule was active —
but also revealed that every other setting had reverted to Sysmon
defaults: `Network connection: disabled` (previously enabled),
`HashingAlgorithms: SHA256` only (previously `MD5,SHA256,IMPHASH`),
and every other rule group (ProcessCreate exclusions, FileCreate
includes, RegistryEvent monitoring, NetworkConnect filtering, DNS
exclusions) was entirely absent from the active configuration.

### Root cause of the regression

Sysmon's `-c <file>` command performs a **full replacement** of the
active configuration, never a merge. Supplying a minimal file
containing only one rule group discards every rule group not present
in that file. This is an important and easily-missed operational
behaviour: Sysmon configuration management has no incremental update
mechanism via the command line — every `-c` call must supply a
complete, self-consistent configuration.

---

## Fix Attempt 2 — Single complete master configuration (successful)

The original SwiftOnSecurity configuration file (`sysmonconfig.xml`,
retained in `$env:TEMP` from Phase 2 installation) was used as the
base. A PowerShell regex replace surgically substituted only the empty
`<ProcessAccess>` block for one containing the LSASS-targeted rule,
leaving every other line of the 700+ line file untouched:

```powershell
(Get-Content "$env:TEMP\sysmonconfig.xml" -Raw) -replace `
  '<ProcessAccess onmatch="include">\s*<!--NOTE:.*?-->\s*</ProcessAccess>', `
  "<ProcessAccess onmatch=`"include`">`r`n            <TargetImage condition=`"end with`">lsass.exe</TargetImage>`r`n        </ProcessAccess>" |
  Set-Content "$env:TEMP\sysmonconfig-with-lsass.xml" -Encoding ASCII
```

The `-Raw` flag reads the entire file as a single string so the
multi-line regex pattern can match across the block correctly, rather
than processing line by line.

The merged rule:

```xml
<ProcessAccess onmatch="include">
    <TargetImage condition="end with">lsass.exe</TargetImage>
</ProcessAccess>
```

This rule includes any ProcessAccess event where the **target**
process image ends with `lsass.exe`, regardless of which process is
the source. This is deliberately broad on the source side (catches
any tool, not just known malicious ones) and narrow on the target side
(only LSASS, avoiding the high-volume problem that justified disabling
ProcessAccess by default).

This complete file was verified to contain both the original ruleset
(spot-checked via `Select-String` showing FileCreate, Event ID 11
rules still present immediately after the ProcessAccess block) and the
new LSASS rule, before being applied:

```powershell
& "C:\Windows\Sysmon64.exe" -c "$env:TEMP\sysmonconfig-with-lsass.xml"
```

---

## Verification

Post-fix, the full attack chain was re-executed (payload re-delivered,
`getsystem` escalation, Kiwi `creds_all`). Direct inspection of the
Windows 10 Sysmon log confirmed the event now generates correctly:

```
TimeCreated : 11/10/2026 7:06:04 AM
Id          : 10
SourceImage: C:\Users\ADMINI~1\AppData\Local\Temp\update_check.exe
TargetImage: C:\WINDOWS\system32\lsass.exe
GrantedAccess: 0x1010
SourceUser: HOMELAB\Administrator
TargetUser: NT AUTHORITY\SYSTEM
```

`GrantedAccess: 0x1010` combines `PROCESS_VM_READ` (0x0010) and
`PROCESS_QUERY_INFORMATION` (0x1000) — the specific access rights
Mimikatz/Kiwi requests to read credential material from LSASS memory.
This is a well-documented, specific indicator, not a generic access
event.

Wazuh confirmed receipt and correctly matched a named, specific rule:

```
Lsass process was accessed by C:\Users\ADMINI~1\AppData\Local\Temp\
update_check.exe with read permissions, possible credential dump
Rule 92900, level 12
```

This is a precise, intent-specific detection — not an indirect
side-effect match as seen with earlier techniques in this lab — stating
plainly what occurred.

A sanity-check comparison was also preserved in the same log output:
three additional ProcessAccess events from `svchost.exe` (SYSTEM
source, SYSTEM target) with lower access masks (0x1000, 0x2000)
represent legitimate Windows internal LSASS interaction, confirming
the new rule captures all LSASS access broadly while Wazuh's rule 92900
correctly distinguishes the suspicious Administrator-sourced 0x1010
access from this background noise.

---

## Secondary Observation — Queue Pressure Recurrence

During the verification run, Wazuh's internal queue-health alerts (see
Incident 007, rules 202/203/204) reappeared briefly:

```
07:06:41  Agent event queue is 90% full.             level 7
07:06:44  Agent event queue is full. Events may be lost.  level 9
07:06:58  Agent event queue is flooded.               level 12
```

The critical LSASS detection event (07:06:05) landed successfully
before this pressure began, so no data was lost for the event that
mattered most here. However, this recurrence under a sustained burst
of back-to-back Meterpreter commands (`getsystem`, `load kiwi`,
`creds_all` in quick succession) indicates the Incident 007 buffer
tuning (10000 queue, 1000 events/sec) provides meaningful but not
unlimited headroom. A genuinely production-scale deployment collecting
this many channels (ten, per the agent.conf from Incident 007) would
need either further buffer tuning or a reduced channel set — a direct,
empirical input to the merged roadmap's Phase 4 telemetry-vs-resource-cost
research question.

---

## Root Cause Summary

The SwiftOnSecurity Sysmon configuration intentionally ships with
ProcessAccess (Event ID 10) monitoring disabled by default via an
empty include rule group, to avoid the high log volume of unrestricted
process memory access monitoring. This is documented in the source
file's own comments but easy to miss when reviewing a 700+ line
configuration. The fix required building a single complete
configuration file — Sysmon's `-c` command always fully replaces the
active configuration rather than merging, so any targeted addition
must be made to a complete base file, never supplied as a standalone
supplemental file.

---

## Lesson Learned

Community security tool configurations are often conservative by
design, deliberately leaving high-value but high-volume detections
switched off and documenting that choice in comments rather than
defaults. Reading configuration files for what they *don't* do is as
important as reading them for what they do — an empty rule block with
no filters is easy to scan past as "present and working" when it is
in fact a silent no-op.

Separately, Sysmon's configuration update model (full replace, not
merge) is a sharp edge that is easy to trigger accidentally while
trying to make a small, targeted change. The correct operational
pattern is to always edit and reapply one complete master configuration
file, never layer partial updates — exactly the same lesson Git and
most configuration management tools teach about avoiding partial state.

This incident also reinforces the verification discipline from
Incident 007: confirm at the source (the local Sysmon log) before
concluding anything about what the SIEM does or does not show.

---

## References

- SwiftOnSecurity sysmon-config: github.com/SwiftOnSecurity/sysmon-config
- Sysmon Event ID 10 (ProcessAccess) documentation: Microsoft Sysinternals
- Windows process access rights (PROCESS_VM_READ, PROCESS_QUERY_INFORMATION):
  learn.microsoft.com/windows/win32/procthread/process-security-and-access-rights
- Wazuh rule 92900 (LSASS credential dump detection):
  /var/ossec/ruleset/rules/0920-win_sysmon_rules.xml (or custom ruleset location)
- troubleshooting-log/007-powershell-detection-pipeline.md — related agent
  buffer tuning referenced in Secondary Observation
