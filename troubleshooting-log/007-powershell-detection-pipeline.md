# Incident 007: PowerShell Detection Pipeline — Zero Detections to Full Coverage

**Date:** 2026-10-27 to 2026-11-09  
**Severity:** High (complete blind spot on T1059.001 technique)  
**Time to resolve:** Multi-session investigation, approximately 4 hours total  
**Phase:** Phase 3 — Threat Simulation (T1059.001 PowerShell Execution)

---

## Symptom

Three PowerShell attack scenarios (Base64 encoded command execution,
download cradle, AMSI bypass reflection string) were executed against
Windows 10 (192.168.1.30) and confirmed to run successfully via direct
Sysmon log inspection on the endpoint. Despite this, Wazuh Threat Hunting
showed zero alerts related to any of the three scenarios across multiple
15-minute search windows immediately following execution.

This represented a complete detection blind spot for one of the most
common attack techniques in the MITRE ATT&CK framework.

---

## Impact

T1059.001 (PowerShell Execution) is used in the majority of real-world
attack chains as a Living off the Land technique. A SIEM with zero
visibility into PowerShell activity has a critical gap that would allow
an attacker significant freedom to execute commands, download tooling,
and attempt defence evasion without triggering any alert.

---

## Diagnosis Process — Four Sequential Root Causes

This incident involved four distinct, independently-discovered problems
layered on top of each other. Each fix revealed the next layer.

### Layer 1 — Manager config vs agent config confusion

**Hypothesis:** Sysmon events were not reaching Wazuh at all.

**Diagnostic steps:**

```bash
sudo grep -i "sysmon" /var/ossec/logs/alerts/alerts.log | tail -20
sudo grep -i "sysmon" /var/ossec/etc/ossec.conf
```

The second command confirmed a `<localfile>` entry for the Sysmon
channel existed in the **manager's** ossec.conf. The first command
returned zero results — no Sysmon events had ever been alerted on.

**Root cause:** The Sysmon collection entry had been added to the
Wazuh **manager's** local ossec.conf during Phase 2, which only
controls what the manager itself collects locally on Ubuntu Server.
It does nothing to tell remote agents (Windows 10, Windows Server
2022, etc.) what to collect. Verified directly by opening the agent's
own ossec.conf on Windows 10:

```powershell
Select-String -Path "C:\Program Files (x86)\ossec-agent\ossec.conf" -Pattern "Sysmon"
```

Result: `Cannot find "sysmon"`. The agent had no knowledge Sysmon
existed.

**Fix:** Created a populated `agent.conf` on the manager at
`/var/ossec/etc/shared/default/agent.conf`, which Wazuh automatically
pushes to every agent in the default group. This is the correct
architecture for distributing configuration to multiple endpoints —
write once on the manager, every agent receives it on their next
60-second check-in. Added ten event channels total: Sysmon, PowerShell
operational, legacy Windows PowerShell, Security, System, Application,
Windows Defender operational, Task Scheduler operational, Terminal
Services local session manager, and WMI activity.

**Verification:** Agent configuration hash changed after manager
restart, confirming the new config was received:
```
Before: ab73af41699f13fdd81903b5f23d8d00
After:  2b1d54084e28502b5c971a78b8278615
```

---

### Layer 2 — Script Block Logging not enabled

**Hypothesis:** With the correct channels now configured, PowerShell
execution should generate Sysmon Event ID 1 (process creation) and
content should be visible.

**Finding:** Sysmon Event ID 1 and Event ID 11 (file creation side
effects) began appearing correctly after the Layer 1 fix. However,
the actual PowerShell command content (decoded Base64, script logic)
was still invisible — only the fact that `powershell.exe` ran was
visible, not what it ran.

**Root cause:** PowerShell Script Block Logging, which captures full
decoded command content to Event ID 4104 on the
`Microsoft-Windows-PowerShell/Operational` channel, is disabled by
default and requires explicit registry configuration (or GPO in a
production domain).

**Fix:**
```powershell
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockInvocationLogging" -Value 1
```

**Complication encountered:** Windows 10 is domain-joined to
homelab.lab. The Domain Controller was offline at the time of initial
testing, raising a hypothesis that Group Policy would overwrite the
manual registry setting once the DC came back online and pushed
policy. This was tested directly:

```powershell
gpupdate /force
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
```

Result: the registry values survived the forced policy update
unchanged (`EnableScriptBlockLogging: 1`), ruling out GPO override as
a factor. PowerShell version was also verified as 5.1.19041.6456,
confirmed to fully support Script Block Logging (requires 5.0+).

**Verification:** `Get-WinEvent -ListLog` confirmed the
`Microsoft-Windows-PowerShell/Operational` channel was enabled and
actively accumulating events (RecordCount: 768 and climbing).

---

### Layer 3 — Operational noise masking genuine events

**Hypothesis:** With Script Block Logging confirmed enabled, re-running
the attack scenarios should now produce visible 4104 events containing
decoded command content.

**Finding:** Initial re-tests appeared to fail — `Get-WinEvent -MaxEvents
5` and similar small-window queries kept returning only PowerShell's own
internal `prompt` script block (which PowerShell re-evaluates every time
the interactive console redraws its prompt) and `$global:?` (an internal
status-check variable), never the actual attack command.

**Root cause:** PowerShell's console `prompt` function is itself a
script block, and it is re-logged as a fresh 4104 event on every single
prompt redraw in an interactive session. In an active terminal this
floods the operational log with dozens of near-identical `prompt`
events between any two typed commands, burying genuine command content
when only the last 5-10 events are inspected.

**Diagnostic fix:** Switched from count-based queries (`-MaxEvents 5`)
to time-window queries:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" |
  Where-Object {$_.Id -eq 4104 -and $_.TimeCreated -gt (Get-Date).AddMinutes(-5)}
```

This surfaced the full set of events in the window rather than only the
most recent handful, and the target command (`whoami`, correctly decoded
from its Base64 encoding) was confirmed present among the noise:

```
TimeCreated : 11/9/2026 10:12:35 AM
Id          : 4104
Message     : Creating Scriptblock text (1 of 1):
              whoami
```

**Operational implication for Phase 4:** Any custom detection rule built
on the PowerShell operational channel must explicitly exclude or
deprioritise `scriptBlockText` values of `prompt` and `$global:?`, or
every rule built on this channel will generate overwhelming false-positive
volume in a live environment. This is a genuine production consideration,
not just a lab artefact — any organisation ingesting PowerShell Script
Block Logging at scale faces this exact noise problem.

---

### Layer 4 — Agent event queue throttling silently dropping events

**Hypothesis:** With the `whoami` event confirmed present in the local
Windows event log, it should now be searchable in Wazuh.

**Finding:** Wazuh search for `data.win.eventdata.scriptBlockText:
"whoami"` returned zero results. A direct grep of the manager's raw
alerts.json for the string `whoami` also returned nothing — the event
had not arrived at the manager at all, despite the agent reporting
Active status with a recent keepalive.

**Root cause:** The agent.conf written in Layer 1 added ten event
channels without any corresponding `client_buffer` tuning. The Wazuh
Windows agent defaults to a queue size of 5000 events and a throughput
cap of 500 events per second. Under the combined load of ten
simultaneously-collecting channels plus PowerShell's own prompt-noise
behaviour (Layer 3), short bursts of a dozen or more events landing
within the same one-second window exceeded this default capacity,
causing silent event loss with no error surfaced to the operator.

This was confirmed directly by Wazuh's own internal self-monitoring
alerts, which began firing during a subsequent test run:

```
Agent event queue is 90% full.                      level 7   rule 202
Agent event queue is full. Events may be lost.       level 9   rule 203
Agent event queue is flooded. Check the agent configuration.  level 12  rule 204
```

**Fix:** Added explicit client buffer tuning to agent.conf:

```xml
<client_buffer>
  <disabled>no</disabled>
  <queue_size>10000</queue_size>
  <events_per_second>1000</events_per_second>
</client_buffer>
```

Queue size doubled from the 5000 default to absorb burst activity.
Throughput cap doubled from 500 to 1000 events per second to allow
multi-channel bursts to clear without backing up. Values were chosen
deliberately conservative rather than maximal, given Ubuntu Server's
constrained 4 GB RAM (see Incident 002 — prior memory exhaustion
history on this same host).

**Verification:** Configuration hash changed to
`8f0dd905834302796455dea5d799b4ff` after manager restart. A follow-up
check of the most recent 100 alerts showed zero further queue-related
warnings, confirming the throttling condition had cleared.

---

## Final Verification — Full Pipeline Confirmed

Following the Layer 4 fix, a fresh test run of the original three attack
scenarios generated specific, correctly-attributed Wazuh alerts:

| Timestamp | Rule ID | Description | Level |
|---|---|---|---|
| 10:38:28.804 | 92201 | powershell.exe created a new scripting file under Windows Temp or User data folder | 9 |
| 10:38:35.928 | 91816 | Powershell script querying system environment variables | 4 |
| 10:38:40.997 | 92066 | SecEdit.exe binary in a suspicious location launched by powershell.exe | 4 |
| 10:39:23.905 | 92052 | Windows command prompt started by an abnormal process | 4 |
| 10:39:58.270 | 92213 | Executable file dropped in folder commonly used by malware | 15 |
| 10:40:14.458 | 92057 | Powershell.exe spawned a powershell process which executed a base64 encoded command | 12 |
| 10:40:14.461 | 92213 | Executable file dropped in folder commonly used by malware | 15 |

Rule 92057 is the most significant result — a direct, named match for
Base64-encoded PowerShell execution at severity level 12, firing
specifically and correctly rather than through an indirect side-effect
as in the original pre-fix test.

---

## Root Cause Summary

Four independent, sequential causes, each masking the next until
individually resolved:

1. **Architecture misunderstanding** — manager ossec.conf confused with
   agent configuration distribution (fixed via agent.conf)
2. **Missing feature enablement** — Script Block Logging disabled by
   default, required explicit registry configuration
3. **Signal-to-noise problem** — PowerShell's own prompt re-evaluation
   behaviour flooding small search windows, requiring time-window
   queries instead of count-based queries
4. **Resource throttling** — default agent buffer limits silently
   dropping events under multi-channel burst load, requiring explicit
   client_buffer tuning

---

## Lesson Learned

A SIEM reporting zero alerts for a technique that is known to have
executed is not evidence of a secure environment — it is evidence of
an undiagnosed gap. Each of the four layers in this investigation
produced a plausible-looking "it's working now" signal that turned out
to be incomplete: Sysmon events for other techniques continued working
normally throughout, masking the PowerShell-specific gap in Layer 1;
the registry setting being present masked the fact that it had only
just taken effect in Layer 2; a handful of visible 4104 events masked
the fact that the specific event of interest was being pushed out of
view by noise in Layer 3; and the agent reporting "Active" status
masked silent packet loss in Layer 4.

The methodology that resolved this was consistent at every layer:
verify the claim directly at the source (the Windows event log itself)
before trusting what the SIEM shows, and never accept "no alerts" as
confirmation of "no activity" without independently confirming the
activity occurred and should have been visible.

This has direct relevance to real SOC work: an analyst who sees a quiet
dashboard must ask "is this quiet because nothing happened, or because
something is broken?" — and this incident is a worked example of how to
answer that question methodically.

---

## References

- Wazuh client_buffer documentation: documentation.wazuh.com/current/user-manual/reference/ossec-conf/client-buffer.html
- Wazuh internal rule group `ossec` (queue monitoring rules 202-204):
  /var/ossec/ruleset/rules/0045-ossec_rules.xml
- Microsoft PowerShell Script Block Logging: learn.microsoft.com/powershell/scripting/windows-powershell/wmf/whats-new/script-logging
- Wazuh rule 92057 (encoded command detection): /var/ossec/ruleset/rules/0920-win_powershell_rules.xml
