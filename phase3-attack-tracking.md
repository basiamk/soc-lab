# Phase 3 — Attack Simulation Tracking

## Purple Team Loop
Attack → Check SIEM → Document → Fix gaps in Phase 4

## Detection Tracking Table

| # | ATT&CK ID | Technique Name | Tool Used | Command Run | Alert Fired? | Alert Level | Notes |
|---|---|---|---|---|---|---|---|
| 1 | T1046 | Network Service Discovery | Nmap | nmap -sS 192.168.1.30 | | | |
| 2 | T1110 | Brute Force | Hydra | hydra ... | | | |
| 3 | T1059.001 | PowerShell Execution | Atomic Red Team | Invoke-AtomicTest | | | |
| 4 | T1003.001 | LSASS Credential Dumping | Metasploit | | | | |
| 5 | T1021.001 | RDP Lateral Movement | xfreerdp | | | | |
| 6 | T1105 | Ingress Tool Transfer | PowerShell | | | | |
| 7 | T1078 | Valid Account abuse | net use | | | | |
| 8 | T1486 | Ransomware precursor | Atomic Red Team | | | | |
| 9 | T1557 | LLMNR Poisoning | Responder | | | | |
| 10 | T1201 | SIM Swap precursor | Hydra + enum | | | | |
| 1 | T1046 | Network Service Discovery | Nmap | nmap -sS -sV -O 192.168.1.30 | PARTIAL | 10 | 
Windows logged application errors from SYN flood. 
No specific port scan rule fired. 
Detection gap: need custom rule for rapid port scanning. 
Fix in Phase 4. |
