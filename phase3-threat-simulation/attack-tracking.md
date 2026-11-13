# Phase 3 — Attack Simulation Tracking

## Purple Team Methodology

Each technique follows this loop:

1. Run the attack from Kali (192.168.1.40) against Windows 10 (192.168.1.30)
2. Check Wazuh Threat Hunting for alerts within 5 minutes
3. Document the result — alert fired, partial detection, or detection gap
4. Detection gaps feed directly into Phase 4 custom rule writing

---

## Environment

| Component | Value |
|---|---|
| Attacker | Kali Linux 2026.1 — 192.168.1.40 |
| Primary target | Windows 10 Evaluation — 192.168.1.30 |
| Secondary target | Windows Server 2022 DC — 192.168.1.10 |
| SIEM | Wazuh 4.11.2 — 192.168.1.20 |
| AD Domain | homelab.lab |
| Test account | labuser (local, member of Remote Desktop Users) |

---

## Detection Tracking Table

| # | ATT&CK ID | Technique | Tool | Command | Alert Fired | Rule ID | Level | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | T1046 | Network Service Discovery | Nmap | nmap -sS -sV -O 192.168.1.30 | Partial | 61061, 60602 | 10, 9 | Windows error events spiked. No dedicated scan rule. Gap for Phase 4. |
| 2 | T1110 | Brute Force RDP | Hydra | hydra -l labuser -P wordlist.txt rdp://192.168.1.30 -t 4 -V -f | YES | 60122, 60204, 92657 | 5, 10, 6 | Full detection chain. Failures detected individually (60122), threshold fired at level 10 (60204), successful logon detected (92657). Attack duration 6 seconds. Password found on attempt 8 of 10. |
| 3 | T1059.001 | PowerShell Execution | Native PowerShell | -EncodedCommand / IEX DownloadString / AMSI reflection | FULL (after fix) | 92057, 92213, 92066 | 12, 15, 4 | Initial zero-detection — 4-layer pipeline fix (see troubleshooting-log/007). Rule 92057 now specifically names base64-encoded execution. |
| 4 | T1003.001 | LSASS Credential Dumping | Metasploit (Kiwi/Mimikatz) | load kiwi; getsystem; creds_all | FULL | 92900 | 12 | Initial gap: Sysmon ProcessAccess (Event ID 10) had zero rules defined in SwiftOnSecurity default config — required custom LSASS-targeted rule. Fix documented in troubleshooting-log/008. After fix: specific, named detection — "possible credential dump" — fired correctly. |
| 5 | T1021.001 | RDP Lateral Movement | xfreerdp (stolen domain admin credential) | xfreerdp /v:192.168.1.30 /u:Administrator /d:homelab.lab /p:'AdminLab@2026' | FULL | 92657, 92653 | 6, 3 | No detection gap — both rules fired on first attempt with no configuration changes needed. DC (192.168.1.10) lateral movement blocked entirely at network layer by pfSense Rule 4 (filtered, not rejected). Windows 10 lateral movement succeeded via NTLM fallback after Kerberos KDC lookup failure, generating Logon Type 3 then Type 10 (RemoteInteractive) sequence. |
| 6 | T1105 | Ingress Tool Transfer | certutil (LOLBin), bitsadmin (LOLBin), Invoke-WebRequest | certutil -urlcache, bitsadmin /transfer, Invoke-WebRequest | PARTIAL | 91816 | 4 | All 3 methods detected via PowerShell 4104 Script Block Logging only. No named certutil/bitsadmin-specific rules in default Wazuh ruleset. BITS failed (0x800704dd session context). Detection fires on intent regardless of transfer success. Phase 4 target: dedicated certutil -urlcache and bitsadmin Sigma rules. |
| 7 | T1078 | Valid Account Abuse | net use | net use with domain credentials | Pending | | | |
| 8 | T1486 | Ransomware Precursor | Atomic Red Team | Invoke-AtomicTest T1486 | Pending | | | |
| 9 | T1557 | LLMNR Poisoning | Responder | responder -I eth0 -rdwv | Pending | | | |
| 10 | T1201 | SIM Swap Precursor | Hydra + enum | Credential harvest + account enum | Pending | | | |

---

## Windows Event IDs Referenced

These Event IDs are the primary telemetry for Windows-based detections.
Knowing these is essential for any SOC analyst role.

| Event ID | Description | Relevance |
|---|---|---|
| 4624 | Successful logon | Baseline, and critical after brute force |
| 4625 | Failed logon | Primary brute force indicator |
| 4648 | Logon with explicit credentials | Lateral movement indicator |
| 4740 | Account locked out | Brute force consequence |
| 4768 | Kerberos TGT requested | Domain authentication |
| 4769 | Kerberos service ticket requested | Kerberoasting indicator |
| 4776 | NTLM credential validation | Pass-the-hash indicator |
| 1 (Sysmon) | Process creation | Execution detection |
| 3 (Sysmon) | Network connection | C2 / lateral movement |
| 10 (Sysmon) | Process access | LSASS dumping indicator |
| 11 (Sysmon) | File created | Dropper / payload detection |

---

## Detection Gap Log

Gaps documented here feed Phase 4 custom Sigma / Wazuh rule writing.

| # | ATT&CK ID | Gap Description | Phase 4 Rule Planned |
|---|---|---|---|
| 1 | T1046 | No dedicated port scan detection rule in default Wazuh | Sigma rule: rapid SYN scan from internal IP |

---

## SA-Specific Scenarios

These additional scenarios are planned after the core 10 techniques:

| Scenario | Technique Basis | Relevance |
|---|---|---|
| SIM swap precursor | T1110 + T1087 | SA banking — digital fraud vector |
| Bulk data export (POPIA) | T1530 | POPIA Section 19 data protection |
| After-hours admin login | T1078 | Insider threat / POPIA audit rule |
| Ransomware file encryption | T1486 | Critical infrastructure protection |
