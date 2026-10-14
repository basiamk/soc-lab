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
| 2 | T1110 | Brute Force — RDP | Hydra | hydra -l labuser -P wordlist.txt rdp://192.168.1.30 -t 4 -V -f | In progress | TBD | TBD | |
| 3 | T1059.001 | PowerShell Execution | Atomic Red Team | Invoke-AtomicTest T1059.001 | Pending | | | |
| 4 | T1003.001 | LSASS Credential Dumping | Metasploit | hashdump via meterpreter | Pending | | | |
| 5 | T1021.001 | RDP Lateral Movement | xfreerdp | xfreerdp /u:labuser /p:... /v:192.168.1.30 | Pending | | | |
| 6 | T1105 | Ingress Tool Transfer | PowerShell | Invoke-WebRequest to download payload | Pending | | | |
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
