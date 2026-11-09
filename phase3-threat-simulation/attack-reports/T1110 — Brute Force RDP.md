# Attack Report: T1046 — Network Service Discovery

**Date:** 2026-04-30  
**ATT&CK ID:** T1046  
**Technique:** Network Service Discovery  
**Tactic:** Discovery  
**Tool:** Nmap 7.99  
**Attacker:** Kali Linux — 192.168.1.40  
**Target:** Windows 10 Evaluation — 192.168.1.30  
**Detection result:** Partial  

---

## Technique Description

Network Service Discovery (T1046) is a Discovery technique where an adversary
scans a network to enumerate running services, open ports, and operating system
versions. This is typically one of the first actions taken after initial access
to understand the environment and identify further attack paths.

In South African financial institutions, internal reconnaissance from a
compromised endpoint is a strong indicator of lateral movement preparation.
SOC analysts monitor for unusual internal scanning — legitimate systems do not
typically scan hundreds of ports against other internal hosts.

---

## Attack Execution

### Command

```bash
nmap -sS -sV -O 192.168.1.30
```

### Command breakdown

| Flag | Meaning |
|---|---|
| -sS | SYN scan: sends TCP SYN, reads response, never completes handshake. Stealthier than full connect scan. |
| -sV | Service version detection: probes open ports to identify software and version. |
| -O | OS detection: fingerprints the operating system from TCP/IP stack behaviour. |

### Output

```
Nmap scan report for 192.168.1.30
Host is up (0.00072s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE    SERVICE   VERSION
135/tcp filtered msrpc
139/tcp filtered netbios-ssn
445/tcp filtered microsoft-ds
MAC Address: 00:0C:29:FE:DF:C5 (VMware)
OS details: Microsoft Windows family
Network Distance: 1 hop
Scan completed in 7.82 seconds
```

### Observations

Ports 135, 139, and 445 were detected as filtered. These are Windows-specific:
- Port 135: Microsoft RPC — used for DCOM and WMI
- Port 139: NetBIOS Session Service — legacy Windows file sharing
- Port 445: SMB — Server Message Block, used for file shares and lateral
  movement. EternalBlue (MS17-010) exploited this port in the WannaCry
  ransomware outbreak.

OS fingerprinting identified the target as Windows family. The scan took 7.82
seconds and generated a measurable spike in network traffic.

---

## SIEM Detection Analysis

### Wazuh dashboard observation

After the scan, Wazuh Threat Hunting showed a spike in events from
Windows10Desktop agent at approximately the same timestamp:

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 61061 | Multiple Windows error application events | 10 | 3 |
| 60602 | Windows application error event | 9 | 2 |
| 60608 | Summary event of report signatures | 4 | 4 |

### What this means

The SYN scan caused Windows to log application errors as it processed the
rapid incoming connection attempts on closed/filtered ports. Wazuh detected
these errors at severity level 10 but did not fire a dedicated port scan alert.

This is an **indirect detection** — a real analyst would see the spike in
Windows error events and investigate further. However, there is no alert that
explicitly says "port scan detected from 192.168.1.40."

---

## Detection Verdict

**PARTIAL** — Wazuh detected anomalous Windows behaviour caused by the scan
but did not fire a dedicated network scanning rule.

---

## Detection Gap

Default Wazuh rules do not include network-layer scan detection. This requires
either a network IDS (Suricata) or a custom rule detecting rapid failed
connection patterns from a single source IP.

### Gap documented for Phase 4

Rule to write: detect rapid port scanning from internal source IP.

Logic: more than 20 distinct destination ports contacted by the same source IP
within 30 seconds.

Sigma rule planned: `detection-rules/T1046-internal-port-scan.yml`

---

## Lessons Learned

A SYN scan is stealthier than a full TCP connect scan because it never
completes the three-way handshake. However, at the volume produced by Nmap
scanning 1000 ports, it still generates enough network noise to appear in
Windows error logs. A real attacker performing slower, more targeted scanning
(--scan-delay flag in Nmap) would generate significantly less noise and might
evade even this indirect detection.

In production environments, network-layer detection (Suricata, Zeek) is the
correct tool for scan detection. SIEM-only environments have a visibility gap
at the network layer that can only be partially compensated by endpoint
telemetry.
